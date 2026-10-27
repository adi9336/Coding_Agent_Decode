# 📚 Pi Agent Teaching & Architecture Library

This folder contains a complete reverse-engineered breakdown of the **Pi Coding Agent Architecture**, structured as a progressive learning path.

---

## 🗺️ Master Curriculum & Learning Path

👉 **[Start with the 5-Level Learning Path: `CURRICULUM.md`](./teaching-material/CURRICULUM.md)**  
*(A progressive, concept-first engineering guide from the Stateless Brain to the Dual-Loop Machine and Context Compaction)*

---

## 📖 Complete Level-by-Level Modules

| Level | Module Document | Core Topic |
|---|---|---|
| **Level 1** | [**`modules/level-1-the-stateless-brain.md`**](./teaching-material/modules/level-1-the-stateless-brain.md) | The Film Strip Analogy, Stateless LLMs, Network Wire Payloads |
| **Level 2** | [**`modules/level-2-the-dual-loop-state-machine.md`**](./teaching-material/modules/level-2-the-dual-loop-state-machine.md) | Inner Reflex Loop vs. Outer Goal Loop, Carpenter/Foreman Analogy |
| **Level 3** | [**`modules/level-3-priority-interrupts-and-steering.md`**](./teaching-material/modules/level-3-priority-interrupts-and-steering.md) | Mid-flight Steering vs Deferred Follow-up Queues |
| **Level 4** | [**`modules/level-4-tool-execution-and-destruction-guards.md`**](./teaching-material/modules/level-4-tool-execution-and-destruction-guards.md) | The Scalpel vs Sledgehammer (Surgical Edits), Truncation Aborts |
| **Level 5** | [**`modules/level-5-long-horizon-autonomy-and-compaction.md`**](./teaching-material/modules/level-5-long-horizon-autonomy-and-compaction.md) | Context Compaction, Surviving Token Limits, Long Sessions |

---

## 📑 Supporting Research & Simulators

1. [**`CURRICULUM.md`**](./teaching-material/CURRICULUM.md)
   * 🗺️ **The 5-Level Mastery Path**: Step-by-step roadmap with core questions, real-life analogies, and code references.

2. [**`01-pi-big-picture.md`**](./teaching-material/01-pi-big-picture.md)
   * High-level overview of what Pi is.
   * **The Restaurant Analogy**: Mapping Pi's 7 monorepo packages to kitchen roles.
   * End-to-end request lifecycle diagram.

3. [**`02-agent-core-engine.md`**](./teaching-material/02-agent-core-engine.md)
   * The **Think-Act-Observe** core agent loop.
   * **The Detective Analogy**: How an AI uses tools to solve programming tasks.
   * Parallel vs. Sequential tool execution.
   * Steering (`steer()`) vs. Follow-up (`followUp()`) message queues.
   * Event stream lifecycles.

4. [**`04-core-agentic-architecture-deep-dive.md`**](./teaching-material/04-core-agentic-architecture-deep-dive.md)
   * 🌟 **Masterclass Deep Dive**: The Dual-Loop architecture (Inner vs Outer).
   * Exact reverse-engineered mechanics from `agent-loop.ts`.
   * Edge-case hardening: token truncation guards, surgical edits, context compaction.

5. [**`05-rea-investigation-report.md`**](./teaching-material/05-rea-investigation-report.md)
   * 🔬 **REA Engine Report**: Automated reverse engineering analysis using `rea-agents` showing exact AST graphs, module node IDs, and verified structural bindings.

6. [**`03-tools-and-prompt-assembly.md`**](./teaching-material/03-tools-and-prompt-assembly.md)
   * The built-in tool suite (`read`, `edit`, `write`, `bash`, `grep`, `find`).
   * The surgical replacement technique: why exact text matching prevents hallucinated file rewrites.
   * How the system prompt is assembled dynamically.

7. [**`deep-loop-explorer.html`**](./teaching-material/deep-loop-explorer.html)
   * 🎮 **Interactive Dual-Loop Visual Simulator**: Live event terminal allowing you to inject mid-flight **Steering interrupts** and **Follow-up tasks** to see how the nested state machine prioritizes execution in real time.

8. [**`pi-architecture-explorer.html`**](./teaching-material/pi-architecture-explorer.html)
   * Monorepo package explorer and introductory simulator.
