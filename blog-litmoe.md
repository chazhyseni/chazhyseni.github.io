# Running Trillion-Parameter MoE Models Behind One Local API

> **TL;DR:** `litmoe` puts **llama.cpp** and **ktransformers** behind a single endpoint that speaks both **OpenAI** and **Anthropic**. One `models.yaml`, one port, every model reachable by name — from a 4B-active MoE that chats at **9–12.7 t/s on a 24-core CPU with no GPU** to Kimi-K3 at 2.78T parameters. It ships a **29-model RAM-tiered catalog** verified against HuggingFace, hardware detection that picks what actually fits, and per-process harness wrappers so pointing Claude Code at a local model **never touches your Anthropic login**. ~3,900 lines of Python. Zero lines of inference code.

---

## The Problem Isn't Inference

Open Mixture-of-Experts models now span 17 GB to 600 GB+, and two engines already cover that range well.

**llama.cpp** owns the GGUF world: 1.5-bit to 8-bit quants across CUDA, HIP, Metal, Vulkan, SYCL, OpenCL, CANN, and plain CPU. **ktransformers** (Tsinghua MADSys Lab + Approaching.AI, SOSP 2025) owns heterogeneous serving: attention and dense layers on one GPU, routed experts on the CPU in native precision via AMX/AVX-512 kernels — the right shape for 200 GB–1 TB models on a single-GPU box with a lot of RAM.

Neither engine needs help with the forward pass. Everything *around* the forward pass is where the day goes:

| Gap | What it costs you |
|---|---|
| Two engines, two CLIs, two flag dialects | You memorize `llama-server` flags *and* `sglang.launch_server --kt-method` flags |
| No unified model list | Clients must know which port holds which model |
| OpenAI-only endpoints | Claude Code, Hermes, and every Anthropic-SDK tool can't reach them at all |
| Manual context sizing | Set `-c 262144` on a 48 GB box and the KV cache kills the process |
| "Which model fits my machine?" | Guesswork, or a 594 GB download that dies at 96% |
| Integration by environment variable | `export ANTHROPIC_BASE_URL=...` silently redirects *every* Claude Code session and SDK client in that shell |

None of these are inference problems. All of them are why local models sit unused. That gap is the entire surface litmoe covers.

---

## First, a Confession: 0.019 Tokens per Second

Version one of this project tried to be a *third engine* — a custom CPU-only C99 forward pass. On a 24-core EPYC it ran Kimi-K3 at **0.019 t/s**: 158 seconds to first token on a four-token prompt, with the worker thread parked in `DISK SLEEP`.

The arithmetic explains it, and no amount of code was going to move it:

| Quantity | Value |
|---|---|
| Prompt tokens × MoE layers × experts per token | 67 × 92 × 16 = **98,496 expert lookups** |
| Bytes per expert | 17.55 MB |
| Unique bytes to read, after ~50% dedup | **~859 GB** |
| Disk floor at 379 MB/s random read | **38 minutes** |
| Compute floor, 24 cores × ~50 ms per expert | **82 minutes** |
| llama.cpp on the same weights, same box | 0.85 t/s — **~45× faster** |

AVX2 matmul, mmap advisors, cross-layer prefetch, 2-bit quantization: all shipped, all irrelevant. The bandwidth simply isn't there.

So the forward pass was deleted. What replaced it is what the project should have been from the first commit — a dispatcher. Three lessons survived the demolition, and they generalize past this repo:

1. **A 50× gap to a mature engine on identical hardware means your optimization is wrong, not the hardware.**
2. **Measurement beats design.** The 82-minute compute floor was *calculated* before it was measured, and shipped past anyway. Several rounds of unmeasured "optimizations" went out before a wall clock settled it.
3. **The integration layer was the product all along.**

The full post-mortem lives in [`docs/METHODOLOGY.md`](https://github.com/chazhyseni/litMoE/blob/main/docs/METHODOLOGY.md).

---

## What Replaced It

```
   Clients (Claude Code, Hermes, Open WebUI, aider, curl, any OpenAI/Anthropic SDK)
        │  HTTP 127.0.0.1:8080
        │  /v1/chat/completions · /v1/completions · /v1/messages
        │  /v1/messages/count_tokens · /v1/models · /v1/models/{id} · /health
        ▼
   litmoe gateway (litmoe/server.py — 781 lines, FastAPI)
        │  read `model` → resolve alias → engine from models.yaml
        │  Anthropic /v1/messages ⇄ OpenAI chat completions, streaming included
        ▼
   engine subprocess :8081                engine subprocess :8082
   llama-server                           python -m sglang.launch_server
   CPU / CUDA / Metal / Vulkan            GPU attention + kt-kernel CPU experts
```

**3,897 lines of Python across 13 files.** No weights, no kernels, no quantization, no forward pass. The gateway costs single-digit milliseconds and zero compute.

| Module | Lines | Responsibility |
|---|---:|---|
| `litmoe/server.py` | 781 | Gateway, Anthropic↔OpenAI translation, engine supervision |
| `litmoe/cli/install.py` | 864 | Engine installer, HF model downloader, `models.yaml` writer |
| `litmoe/models.py` | 481 | The catalog — 29 models, 6 RAM tiers, fit math. Pure data, pure functions |
| `litmoe/platform_utils.py` | 403 | RAM, physical cores, NUMA, AVX-512/AMX, NVIDIA probing, macOS dylib repair |
| `litmoe/cli/main.py` | 302 | `doctor · models · init · install · serve · status · stop` |
| `litmoe/engines/*.py` | 485 | Engine ABC plus the llama.cpp and sglang-kt adapters |
| `litmoe/config.py` | 152 | Pydantic `models.yaml` schema and validation |
| `tests/test_litmoe.py` | 416 | 29 test functions / 37 cases, no network and no engines required |

Everything below is a consequence of that shape.

---

## Speaking Anthropic, Properly

Six of the gateway's seven routes are plumbing. `/v1/messages` is the one that unlocks an ecosystem — because Claude Code, Hermes, and every Anthropic SDK client speak Messages API and nothing else.

Translating the request is mostly bookkeeping:

| Anthropic construct | Becomes |
|---|---|
| `system` as string **or** list of text blocks | one or more `role: system` messages |
| `tool_use` blocks (assistant) | OpenAI `tool_calls` with JSON-serialized `arguments` |
| `tool_result` blocks (user) | separate `role: tool` messages carrying `tool_call_id` |
| `image` blocks | an inline `[image: <source type>]` placeholder |
| `thinking` / `redacted_thinking` on input | dropped — engines regenerate reasoning each turn |
| `tools` + `input_schema` | `type: function` with `parameters` |
| `tool_choice: any` or `tool` | `"required"` — llama-server accepts only string values |
| `tool_choice: none` | `tools` removed entirely |
| `stop_sequences` | `stop` |
| `finish_reason` | `stop_reason`: `stop→end_turn`, `length→max_tokens`, `tool_calls→tool_use`, `content_filter→refusal` |

Streaming is where naive gateways fall over, so it gets two separate paths. OpenAI streams pass through **byte for byte** — no parsing, no reframing, no buffering. Anthropic streams go through a real state machine that emits the full `message_start → content_block_start → content_block_delta → content_block_stop → message_delta → message_stop` lifecycle while tracking block indices across three block types:

- **text** — `text_delta` events.
- **thinking** — `thinking_delta` events, closed with a synthetic `signature_delta` so clients that validate block signatures don't reject the stream.
- **tool_use** — OpenAI's `tool_calls[].index` mapped onto Anthropic block indices, with arguments streamed as `input_json_delta` / `partial_json`.

Two details that only show up in practice: `stream_options: {include_usage: true}` is injected so the final chunk carries real token counts instead of a running guess, and an upstream HTTP ≥ 400 is converted into a well-formed Anthropic `error` event rather than a stream that simply stops.

`/v1/messages/count_tokens` is implemented too — Claude Code calls it before every request — and it is **honest about being an estimate** (~4 characters per token over the serialized prompt), because the engines expose no cross-model tokenizer.

Aliases are a gateway concept, not an engine one. The route table is keyed by canonical id *and* every alias, so resolution is one dict lookup; the payload's `model` field is then rewritten to the canonical id before forwarding, because sglang validates against `--served-model-name` and would reject an alias. When a model genuinely isn't served, the 404 lists every id and alias that *is*.

---

## A Catalog That Knows Your Machine

`litmoe/models.py` holds **29 entries** — 23 GGUF models for llama.cpp, 6 native-precision models for ktransformers — sorted into **six RAM tiers**. Every entry was checked against the HuggingFace file listing and llama.cpp's architecture table on 2026-09-16, and a `validate_catalog()` invariant check runs in the test suite.

| Tier | Representative models | Total / active | Default quant | Disk |
|---|---|---|---|---|
| **48 GB laptop** | `gemma-4-26b-a4b` **(default)** | 26B / 4B | UD-Q4_K_XL | 17 GB |
| | `qwen3.6-35b-a3b`, `nemotron-3.5-lightning-30b-a3b`, `gpt-oss-20b`, `kimi-linear-48b` | 21–48B / 3–3.6B | UD-Q4_K_XL, Q4_K_M | 12–30 GB |
| **96 GB** | `gpt-oss-120b`, `qwen3.5-122b-a10b`, `nemotron-3-super-120b-a12b`, `llama-4-scout` | 109–122B / 5.1–17B | UD-Q4_K_XL, UD-IQ4_XS | 60–64 GB |
| **192 GB** | `qwen3.8-flash-next`, `minimax-m2.7` (229B / 10B), `deepseek-v4-flash` | 177B–284B total | UD-Q4_K_XL | 111–155 GB |
| **512 GB** | `minimax-m3`, `glm-5.3`, `deepseek-v3.2`, `kimi-k2.5`, `kimi-k2.6` | 426B–1.03T | Q2–Q4 | 247–345 GB |
| **768 GB** | `qwen3.8` (2.4T / 95B), `kimi-k3` (2.78T / 93B) | 2.4–2.78T | UD-IQ1_S | 508 / 594 GB |

The fit rule is written down rather than felt:

```
RAM needed = weights × 1.08 (mmap + compute buffers)
           + KV cache @ 32K tokens (per-architecture bytes/token)
           + 4 GB OS headroom
```

macOS gets **75% of physical RAM** as its budget, since unified memory is shared with the OS and GPU. `litmoe models` prints the table with a fits / does-not-fit column for *your* machine, and `largest_quant_that_fits()` answers the follow-up question: `qwen3.5-122b-a10b` has to drop to UD-IQ2 on a 48 GB box, and `kimi-k3` will never fit in 96 GB at any quant.

The default tier is small-active MoEs, and that choice is empirical.

---

## The Measurement That Sets the Default

Every throughput figure in the repo is a `print_timing` line in a log committed under [`docs/measurements/`](https://github.com/chazhyseni/litMoE/tree/main/docs/measurements). `.gitignore` excludes `*.log` everywhere, then explicitly re-includes `!docs/measurements/*.log` so the evidence can't be dropped by accident. Check it yourself:

```bash
grep -oE "eval time =.*tokens per second" docs/measurements/*.log
```

One machine, no GPU: AMD EPYC 7B13, 24 physical cores / 48 threads, **AVX2 only** (no AVX-512, no AMX), DDR4-3200, Google Cloud persistent disk at ~379 MB/s random and ~778 MB/s sequential.

| Model | Quant / size | Threads | Generation t/s | Date |
|---|---|---:|---|---|
| **gemma-4-26b-a4b** (MoE, 4B active, vision) | UD-Q4_K_XL, 17 GB | 24 | **9.0–12.7** resident; 1.6–4.6 on the first requests after each restart | 2026-09-16 |
| **Qwen3.8-9B-Distill** (dense) | Q4_K_M, 6 GB | 8 | 8.3–8.5 | 2026-09-01 |
| Kimi-Linear-48B-A3B (MoE, 3B active) | Q4_K_M, 30 GB | 48 | 0.4–0.6 — **disk-bound, not model-bound** | 2026-08-20 |
| DeepSeek-V4-Flash | UD-IQ1_S, 83 GB | 48 | 0.32–0.34 — **disk-bound** | 2026-08-20 |

**The argument is the first two rows read together.** A 26B MoE with 4B active and a 9B dense model land in the same ~8–13 t/s band on the same hardware, because CPU throughput tracks the parameters *touched per token*, not the parameters stored. Same speed class — but one of them is dramatically stronger and multimodal. That is the whole reason the laptop tier is small-active MoEs rather than the largest dense model that technically fits.

The two disk-bound rows stay in the docs deliberately. Those runs used 48 threads on 24 physical cores (SMT oversubscription, since fixed — litmoe now defaults to physical cores) and never got their experts resident. **They measure the storage, not the model,** and labeling them as such is more useful than deleting them.

Two other figures *were* deleted. Earlier revisions quoted 0.69 t/s for Qwen3.8-9B and 0.85 t/s for Kimi-K3 from August runs whose logs a restart truncated before append-only logging existed. They may well have been right. They aren't reproducible from the repository, so they're gone.

---

## Getting an Engine and a Model, Once

```bash
litmoe doctor     # physical cores, RAM, AVX-512/AMX, NVIDIA GPUs, engines found, config sanity
litmoe models     # catalog by RAM tier, with fits / does-not-fit for this machine
litmoe init       # write models.yaml with fast defaults for this RAM
litmoe install    # install engines and/or download a model
litmoe serve      # gateway + all configured engines (Ctrl-C stops both)
litmoe status     # gateway health and per-engine state
litmoe stop       # stop only the engines litmoe started
```

`litmoe install --engine llamacpp` resolves the current `ggml-org/llama.cpp` release by following the `nightly-tag.txt` pointer, falling back to a prerelease scan when that tag has no asset for your platform yet. It selects from a 5-variant × 4-platform asset matrix (`cpu | cuda | cuda13 | vulkan | rocm`), auto-picking CUDA when an NVIDIA GPU is visible, confirms the binary actually runs, then symlinks `llama-server` into `~/.local/bin/`. Linux release binaries need **glibc ≥ 2.34**; older distributions fall back to a source build automatically, with BLAS and `GGML_NATIVE` enabled.

`litmoe install --model <id>` absorbs the parts that make hand-rolled GGUF downloads tedious: root-layout versus per-quant-subdirectory repos, sharded files pulled as a complete set, `mmproj` vision projectors chosen with F16 preference, and exclusions for MTP, imatrix, and vendor subdirectories that aren't quants. It then writes the `models.yaml` entry with a memory-aware context size and the right `extra_args`.

Not in the mood to pre-download? `litmoe init` writes `model_path` as a HuggingFace spec (`owner/repo:QUANT`) and llama-server fetches on first start.

---

## Supervision That Minds Its Own Business

This is the unglamorous half that decides whether a gateway survives daily use.

**Context sizing.** `n_ctx: 0` — or anything below `MIN_SANE_CTX = 16384` — means "use the model's native window." The gateway measures actual weight size, globbing sibling shards for local GGUFs and consulting the catalog for HF specs, then shrinks the context only as far as the KV cache requires to sit alongside those weights in RAM. The resolved value is **written back to `models.yaml`**, so the next start is deterministic. The README states the trade-off plainly: that rewrite doesn't preserve YAML comments.

**Port allocation.** Engines take 8081, 8082, … in `models.yaml` order, skipping the gateway's own port and **any port another process already holds** — established by an actual bind probe, not an assumption. A stray Ollama or LM Studio on 8081 won't break your startup, and a test that binds a real socket proves it.

**Process lifecycle.** Each engine spawns with `start_new_session=True` so it owns its process group. PID files live in `~/.litmoe/run/<id>.pid`. Readiness polls the engine's own `/health` every 2 seconds up to `LITMOE_READY_TIMEOUT` (300 s default). Shutdown is `killpg` SIGTERM → 15 s grace → SIGKILL. `SIGTERM`, `SIGINT`, and `SIGHUP` to the gateway stop every engine — litmoe installs a handler specifically so that uvicorn's signal re-raise runs engine shutdown instead of orphaning subprocesses.

**`litmoe stop` reads only the PID files.** No `pgrep` over process names. Your Ollama, your LM Studio, your hand-launched `llama-server` survive untouched unless you explicitly pass `--all`. There is a test called `test_gateway_never_kills_processes_it_did_not_start` that spawns a bystander process and asserts exactly that.

**Logs append** to `logs/<model-id>.log` with a `===== litmoe session` header and full argv per start — which is precisely why the Gemma measurement log can show seven restarts and explain its own cold-start dips instead of looking like noise.

---

## Integration You Don't Have to Undo

**The design rule:** *using a local model must never change what a harness does when you run it normally.*

litmoe never writes to `~/.claude/`, `~/.hermes/config.yaml`, `~/.hermes/.env`, or your shell rc. It reads only `LITMOE_*` variables and writes only under `~/.litmoe/` and your `models.yaml`. It never exports `ANTHROPIC_*` or `OPENAI_*` into your shell.

```bash
./scripts/claude-local                              # Claude Code → local model, isolated
./scripts/claude-local --model qwen3.6-35b-a3b -p "explain this repo"
claude                                              # normal Claude Code, still your Anthropic account
```

`claude-local` is 91 lines of bash that, **for one exec'd process only**:

1. **Unsets** six inherited credential variables (`ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, Bedrock, Vertex) so nothing leaks in either direction.
2. Sets `ANTHROPIC_BASE_URL` and a dummy `ANTHROPIC_AUTH_TOKEN`.
3. Sets `ANTHROPIC_MODEL`, all three `ANTHROPIC_DEFAULT_{SONNET,OPUS,HAIKU}_MODEL`, and `CLAUDE_CODE_SUBAGENT_MODEL` — so subagents and Claude Code's background haiku calls stay local too.
4. Sets `CLAUDE_CONFIG_DIR=~/.litmoe/claude-config`, keeping local sessions out of your real ones.
5. Sets `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`.
6. `exec`s `claude` — and every one of the above dies with that process.

**Verified end to end on 2026-09-16:** inside the wrapper, `claude auth status --text` reported the local base URL and `Auth token: ANTHROPIC_AUTH_TOKEN`, and a `-p` prompt completed with `modelUsage: gemma-4-26b-a4b`. Immediately afterwards, plain `claude auth status` still showed the Enterprise login, and the shell held no `ANTHROPIC_*` or `CLAUDE_*` variables.

The anti-patterns are documented just as explicitly — including one that litmoe's own earlier docs got wrong:

| Don't | Why |
|---|---|
| `export ANTHROPIC_BASE_URL=...` in your shell or rc | Every `claude` session *and every Anthropic SDK client* in that shell silently goes local |
| Put `ANTHROPIC_BASE_URL` in `~/.claude/settings.json` `env` | Global to every session — use a project's `.claude/settings.local.json` instead |
| Set `ANTHROPIC_API_KEY` globally to a dummy value | Claude Code prefers it over your subscription login |
| `hermes config set model.provider custom` | Rewrites the default profile; every later Hermes session goes local until you undo it |

Hermes gets three safe options instead: the per-process `scripts/hermes-local` wrapper, a cloned profile (`hermes profile create litmoe --clone`), or a `model_aliases:` entry with its **own** `api_key`. That last line matters more than it looks — Hermes refuses to reuse the default provider's credential for an alias endpoint, so an alias without its own key fails loudly instead of quietly shipping your real key to localhost.

`docs/HARNESSES.md` closes with a four-command checklist to run *before* concluding that something is broken.

---

## One File Configures All of It

```yaml
host: 127.0.0.1
port: 8080
api_key: null            # or a string to require Bearer / x-api-key auth

models:
  - id: gemma-4-26b-a4b
    engine: llamacpp
    model_path: unsloth/gemma-4-26B-A4B-it-GGUF:UD-Q4_K_XL   # HF spec, file, dir, or URL
    n_gpu_layers: -1
    n_ctx: 0                                                  # 0 = native, auto-reduced to fit RAM
    aliases: [claude-sonnet-4-5, claude-opus-4-1, claude-haiku-4-5]

  - id: glm-5.3-flash
    engine: ktransformers
    model_path: zai-org/GLM-5.3-Flash
    kt_method: FP8                                            # CPU expert backend
    kt_num_gpu_experts: 0
    n_ctx: 262144
    extra_args: ["--tool-call-parser", "glm47", "--reasoning-parser", "glm45"]
```

Mixed engines, one file, one flat model list at one endpoint. The nine `kt_method` backends are validated up front rather than at launch: `FP8`, `FP8_PERCHANNEL`, `BF16`, `RAWINT4`, `MXFP4`, and `MXFP8` need AVX-512; `AMXINT4` and `AMXINT8` need Intel AMX; `LLAMAFILE` runs GGUF experts on plain AVX2. `extra_args` passes through verbatim to whichever engine, and a `-t` there overrides the physical-core thread default. Duplicate ids or aliases are rejected by a pydantic validator at load time, not discovered at request time.

Auth is optional and behaves correctly when enabled: set `api_key` and the gateway accepts `Authorization: Bearer` **or** `x-api-key`, then **strips the Authorization header** before forwarding — because llama-server rejects Bearer tokens that don't match its own key.

---

## What It Deliberately Doesn't Do

- **No inference code, weights, kernels, or quantization.** The engines do all compute.
- **No multi-node distribution.** Single node.
- **No model conversion.** Use `llama-quantize`, Unsloth, or pre-quantized GGUFs.
- **No fine-tuning.** For LoRA on MoE experts, see the ktransformers × LlamaFactory cookbook upstream.

Each of those is a boundary that keeps this at 3,897 lines instead of 40,000.

---

## Why This Matters

Local-inference discourse is dominated by benchmarks and quantization schemes. The actual blocker sits further upstream: **you can't use a local model your tools can't reach, and you won't keep using one whose setup broke the tools you already had.**

litmoe takes three positions on that.

**Use the mature engines.** llama.cpp and ktransformers have years of specialist work behind them; the dispatcher's job is to make them reachable, not to compete. That position was bought at full price — by writing a forward pass that ran 45× slower and then deleting it.

**Default to fast, not merely to fitting.** A catalog that understands active-versus-total parameters and per-architecture KV cost is the difference between "technically runs" and "usable in a chat loop." On a laptop, 4B active at 17 GB beats 31B dense at 19 GB, and the catalog encodes that rather than leaving it to folklore.

**Make integration reversible.** Per-process environment, per-profile config, PID-file-scoped shutdown. Nothing global, nothing to undo, nothing that quietly redirects a tool you weren't thinking about.

The payoff is mundane in the best way: a 48 GB laptop chats with a multimodal 26B MoE at double-digit tokens per second, a 512 GB CUDA server runs GLM-5.3-Flash at native FP8 through sglang-kt, and **both answer on `http://127.0.0.1:8080/v1` under a name you chose** — to curl, to Open WebUI, to aider, and to Claude Code, which never finds out it isn't talking to Anthropic.

---

## Try It

```bash
git clone https://github.com/chazhyseni/litMoE
cd litMoE
pip install -e .            # use the SAME Python for install and serve

litmoe doctor               # hardware, engines, recommended models for your RAM
litmoe install              # installs llama.cpp and lists models that fit
litmoe install --model gemma-4-26b-a4b   # 17 GB; writes the entry into models.yaml
litmoe serve

curl http://127.0.0.1:8080/v1/models
```

Then point a harness at it, without touching that harness's configuration:

```bash
./scripts/claude-local -p "explain this repo"
claude                      # still your Anthropic account
```

**Repo:** [github.com/chazhyseni/litMoE](https://github.com/chazhyseni/litMoE)
**Architecture:** [docs/ARCHITECTURE.md](https://github.com/chazhyseni/litMoE/blob/main/docs/ARCHITECTURE.md) ·
**Design rationale and the 0.019 t/s post-mortem:** [docs/METHODOLOGY.md](https://github.com/chazhyseni/litMoE/blob/main/docs/METHODOLOGY.md) ·
**Raw benchmark logs:** [docs/measurements/](https://github.com/chazhyseni/litMoE/tree/main/docs/measurements) ·
**Harness isolation:** [docs/HARNESSES.md](https://github.com/chazhyseni/litMoE/blob/main/docs/HARNESSES.md)

Apache 2.0.
