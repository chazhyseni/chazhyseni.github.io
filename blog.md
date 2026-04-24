# Building an Auto-Learning AI Agent Harness

> **TL;DR:** `ai-skillweave` aggregates ~450 on-disk skills from 5 open-source skill libraries and delivers them natively to Claude Code, Codex, OpenClaw, Pi, and Copilot CLI. It runs 10 MCP servers for Claude Code — 8 for Copilot CLI — (memory, sequential thinking, browser automation, codebase context, docs lookup, token optimization, and more). And it learns from your corrections automatically — capturing them in real-time via hooks and distilling them into reusable skills via a 4-stage pipeline. One `./install.sh` configures everything.

---

## The Problem: Every Agent Harness Is Incomplete

AI agents are only as good as their context. Claude Code, Codex, OpenClaw, Pi, Copilot CLI — each ships with zero skills, zero MCP servers, and zero memory of what you taught them yesterday. You get a blank slate every session.

The existing workarounds don't solve this:
- **Custom instructions** (`CLAUDE.md`) — static, manual, never updated
- **Conversation history** — buried in JSON files the agent never reads
- **Copy-paste prompts** — fragile, version-drifted, lost in terminal scrollback
- **Per-agent configs** — skills you install for Claude Code don't exist for Codex or Pi

None of them aggregate. None of them learn. None of them propagate.

The goal isn't to use five agents simultaneously. It's to have the best possible harness — regardless of which agent you choose for a given task. Whether you prefer Claude Code for deep codebase work, Codex for rapid prototyping, or Copilot CLI for quick edits, the harness should be the same: fully loaded, context-aware, and improving over time.

---

## What ~450 Skills Actually Do

Before explaining the learning mechanism, let's be clear about what "skills" means here. These aren't vague prompt snippets. Each skill is a structured `SKILL.md` file with:

- **Condition** — when to apply this skill (`when running tests`, `when writing API routes`)
- **Strategy** — what to do (`validate output before declaring success`, `use absolute paths`)
- **Example** — concrete code or command showing correct vs. incorrect
- **Anti-pattern** — what to avoid
- **Tools** — which MCP tools or shell commands to use

**The aggregation story:** These ~450 on-disk skills come from 5 distinct sources. No single library has this breadth. Anthropic's 17 official skills cover software engineering fundamentals. OpenAI's 44 Codex-curated skills add API design, testing, and security patterns. ECC contributes 184 community-distilled skills spanning 15+ languages, architecture, DevOps, and data science. K-Dense adds 134 scientific skills — bioinformatics pipelines, cheminformatics, proteomics, clinical research, and 100+ database integrations. ClawBio ships 56 executable Python scripts for molecular dynamics and drug discovery, not just text prompts. **This is the only system that combines all of them.**

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

The total is **~450 on-disk skills** synced to every harness. They all load automatically — you don't invoke them explicitly.

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

**The combined cache** concatenates all ~450 skills into a single `combined-skills.txt`. Claude Code's prompt caching means this block costs ~90% less after the first session each day. Full injection beats selective loading because the agent automatically applies the right skill without explicit `/skill` invocation.

---

## The 10 MCP Servers: What They Actually Do

MCP servers are tools the agent can call. Most agent setups have zero. `ai-skillweave` configures 10 for Claude Code out of the box (8 for Copilot CLI):

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
| **github** | GitHub API integration — issues, PRs, repos, search | GitHub operations without leaving the terminal |
| **exa-web-search** | Neural web search with structured results | Finding papers, docs, or recent releases not in training data |

The **codesight** integration is worth highlighting separately. When Claude Code starts in any repo, a PreToolUse hook checks if the agent is about to do a broad `grep` or `glob` search. Instead of expensive recursive scanning, it calls `codesight --mcp` to get a structured map of routes, database schema, components, and dependencies. The agent gets context instantly instead of burning tokens exploring files.

---

## Cross-Harness Delivery: Learn Once, Use Everywhere

Every skill library was built for one harness. `ai-skillweave` extends them to load natively into five:

| Harness | Skills | Mechanism |
|---------|--------|-----------|
| **Claude Code** | 385 native + combined cache | `~/.claude/skills/` SKILL.md dirs + `--append-system-prompt-file` for cache |
| **OpenClaw** | 456 skill directories | `~/.openclaw/workspace/skills/` (YAML frontmatter sanitized) |
| **Codex** | 401 skill directories | `~/.codex/skills/` (YAML frontmatter sanitized) |
| **Pi** | 455 skill directories | `~/.pi/agent/skills/` (symlinked from ECC) |
| **Copilot CLI** | MCP servers only | `~/.copilot/mcp-config.json` — no native SKILL.md support yet |

**One `./install.sh`** sets up all five harnesses. **`./safe-install.sh`** does a zero-risk reinstall. **`update-ecc.sh`** pulls upstream skill library updates and rebuilds caches without touching your configs.

**The key insight:** When you correct Claude Code on a pattern, that learned skill propagates to OpenClaw, Codex, and Pi automatically. You don't re-teach each agent. The correction becomes institutional knowledge.

**Copilot CLI caveat:** Copilot reads MCP servers from `~/.copilot/mcp-config.json` (memory, codesight, etc.) but does not load `SKILL.md` directories natively — it uses `.github/copilot-instructions.md` for per-repo guidance. The MCP servers alone are a major upgrade over vanilla Copilot.

---

## Two Learning Mechanisms

### 1. BMO-Style Real-Time Capture (experimental)

Inspired by [Joel Hans' BMO agent](https://github.com/joelhans/bmo-agent), a `UserPromptSubmit` hook fires on every message and detects three signal types:

| Signal | Example | What gets saved |
|--------|---------|-----------------|
| **Correction** | "No, use absolute paths" | `correction` event JSON |
| **Preference** | "I always want type hints" | `preference` event JSON |
| **Pattern** | "Best practice: validate first" | `pattern` event JSON |

Events go to `~/.claude/skills/learned/events/`. At session end, a consolidation script clusters similar events and writes a `SKILL.md` file with a short imperative name like `verify-output-completeness`.

**Current status:** Hook is installed and firing. Consolidation pipeline exists. Event volume is low because it requires real-session usage to trigger.

### 2. Batch Pipeline (ALMA-inspired)

For bulk distillation from conversation history, a 4-stage pipeline extracts recurring patterns:

1. **Ingest** — parse conversation JSON, classify corrections via expanded pattern matching (direct negations, near-miss signals, clarifications, output rejections, polite redirections)
2. **Learn** — group by semantic similarity using sentence-transformer embeddings (`all-MiniLM-L6-v2`, cosine ≥ 0.72) with agglomerative clustering; session-level deduplication prevents repeated corrections from inflating counts
3. **Consolidate** — abstract each cluster into condition + strategy + anti-pattern via LLM distillation (Ollama HTTP API); no brittle keyword templates — the LLM generates precise skill text from 3–5 representative corrections per cluster
4. **Output** — write `SKILL.md` with YAML frontmatter

**LLM distillation:** Ollama integration uses the standard HTTP API (`localhost:11434/api/generate`). `--llm` is default; `--no-llm` disables it. When active, it produces proper condition/strategy/anti-pattern triplets from raw corrections, even for patterns never seen before.

**The quality gates remain aggressive:** empty, generic, or single-project patterns get rejected. The pipeline catches nuanced feedback that previously slipped through — "that won't work because...", "I meant...", "can you instead..." — and clusters them semantically even when wording differs.

**Current output:** The pipeline produces 5–8 high-signal skills per run from ~4,500 corrections. Yield is intentionally conservative (precision over recall), but extraction breadth and clustering accuracy have improved substantially. End-to-end runtime is ~3 minutes with parallel LLM distillation.

Both paths write to `~/.claude/skills/learned/` using the same ECC-compatible `SKILL.md` format. `sync-learned-skills.sh` propagates them to all harnesses.

---

## What Works Today

**Static skill aggregation works out of the box.** ~450 skills from 5 libraries load automatically into Claude Code, Codex, OpenClaw, Pi, and Copilot CLI (via MCP). One `./install.sh` sets up MCP servers, harness configs, and global instructions. The combined skills cache injects into Claude Code's system prompt with ~90% token savings via prompt caching.

**Auto-learning runs end-to-end.** The batch pipeline ingests conversation histories, clusters corrections by semantic similarity, distills them into structured `SKILL.md` files via local LLM, and syncs the results to all harnesses automatically. Quality gates are strict — most patterns get rejected — so the yield is low but the signal is high.

**Bootstrap installation handles fresh systems.** `install.sh` auto-detects and installs missing prerequisites (git, Node.js, Ollama, Python) so you don't have to.

---

## Known Gaps

- **Generated skill quality has minor cosmetic issues.** The LLM distillation prompt instructs complete sentences ending with periods, but the SKILL.md renderer unconditionally appends another period to the condition field, producing double periods in some outputs. The `_build_description` verb mapping can also produce awkward phrasing (e.g., "Avoid prioritize" when the strategy starts with a verb and the memory type is `anti_pattern`). These are rendering bugs, not skill logic issues — the skills load and fire correctly.
- **Copilot CLI has no native SKILL.md support.** Skills must be injected via `.github/copilot-instructions.md` per repository. MCP servers (memory, codesight, etc.) work globally, but the skill library itself is unavailable there.
- **Pipeline is batch-speed by design.** ~3 minutes end-to-end is acceptable for a daily cron job, but real-time feedback during a session would require caching distilled outputs by correction hash.

---

## Why This Matters

Most AI agent setups are stateless. You start fresh every session. The agent has no memory of what worked, no access to domain expertise, and no tools beyond its training cutoff.

`ai-skillweave` builds stateful harnesses in three ways:

1. **Skill library** — ~450 distilled patterns load automatically. The agent doesn't guess conventions; it knows them.
2. **MCP servers** — 10 tools extend what the agent can *do* (browse, remember, look up docs, analyze tokens, query knowledge graphs, search the web).
3. **Learning pipeline** — Corrections become skills that propagate to all harnesses. The harness improves between sessions.

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

Then correct your agent. Watch `~/.claude/skills/learned/events/` fill up. Run `python3 scripts/consolidate-learnings.py` to see what it learned.

**Repo:** [github.com/chazhyseni/ai-skillweave](https://github.com/chazhyseni/ai-skillweave)  
**Full catalog:** [docs/SKILLS-CATALOG.md](https://github.com/chazhyseni/ai-skillweave/blob/main/docs/SKILLS-CATALOG.md)
