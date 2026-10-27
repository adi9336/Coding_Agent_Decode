# 🗺️ The AI Agent Engineering Mastery Path
### A Reverse-Engineered Curriculum: From Zero to Building Production Coding Agents

This curriculum is structured into **5 progressive levels**. Each level answers one central engineering question, provides practical exercises, and references concrete source code from real production systems.

```
┌────────────────────────────────────────────────────────────────────────┐
│  LEVEL 1: The Stateless Brain & The Runtime Illusion                  │
│  "How does a model with 0 memory write 1,000 lines of working code?"  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  LEVEL 2: The Dual-Loop State Machine                                  │
│  "How do you separate reflexive tool calls from high-level goals?"     │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  LEVEL 3: Priority Interrupts & Steering                               │
│  "How do you stop an agent mid-flight without crashing the session?"   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  LEVEL 4: Tool Execution & Destruction Guards                          │
│  "Why LLMs break codebases and how surgical edits protect disks"       │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│  LEVEL 5: Long-Horizon Autonomy & Context Compaction                   │
│  "How agents survive 10-hour coding sessions within finite tokens"     │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🔰 LEVEL 1: The Stateless Brain & The Runtime Illusion

### 🎯 The Core Question
> *"If an LLM has zero memory between calls, how does a conversation continue?"*

### 🧠 The Conceptual Model
* An AI model is a pure mathematical function: `f(Prompt) -> Output`.
* The "Agent" is not the AI model. The agent is the **Runtime Harness** wrapping the model.
* The illusion of memory is maintained by replaying the entire history of user messages, thoughts, and tool results back to the model on every single turn.

### 🔬 Production Code Ground Truth
* [`pi/packages/agent/src/agent.ts`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent.ts#L83-L112): Notice how `createMutableAgentState` holds a plain JavaScript array `messages = []`.
* [`pi/packages/agent/src/agent-loop.ts`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent-loop.ts#L404): Notice how the internal rich messages (`AgentMessage[]`) are converted to provider format (`Message[]`) right before the network request.

### 🛠️ Practical Milestone 1
> Write a 30-line Node.js/Python script that calls an LLM, appends the output to an array, and feeds it back. Observe how token costs scale quadratically with message length.

---

## ⚡ LEVEL 2: The Dual-Loop State Machine

### 🎯 The Core Question
> *"Why do production agents use TWO nested loops instead of a single `while (true)`?"*

### 🧠 The Conceptual Model
* **The Reflex Loop (Inner Loop)**: Fast, atomic, and reactive. Executes tool calls (`read`, `edit`, `bash`) until the model stops asking for tools.
* **The Goal Sequencer (Outer Loop)**: The job supervisor. Stays alive after the inner loop finishes to check if there are queued follow-up tasks.

### 🔬 Production Code Ground Truth
* [`agent-loop.ts#L188`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent-loop.ts#L188): `while (true)` (Outer Loop)
* [`agent-loop.ts#L192`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent-loop.ts#L192): `while (hasMoreToolCalls || pendingMessages.length > 0)` (Inner Loop)

### 🛠️ Practical Milestone 2
> Open [**`deep-loop-explorer.html`**](file:///C:/Users/adi/CODING_AGENT_DECODE/teaching-material/deep-loop-explorer.html) in your browser. Click **"Run Next Turn"** to watch the inner loop cycle through tool calls until completion, and observe when the outer loop goes idle.

---

## 🚗 LEVEL 3: Priority Interrupts & Steering

### 🎯 The Core Question
> *"If the agent begins deleting the wrong directory or going down a rabbit hole, how do you steer it mid-flight without killing the process?"*

### 🧠 The Conceptual Model
* Most simple agents lock the terminal while running. If you hit `Ctrl+C`, you kill the process and lose your work.
* Production agents implement two distinct message queues:
  1. **Steering Queue (`steer()`)**: Checked **between tool calls** inside the inner loop. The agent incorporates your urgent instruction immediately on the next step.
  2. **Follow-Up Queue (`followUp()`)**: Checked **only when the agent is idle** in the outer loop. Queues future tasks without interrupting current focus.

### 🔬 Production Code Ground Truth
* [`agent.ts#L149`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent.ts#L149): `PendingMessageQueue` implementation.
* [`agent-loop.ts#L185`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent-loop.ts#L185): `config.getSteeringMessages()` draining the steering queue before streaming the next turn.

### 🛠️ Practical Milestone 3
> In [**`deep-loop-explorer.html`**](file:///C:/Users/adi/CODING_AGENT_DECODE/teaching-material/deep-loop-explorer.html), run a turn and immediately click **"⚡ Steer (Urgent Interrupt)"**. Observe how the inner loop pauses tool execution to process your steering message first.

---

## 🛡️ LEVEL 4: Tool Execution & Destruction Guards

### 🎯 The Core Question
> *"Why do naive agents destroy codebases, and how do production agents guarantee safe edits?"*

### 🧠 The Conceptual Model
* **The Full-Rewrite Fallacy**: Asking an LLM to rewrite an entire file results in forgotten functions, stripped comments, and syntax errors.
* **Surgical Edits**: The agent must specify the **exact existing text** (`target`) and the **replacement text**. If a single whitespace character is wrong, the tool rejects the edit and forces the model to re-read.
* **Token Limit Truncation**: When an LLM runs out of output tokens mid-JSON, naive agents execute corrupted arguments. Production engines abort truncated tool batches immediately.

### 🔬 Production Code Ground Truth
* [`edit.ts`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/coding-agent/src/core/tools/edit.ts): The exact find-and-replace algorithm.
* [`agent-loop.ts#L488`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent-loop.ts#L488): `failToolCallsFromTruncatedMessage` aborting corrupted tool calls when `stopReason === "length"`.

### 🛠️ Practical Milestone 4
> Inspect [`03-tools-and-prompt-assembly.md`](file:///C:/Users/adi/CODING_AGENT_DECODE/teaching-material/03-tools-and-prompt-assembly.md) to study the exact JSON schema that makes tool execution deterministic.

---

## 📈 LEVEL 5: Long-Horizon Autonomy & Context Compaction

### 🎯 The Core Question
> *"How does an agent work on a repository for 6 hours straight without crashing the 200,000 token context window?"*

### 🧠 The Conceptual Model
* Conversations expand with every tool result (terminal logs, file contents).
* When context exceeds ~80% of model limits:
  1. The agent triggers an automated **Compaction Phase**.
  2. A separate background LLM call takes the oldest turns and drafts a structured summary (decisions made, files modified, pending tasks).
  3. The runtime replaces the raw historical turns with the summary block, shrinking context by 70%+ while preserving continuity.

### 🔬 Production Code Ground Truth
* [`packages/coding-agent/src/core/compaction`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/coding-agent/src/core/compaction): Token estimation, branch summaries, and automated compaction triggers.

### 🛠️ Practical Milestone 5
> Examine `packages/coding-agent/src/core/compaction/prepare-compaction.ts` to see how older conversation slices are extracted and summarized without discarding active system instructions.

---

## 🏁 Capstone Architecture Blueprint

By completing these 5 levels, you understand the architecture of modern AI coding agents:

```
[User Input]
     │
     ▼
[Settings & Auth Loader]
     │
     ▼
[System Prompt Builder] (Injects Rules + Skills + CWD)
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    OUTER LOOP (Follow-ups)                      │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              INNER LOOP (Reflexive Turns)               │   │
│   │                                                         │   │
│   │   1. Drain Steering Queue (User Interrupts)             │   │
│   │   2. Check Context Token Limit (Auto-Compact if > 80%)  │   │
│   │   3. Stream LLM Response (Tokens + Tool Calls)          │   │
│   │   4. Guard against Token Truncation                     │   │
│   │   5. Execute Tools (Surgical Edits, Sandboxed Bash)     │   │
│   │   6. Append Tool Results to Transcript JSONL            │   │
│   │                                                         │   │
│   └────────────────────────────┬────────────────────────────┘   │
│                                │                                │
│        Inner Loop Settles ─────┘                                │
│        Checks Follow-Up Queue                                   │
│        If tasks remain ──► Restart Inner Loop                   │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                   Both Queues Empty ──► IDLE
```

