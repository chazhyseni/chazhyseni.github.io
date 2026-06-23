# Building an Auto-Learning AI Agent Harness

> **TL;DR:** `ai-skillweave` started with ~450 skills from 5 open-source libraries for Claude Code. It now ships **>2,500 unique skills** drawn from **14 upstream repos** to **6 harnesses** — Claude Code, Codex, OpenClaw, Pi, Copilot, and Hermes. Same one-command install, plus the 4-stage learning pipeline that turns your corrections into reusable skills across every harness.

---

## The Problem: Every Agent Harness Is Incomplete

AI agents are only as good as their context. Claude Code, Codex, OpenClaw, Pi, Copilot, Hermes — each has built-in tools, but without configuration they don't retain your corrections, domain expertise, or custom conventions across sessions. You start fresh every time.

The existing workarounds don't solve this:
- **Custom instructions** (`CLAUDE.md`) — static, manual, rarely updated
- **Conversation history** — buried in JSON files the agent rarely reads
- **Copy-paste prompts** — fragile, version-drifted, lost in terminal scrollback
- **Per-agent configs** — skills you install for Claude Code don't exist for Codex or Pi

None of them aggregate. None of them learn. None of them propagate.

The goal isn't to use six agents simultaneously. It's to have the best possible harness — regardless of which agent you choose for a given task. Whether you prefer Claude Code for deep codebase work, Codex for rapid prototyping, or Copilot CLI for quick edits, the harness should be the same: fully loaded, context-aware, and improving over time.

---

## How It Grew: From ~450 to >2,500

The first install aggregated ~450 on-disk skills from 5 sources (Anthropic's reference set, OpenAI's Codex-curated skills, ECC's community patterns, K-Dense's scientific set, and ClawBio's executable Python pipelines). That worked. Then it kept growing:

| Milestone | Skills | Sources | Harnesses |
|-----------|-------:|--------:|----------:|
| **v1 — initial release** | ~450 | 5 | 5 |
| **+ bioSkills, SciAgent, OpenClaw-Medical, operon** | ~2,000 | 9 | 5 |
| **+ ToolUniverse, DeepMind, Bipartite** | ~2,400 | 12 | 5 |
| **+ BioNeMo (GPU bio), Nature-Paper (manuscript workflows)** | **>2,500** | **14** | 5 |
| **+ Hermes** | **>2,500** | **14** | **6** |

The aggregation story is the same now as it was then — no single library has this breadth, and that's the point. What's changed is coverage: clinical workflows (OpenClaw-Medical, 896 skills), bioinformatics protocols (operon, bioSkills, SciAgent), drug discovery (ToolUniverse, K-Dense), GPU-accelerated structural biology (BioNeMo), manuscript workflows (Nature-Paper), and a long tail of single-cell, genomics, and proteomics tools. **14 repos, no overlap, every harness gets the full pool.**

---

## What 2,652 Skills Actually Do

Before explaining the learning mechanism, let's be clear about what "skills" means here. These aren't vague prompt snippets. Each skill is a structured `SKILL.md` file with:

- **Condition** — when to apply this skill (`when running tests`, `when writing API routes`)
- **Strategy** — what to do (`validate output before declaring success`, `use absolute paths`)
- **Example** — concrete code or command showing correct vs. incorrect
- **Anti-pattern** — what to avoid
- **Tools** — which MCP tools or shell commands to use

**The aggregation story:** These 2,652 unique skills come from 14 distinct sources. Anthropic's 17 official skills cover software engineering fundamentals. ECC contributes 272 community-distilled skills spanning 15+ languages, architecture, DevOps, and data science. OpenClaw-Medical ships 896 clinical workflows meta-aggregated from 12+ repos. operon adds 556 bioinformatics protocols (RNA-seq, scRNA-seq, ATAC-seq, ChIP-seq, WGS/WES, spatial, proteomics, GWAS). bioSkills adds 546 across 63 bioinformatics categories. SciAgent, K-Dense, ClawBio, ToolUniverse, DeepMind, Bipartite, BioNeMo, and Nature-Paper round out the long tail — drug discovery, scientific writing, GPU-accelerated bio, manuscript workflows, executable pipelines. **No single library has this breadth on its own.**

Having this loaded into the system prompt means the agent doesn't guess your conventions — it already knows them. Here's what that looks like in practice:

| Domain | Example Skill | What It Prevents |
|--------|---------------|------------------|
| **Testing** | `tdd-workflow` | Agent writing code without tests, then backfilling |
| **Security** | `security-review` | Agent generating SQL injection vulnerabilities |
| **API Design** | `api-design` | Inconsistent REST conventions, missing pagination |
| **DevOps** | `docker-patterns` | Multi-stage build anti-patterns, layer caching mistakes |
| **Bioinformatics** | `rnaseq-de` | Agent using wrong normalization, missing batch correction |
| **Drug Discovery** | `rdkit` | Cheminformatics errors, wrong descriptor calculations |
| **Clinical** | `clinical-decision-support` | Hallucinated medical evidence, wrong dosing |
| **Manuscript** | `nature-revision-response` | Reviewer-comment replies that ignore the actual critique |
| **GPU-accelerated bio** | `bionemo-boltz2` | Wrong protein-structure model selection, missing MSA |

The total is **2,652 unique skills** synced to every harness. They all load automatically — you don't invoke them explicitly.

---

## Performance Impact: What Skills + MCP Actually Change

Skills and MCP servers aren't theoretical conveniences. They change what the agent does and how well it does it:

| Without `ai-skillweave` | With `ai-skillweave` |
|-------------------------|----------------------|
| Agent guesses your testing conventions | Knows TDD workflow, property-based testing, mutation testing |
| Agent does broad `grep`/`glob` to understand codebase | `codesight` returns structured routes, schema, deps instantly |
| Agent hallucinates API docs from training cutoff | `context7` pulls real-time documentation for libraries released yesterday |
| Agent burns tokens exploring file structure | Pre-loaded skills skip exploration — agent applies conventions directly |
| Agent forgets your preferences between sessions | `memory` MCP persists key-value context across sessions |
| Agent can't reason about multi-step failures | `sequential-thinking` MCP decomposes complex debugging into steps |
| Agent produces generic bioinformatics advice | `skillgraph` answers pipeline questions with evidence-backed paths |

**The combined cache** concatenates all 2,652 skills into a single `combined-skills.txt`. Claude Code's prompt caching means this block costs ~90% less after the first session each day. Full injection beats selective loading because the agent automatically applies the right skill without explicit `/skill` invocation.

---

## The 8 MCP Servers: What They Actually Do

MCP servers extend what the agent can do beyond its built-in tools. `ai-skillweave` configures 8 for Claude Code and Copilot CLI out of the box:

| Server | What It Does | When You Need It |
|--------|-------------|------------------|
| **memory** | Persistent key-value storage across sessions | Agent remembers your preferences from yesterday |
| **sequential-thinking** | Decomposes complex problems into reasoning steps | Debugging multi-step failures, design decisions |
| **context7** | Real-time documentation lookup (up-to-date, not training-cutoff) | API you just installed, library released last week |
| **playwright** | Browser automation — click, type, screenshot, navigate | Testing web apps, scraping, visual verification |
| **google-docs-editor** | Read/write Google Docs programmatically | Collaborative docs, requirement specs |
| **token-optimizer** | Analyzes prompt/token usage and suggests compression | You're hitting context limits on large codebases |
| **codesight** | Generates structured codebase context maps (routes, schema, deps) | Agent starts work in an unfamiliar repo |
| **skillgraph** | Knowledge graph of bioinformatics skills with pipeline paths | Bioinformatics questions requiring pipeline reasoning |

The config also ships two **opt-in templates** for `github` (issues/PRs/repos) and `exa-web-search` (neural web search). They need API keys — uncomment the `_api_key_servers_commented` block in `configs/claude-mcp-servers.json` and re-run `scripts/setup-mcp.sh` to enable either.

The **codesight** integration is worth highlighting separately. When Claude Code starts in any repo, a PreToolUse hook checks if the agent is about to do a broad `grep` or `glob` search. Instead of expensive recursive scanning, it calls `codesight --mcp` to get a structured map of routes, database schema, components, and dependencies. The agent gets context instantly instead of burning tokens exploring files.

---

## Cross-Harness Delivery: Learn Once, Use Everywhere

Every skill library was built for one harness. `ai-skillweave` extends them to load natively into six:

| Harness | Mechanism |
|---------|-----------|
| **Claude Code** | `~/.claude/skills/` SKILL.md dirs + `--append-system-prompt-file` for cache |
| **OpenClaw** | `~/.openclaw/workspace/skills/` (YAML frontmatter sanitized) |
| **Codex** | `~/.codex/skills/` (YAML frontmatter sanitized) |
| **Pi** | `~/.pi/agent/skills/` (symlinked from ECC) |
| **Copilot CLI** | `~/.copilot/mcp-config.json` (MCP servers; native SKILL.md support pending) |
| **Hermes** | `~/.hermes/skills/ai-skillweave/` (real copies; native toolsets preserved) |

**One `./install.sh`** sets up all six harnesses. **`./safe-install.sh`** does a safer reinstall (backs up configs first). **`update-ecc.sh`** pulls upstream skill library updates, runs the manifest-based prune against deleted sources, and rebuilds caches without touching your configs.

**The key insight:** When you correct Claude Code on a pattern, that learned skill propagates to OpenClaw, Codex, Pi, Copilot, and Hermes automatically. You don't re-teach each agent. The correction becomes institutional knowledge across the entire harness fleet.

**Copilot CLI caveat:** Copilot reads MCP servers from `~/.copilot/mcp-config.json` (memory, codesight, etc.) but does not load `SKILL.md` directories natively — it uses `.github/copilot-instructions.md` for per-repo guidance. The MCP servers alone are a major upgrade over vanilla Copilot.

**Hermes note:** Hermes receives the full 2,652-skill pool as real copies in its own `ai-skillweave/` category. Hermes's native toolsets and the openclaw-imports staging set are left untouched.

---

## Two Learning Mechanisms

### 1. Real-Time Capture

A `UserPromptSubmit` hook fires on every message and detects three signal types:

| Signal | Example | What gets saved |
|--------|---------|-----------------|
| **Correction** | "No, use absolute paths" | `correction` event JSON |
| **Preference** | "I always want type hints" | `preference` event JSON |
| **Pattern** | "Best practice: validate first" | `pattern` event JSON |

Events go to `~/.claude/skills/learned/events/`. At session end, a consolidation script clusters similar events and writes a `SKILL.md` file with a short imperative name like `verify-output-completeness`.

### 2. Batch Pipeline

For bulk distillation from conversation history, a 4-stage pipeline extracts recurring patterns:

1. **Ingest** — parse conversation JSON, classify corrections via expanded pattern matching (direct negations, near-miss signals, clarifications, output rejections, polite redirections)
2. **Learn** — group by semantic similarity using sentence-transformer embeddings (`all-MiniLM-L6-v2`, cosine ≥ 0.72) with agglomerative clustering; session-level deduplication prevents repeated corrections from inflating counts
3. **Consolidate** — abstract each cluster into condition + strategy + anti-pattern via LLM distillation (Ollama HTTP API); no brittle keyword templates — the LLM generates precise skill text from 3–5 representative corrections per cluster
4. **Output** — write `SKILL.md` with YAML frontmatter

**Why the batch pipeline matters:** Batched LLM distillation runs **30–50× faster than naive per-group LLM calls** — ~580 groups processed in 2–5 minutes via batched prompts + 16 parallel workers + HTTP connection pooling, instead of 4.8 hours sequentially. That's what makes "distill every conversation" tractable on a daily cron.

**LLM distillation:** Ollama integration uses the standard HTTP API (`localhost:11434/api/generate`). `--llm` is default; `--no-llm` disables it. When active, it produces proper condition/strategy/anti-pattern triplets from raw corrections, even for patterns never seen before.

**The quality gates remain aggressive:** empty, generic, or single-project patterns get rejected. The pipeline catches nuanced feedback that previously slipped through — "that won't work because...", "I meant...", "can you instead..." — and clusters them semantically even when wording differs.

**Current output:** The pipeline produces 5–8 high-signal skills per run from ~3,000 corrections. Yield is intentionally conservative (precision over recall), but extraction breadth and clustering accuracy have improved substantially.

Both paths write to `~/.claude/skills/learned/` using the same ECC-compatible `SKILL.md` format. `sync-learned-skills.sh` propagates them to all six harnesses.

---

## What Works Today

**Static skill aggregation works out of the box.** 2,652 skills from 14 libraries load automatically into Claude Code, Codex, OpenClaw, Pi, Copilot, and Hermes. One `./install.sh` sets up MCP servers, harness configs, and global instructions. The combined skills cache injects into Claude Code's system prompt with ~90% token savings via prompt caching.

**Auto-learning runs end-to-end.** The batch pipeline ingests conversation histories, clusters corrections by semantic similarity, distills them into structured `SKILL.md` files via local LLM, and syncs the results to all six harnesses automatically. Quality gates are strict — most patterns get rejected — so the yield is low but the signal is high.

**Bootstrap installation handles fresh systems.** `install.sh` auto-detects and installs missing prerequisites (git, Node.js, Ollama, Python) so you don't have to.

**Manifest-based pruning keeps state clean.** When a source repo deletes skills, `update-ecc.sh` removes the corresponding entries on next run — no orphaned skill directories piling up.

---

## What's Live vs. What's Next

| Status | Component | What It Does Today |
|--------|-----------|-------------------|
| **Live** | **Static skill aggregation** | 2,652 skills from 14 libraries load automatically into Claude Code, Codex, OpenClaw, Pi, Copilot, and Hermes. Each harness gets the full pool; delivery format is the only thing that differs. |
| **Live** | **MCP server suite** | 8 servers configured out of the box: memory, sequential thinking, browser automation, codebase context, documentation lookup, token optimization, knowledge graph, and Google Docs editing. Two more (GitHub, exa-web-search) ship as opt-in templates in the config. |
| **Live** | **Batch learning pipeline** | Ingests conversation history, clusters corrections by semantic similarity, distills them into structured `SKILL.md` files via local LLM (30–50× faster than naive per-group calls), and syncs to all six harnesses. Designed to run as a daily cron job. |
| **Live** | **Real-time hook infrastructure** | `UserPromptSubmit` and `PreToolUse` hooks are installed and firing. Events are captured to `~/.claude/skills/learned/events/`; session-end consolidation script clusters them automatically. |
| **Live** | **Manifest-based prune** | `update-ecc.sh` reconciles installed skills against the source manifest on each run — deleted upstream skills disappear locally without manual cleanup. |
| **In progress** | **Skill rendering polish** | The distillation pipeline produces valid, loadable skills. Minor cosmetic refinement is ongoing — sentence-boundary handling in the condition field and verb-mapping in descriptions. |
| **In progress** | **Real-time volume** | The hooks capture events correctly, but meaningful learning requires sustained real-session usage. As session volume grows, the real-time path will overtake the batch pipeline as the primary learning source. |
| **Planned** | **Copilot CLI native skills** | Currently MCP-only. Native `SKILL.md` support for Copilot CLI is planned pending Microsoft/GitHub API availability. |

---

## Why This Matters

Most AI agent setups are stateless by default. Without configuration, they don't retain your corrections, domain expertise, or custom conventions across sessions.

`ai-skillweave` builds stateful harnesses in three ways:

1. **Skill library** — 2,652 distilled patterns load automatically. The agent doesn't guess conventions; it knows them.
2. **MCP servers** — 8 tools extend what the agent can *do* (browse, remember, look up docs, analyze tokens, query knowledge graphs, edit Google Docs), plus 2 opt-in API-key templates (GitHub, web search).
3. **Learning pipeline** — Corrections become skills that propagate to all six harnesses. The harness improves between sessions.

The result: whichever agent you choose for a task, it runs with the same loaded expertise and toolset. No re-teaching. No config drift.

---

## Try It

```bash
git clone https://github.com/chazhyseni/ai-skillweave
cd ai-skillweave
./install.sh
```

Configure just one harness:
```bash
./install.sh --only claude      # Claude Code + MCP + skills
./install.sh --only copilot     # Copilot CLI + MCP servers
```

Then correct your agent. Watch `~/.claude/skills/learned/events/` fill up. Run `python3 scripts/consolidate-learning.py` to see what it learned.

**Repo:** [github.com/chazhyseni/ai-skillweave](https://github.com/chazhyseni/ai-skillweave)  
**Full catalog:** [docs/SKILLS-CATALOG.md](https://github.com/chazhyseni/ai-skillweave/blob/main/docs/SKILLS-CATALOG.md)