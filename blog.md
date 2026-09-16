# Blog

Notes on the tooling I build around AI agents and genomics workloads. Newest first.

---

## [Running Big Models on Small Machines](blog-litmoe.md)

*September 2026 · [litMoE](https://github.com/chazhyseni/litMoE)*

A small gateway that lets the AI tools you already use — Claude Code, Open WebUI, aider — talk to models running on your own hardware. It works out which model fits your RAM, starts the right engine, and puts everything at one address.

Why a 26-billion-parameter model can be as fast as a 9-billion one, how to point Claude Code at your laptop without breaking your Anthropic account, and what I learned writing an inference engine that ran at 0.019 tokens per second before deleting it.

[Read the post →](blog-litmoe.md)

---

## [Building an Auto-Learning AI Agent Harness](blog-skillweave.md)

*June 2026 · [ai-skillweave](https://github.com/chazhyseni/ai-skillweave)*

AI coding agents start every session from scratch. Correct one today and it forgets by tomorrow — and a skill you install for one agent doesn't exist for the next.

`ai-skillweave` fixes both: 2,652 skills from 14 open-source libraries, installed into all six major agent harnesses at once, plus a pipeline that turns your corrections into reusable skills that propagate everywhere.

[Read the post →](blog-skillweave.md)
