# 🧠 Coding Agent Decode

> A complete, reverse-engineered masterclass on how production AI coding agents actually work under the hood.

---

## 🗺️ Master Curriculum

Start with the 5-Level Mastery Path:
👉 **[CURRICULUM.md](./teaching-material/CURRICULUM.md)**

---

## 📖 The Core Modules

| Level | Module | Core Concept |
|---|---|---|
| **Level 1** | [**The Stateless Brain & Runtime Illusion**](./modules/level-1-the-stateless-brain.md) | Why LLMs have zero memory, the Film Strip analogy, and $O(N^2)$ quadratic token growth. |
| **Level 2** | [**The Dual-Loop State Machine**](./modules/level-2-the-dual-loop-state-machine.md) | Inner reflex tool loop vs. Outer goal sequencing loop. |
| **Level 2.5** | [**Inner vs. Outer Loop Full Workflow**](./teaching-material/06-inner-vs-outer-loop-workflow.md) | 🔄 Detailed architecture flowchart, lifecycle trace, and comparison table. |
| **Level 3** | [**Priority Interrupts & Steering**](./modules/level-3-priority-interrupts-and-steering.md) | Mid-flight human steering (`steer()`) vs. deferred follow-up tasks (`followUp()`). |
| **Level 4** | [**Tool Execution & Destruction Guards**](./modules/level-4-tool-execution-and-destruction-guards.md) | The Scalpel vs. Sledgehammer: exact string replacement and truncation abort gates. |
| **Level 5** | [**Long-Horizon Autonomy & Compaction**](./modules/level-5-long-horizon-autonomy-and-compaction.md) | How agents survive 10-hour sessions via automatic context compaction at 80% token limits. |

---

## 🔬 In-Depth Engineering & Reverse Engineering Reports

* 🌟 [**Core Agentic Architecture Deep Dive**](./teaching-material/04-core-agentic-architecture-deep-dive.md) — Comprehensive technical breakdown of `agent-loop.ts`.
* 🔬 [**REA Investigation Report**](./teaching-material/05-rea-investigation-report.md) — Automated AST analysis verifying structural module bindings.
* 🛠️ [**Tools & Prompt Assembly**](./teaching-material/03-tools-and-prompt-assembly.md) — How system prompts and surgical tools are constructed.

---

## 🎮 Interactive HTML Simulators

You can open these directly in your browser:
* 🖥️ [**Dual-Loop & Steering Simulator**](./teaching-material/deep-loop-explorer.html) — Live interactive terminal to trigger turns, steering interrupts, and follow-ups.
* 📦 [**Pi Architecture Explorer**](./teaching-material/pi-architecture-explorer.html) — Monorepo package explorer with step-by-step walkthrough.

