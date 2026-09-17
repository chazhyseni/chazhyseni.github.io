# Running Big Models on Small Machines

> **TL;DR:** A coding assistant can run entirely on hardware you already own — nothing metered, requests never leave the machine, no vendor to depend on. The hardware isn't the obstacle; the plumbing is. `litMoE` is the plumbing: it works out which model fits your RAM, fetches it, starts the right engine, and puts everything at one local address that speaks the two API dialects your tools use. On an ordinary 24-core CPU with no GPU, a 26-billion-parameter model answers faster than you can read.

---

## The interesting thing isn't that this works. It's that it works on a CPU.

Claude Code ran end to end against a model on my own machine. Nothing was billed, and every request stayed on that machine.

That's possible on a CPU because of how these models are built — not because of anything new in the hardware. More on that in a moment, since it's the reason the default model in this project is what it is.

What stands between most people and that capability isn't the model or the machine. It's friction. Two excellent open-source engines solve the hard part:

- **llama.cpp** runs compressed models on almost anything — a MacBook, a gaming GPU, a bare CPU.
- **ktransformers** (Tsinghua University, published at SOSP 2025) splits a giant model between your GPU and your RAM so one machine can run something enormous.

Neither needs help doing math. Neither does anything *around* the math:

- They're separate programs with incompatible flags.
- Each serves one model on one port, so your tools must know which port is which.
- They speak the OpenAI API dialect, so Anthropic-format tools — Claude Code above all — can't reach them at all.
- Nothing tells you which of hundreds of available models will actually run on *your* machine. You find out when the process dies, often after a long download.

None of that is a research problem. It's just nobody's job, and it's where most people quit.

---

## Why a 26B model can be as fast as a 9B one

This is the one idea worth internalizing, because everything else follows from it.

Most models are *dense*: every parameter participates in producing every word. A 30-billion-parameter dense model does 30 billion parameters' worth of work per word, and on a CPU that work is dominated by hauling those numbers out of RAM.

**Mixture-of-Experts (MoE)** models are built differently. The model is split into many specialist sub-networks — "experts" — and a router picks only a handful for each word. A model can hold 26 billion parameters in total but touch only 4 billion of them per word. Those 4 billion are the *active* parameters.

Your CPU only pays for what it touches. So:

> A 26B model with 4B active runs at roughly the speed of a 9B dense model — and is a far better model.

I measured this on one machine — with a caveat I'll get to — and it's why litMoE's default recommendation on a laptop is a small-active MoE rather than the biggest dense model that technically fits. "Fits in RAM" and "fast enough to talk to" are different questions, and most tooling only answers the first.

---

## What litMoE actually is

A traffic cop. Nothing more.

```
  Your tools (Claude Code, Open WebUI, aider, curl, any OpenAI/Anthropic client)
        │
        │   one address: http://127.0.0.1:8080
        ▼
  litMoE gateway  ── reads which model you asked for, forwards to the right engine
        │
        ├──► llama-server              (llama.cpp)
        └──► sglang.launch_server      (ktransformers)
```

It contains no model weights, no math, no GPU code. It starts the engines as background programs, keeps an eye on them, shuts them down cleanly, and translates between API dialects in between. It adds a few milliseconds and zero computation.

**~3,900 lines of Python across 13 files.** That number is small on purpose, and it's small because of how version one went.

---

## Version one was an inference engine. It ran at 0.019 tokens per second.

I originally tried to write the math myself: a from-scratch CPU implementation in C. On a 24-core server it produced **0.019 tokens per second**. A four-token prompt — a few words — took 158 seconds just to begin answering.

The arithmetic explains it, and no amount of clever code was going to change it. For a 67-token prompt:

- The model needed **98,496 expert lookups** (67 tokens × 92 layers × 16 experts per layer).
- Each expert is 17.55 MB of weights.
- Even after skipping repeats, that's **~859 GB to read off disk** — 38 minutes at this disk's actual speed.
- The CPU work alone was another 82 minutes.
- llama.cpp ran the same model on the same machine **~45× faster**.

I'd already shipped hand-tuned math routines, memory-mapping tricks, and aggressive compression. All of it was irrelevant. The bottleneck was physics, not code.

So I deleted the engine. Three things survived:

1. **A 45× gap to a mature tool on identical hardware means your approach is wrong, not the hardware.**
2. **Measure before optimizing.** I calculated an 82-minute floor and then shipped several more rounds of "optimizations" before a stopwatch settled it.
3. **The boring integration layer was the actual product.**

---

## Making Claude Code talk to your own hardware

Claude Code speaks Anthropic's API. Local engines speak OpenAI's. They're similar but not compatible — different names for the same ideas, and a completely different format for streaming text as it's generated.

litMoE translates in both directions, including the hard part: streaming. As your local model produces text, litMoE re-packages it on the fly into exactly the sequence of events Claude Code expects — ordinary text, tool calls, and reasoning blocks all handled separately and in the right order. Claude Code behaves normally throughout; the only tell is a one-line notice that it doesn't recognize the model name.

The part I care about more is that using it **changes nothing**:

```bash
./scripts/claude-local          # Claude Code → your local model
claude                          # normal Claude Code, still your Anthropic account
```

The obvious approach — and what this project's own earlier docs recommended — is to `export ANTHROPIC_BASE_URL=...` in your shell. Don't. That quietly redirects *every* Claude Code session and every Anthropic client in that terminal until you remember to undo it. `claude-local` sets those variables for one single process and then hands off to `claude`. Nothing global is written. There's no cleanup step, because there's nothing to clean up.

Verified end to end: inside the wrapper, Claude Code reported the local address and answered from the local model. In the very next terminal, plain `claude` was still signed in to my normal account and the shell had no stray variables.

Hermes, Open WebUI, aider, and plain `curl` work the same way — one extra endpoint, nothing replaced.

---

## Knowing what will actually run

`litmoe doctor` looks at your machine — cores, RAM, CPU features, any NVIDIA GPU — and `litmoe models` prints a catalog of 29 models grouped by how much memory they need, marking which ones fit *you*.

| Your machine | What you can run |
|---|---|
| **48 GB laptop** | Gemma-4-26B-A4B (the default — fast, handles images, 256K context), Qwen3.6-35B, GPT-OSS-20B |
| **96 GB** | GPT-OSS-120B, Qwen3.5-122B, Llama-4-Scout |
| **192 GB workstation** | DeepSeek-V4-Flash, MiniMax-M2.7 |
| **512 GB server** | GLM-5.3, DeepSeek-V3.2, Kimi-K2.6 — up to a trillion parameters |
| **768 GB server** | Kimi-K3 (2.78 trillion), Qwen3.8 (2.4 trillion) |

Every entry was checked against the actual files on HuggingFace, so the sizes are real rather than estimated from the model's name.

The fit calculation is simple and written down: the compressed weights, plus about 8% overhead, plus the **KV cache** (working memory that grows with how long your conversation is), plus 4 GB for the operating system. On a Mac it budgets 75% of your RAM, since Apple's unified memory is shared with the graphics chip.

Then `litmoe install --model <name>` downloads it — handling models split across a dozen files, the separate vision component multimodal models need, and the several incompatible ways repositories are laid out — and writes the config for you. The context length is set to the model's maximum and trimmed only if the working memory wouldn't fit. Guessing is removed from the process.

---

## The numbers, with receipts

Every speed figure in the table below comes from a log file committed alongside it. You can grep them yourself. The `.gitignore` explicitly re-includes those logs so they can't be dropped by accident.

One machine, **no GPU**: a 24-core AMD EPYC server (48 hardware threads), AVX2 only — none of the newer vector instructions that help most here — and an ordinary cloud disk at roughly 400 MB/s.

| Model | Size on disk | Threads | Speed |
|---|---|---:|---|
| **Gemma-4-26B-A4B** — 26B total, 4B active, handles images | 17 GB | 24 | **9.0–12.7 tokens/sec** once loaded; 1.6–5.6 on the first request after each restart |
| **Qwen3.8-9B-Distill** — 9B dense | 6 GB | 8 | 8.3–8.5 tokens/sec |
| Kimi-Linear-48B | 30 GB | 48 | 0.4–0.6 — *bottlenecked by disk, not the model* |
| DeepSeek-V4-Flash, heavily compressed | 83 GB | 48 | 0.32–0.34 — *same problem* |

The first two rows are the point, with one honest caveat: they were run two weeks apart with different thread counts, so this is **not** a head-to-head race. What it does show is that both models land in the same **8–13 tokens/sec class** on this machine, despite one being three times the size. A 26B model has no business keeping up with a 9B one — unless only 4B of it is doing the work, which is exactly the case. That's the MoE effect, and it's why the laptop default is a small-active MoE and not the largest dense model that happens to fit.

Gemma's slow first requests are worth understanding rather than hiding: the engine was restarted seven times during that session, and each restart means re-reading 17 GB from disk before it's back in memory. Steady-state is the 9–12.7 band.

The bottom two rows stay published on purpose, and they're a lesson in what these numbers can and can't tell you. Those models never fit in memory, so they were being read off the disk continuously — the figures describe my storage, not the models. They also used 48 threads on 24 physical cores, which oversubscribes the CPU and hurts rather than helps. litMoE now defaults to physical cores for exactly that reason.

Two other figures were **deleted** from the docs. They came from runs whose logs got overwritten before I fixed how logging worked. They were probably accurate. They aren't reproducible, so they're gone.

---

## Running it

```bash
litmoe doctor     # what's in this machine, what's installed, what to run
litmoe models     # the catalog, with fits / doesn't-fit for you
litmoe install    # get an engine, get a model
litmoe serve      # start everything; Ctrl-C stops everything
litmoe stop       # stop only the things litmoe started
```

That last line matters. Each engine runs in its own process group with its own recorded process ID, and `litmoe stop` only touches those. Your Ollama, your LM Studio, the `llama-server` you launched by hand this morning — all untouched. There's a test that starts an unrelated process and checks it's still alive afterwards.

Similarly, engines claim ports starting at 8081 but skip any port something else is already using — checked by actually trying to bind it, not by assuming. A stray program on 8081 won't break your startup.

Small things, but they're the difference between a tool you use daily and one you uninstall after it kills the wrong process once.

---

## What it deliberately doesn't do

No inference. No model conversion. No fine-tuning. No multi-machine clustering. Every one of those is a boundary that keeps this small — and each one is already somebody else's well-maintained project.

---

## Why this matters

The point isn't to stop paying for frontier models. On hard problems they're still better.

The point is where the **floor** is. A genuinely useful coding assistant runs on a machine you already own — which means it costs nothing per token, works on a plane, keeps client code and patient data on hardware you control, and can't be deprecated, rate-limited, or repriced out from under you. For regulated work, that last category isn't a preference. It's the difference between using these tools and not.

MoE architectures are what make that practical on a CPU. Plumbing is what's kept it out of reach anyway — and plumbing is a solvable problem that simply hadn't been anyone's job.

So litMoE takes three positions:

**Use the engines other people perfected.** llama.cpp and ktransformers have years of specialist work behind them. This project's job is to make them reachable, not to compete — a lesson I paid full price for.

**Fast beats fits.** Knowing the difference between active and total parameters is the difference between a model that technically loads and one you'll actually talk to. Most tooling answers the first question and leaves you to discover the second.

**Setup must be reversible.** Per-process settings, nothing global, nothing to undo. The fastest way to lose someone is to break the tool they had before you showed up.

The result is unglamorous in the best way: a laptop holds a real conversation with a capable multimodal model, a server can run something with a trillion parameters, and **both answer at the same local address under a name you chose** — to curl, to Open WebUI, and to Claude Code, which carries on as normal.

---

## Try it

```bash
git clone https://github.com/chazhyseni/litMoE
cd litMoE
pip install -e .

litmoe doctor                             # what does this machine support?
litmoe install --model gemma-4-26b-a4b    # 17 GB
litmoe serve
```

Then point Claude Code at it, without touching its configuration:

```bash
./scripts/claude-local -p "explain this repo"
claude                                    # still your Anthropic account
```

**Code:** [github.com/chazhyseni/litMoE](https://github.com/chazhyseni/litMoE) (Apache 2.0) ·
**How it's built:** [ARCHITECTURE.md](https://github.com/chazhyseni/litMoE/blob/main/docs/ARCHITECTURE.md) ·
**Why it's built that way:** [METHODOLOGY.md](https://github.com/chazhyseni/litMoE/blob/main/docs/METHODOLOGY.md) ·
**Raw benchmark logs:** [docs/measurements/](https://github.com/chazhyseni/litMoE/tree/main/docs/measurements)
