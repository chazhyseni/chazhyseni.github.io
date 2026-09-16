# Blog

Notes on the tooling I build around AI agents and genomics workloads. Newest first.

---

## [Running Trillion-Parameter MoE Models Behind One Local API](blog-litmoe.md)

*September 2026 · [litMoE](https://github.com/chazhyseni/litMoE)*

`litmoe` puts **llama.cpp** and **ktransformers** behind a single endpoint that speaks both OpenAI and Anthropic. One `models.yaml`, one port, every model reachable by name — from a 4B-active MoE that chats at 9–12.7 t/s on a 24-core CPU with no GPU, to Kimi-K3 at 2.78T parameters.

A 29-model RAM-tiered catalog verified against HuggingFace, hardware detection that picks what actually fits, and per-process harness wrappers so pointing Claude Code at a local model never touches your Anthropic login. ~3,900 lines of Python, zero lines of inference code — because version one *did* write a forward pass, and it ran at 0.019 tokens/sec.

[Read the post →](blog-litmoe.md)

---

## [Building an Auto-Learning AI Agent Harness](blog-skillweave.md)

*June 2026 · [ai-skillweave](https://github.com/chazhyseni/ai-skillweave)*

`ai-skillweave` started with ~450 skills from 5 open-source libraries for Claude Code. It now ships **2,652 unique skills** drawn from 14 upstream repos to 6 harnesses — Claude Code, Codex, OpenClaw, Pi, Copilot, and Hermes.

Same one-command install, 8 pre-configured MCP servers, plus a 4-stage learning pipeline that turns your corrections into reusable skills across every harness — with batched LLM distillation running 30–50× faster than naive per-group calls.

[Read the post →](blog-skillweave.md)
