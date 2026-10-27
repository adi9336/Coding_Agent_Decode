# 🎓 The Engineering Masterclass: How AI Coding Agents Actually Work (Deep Dive)

You asked the exact right question: **"What is the actual core agentic architecture?"**

Most tutorials stop at: *"It's an LLM with tools in a loop."* That is like saying a modern jet engine is just a fan with fuel. Real production coding agents (like Pi, Claude Code, Cursor, Codex) rely on an intricate, hardened architecture to prevent infinite loops, token exhaustion, corrupted files, and lost context.

Here is the complete, unfiltered reverse-engineered blueprint of how an AI coding agent works under the hood.

---

## 🏛️ Phase 1: The Core Mental Model (The Brain is Stateless)

Before looking at code, you must understand one fundamental reality:

> **AI Models (LLMs) have ZERO memory.**
> Every time the agent asks Claude or GPT a question, it is completely fresh. The LLM remembers nothing from 2 seconds ago.

Therefore, an AI coding agent is **not** an AI that "does work".
An AI coding agent is a **Runtime Harness** that manages state, transcripts, files, and loops outside the LLM, feeding the LLM an ever-evolving illusion of continuous thought.

---

## 🔄 Phase 2: The Two Loops Architecture (Inner Loop vs. Outer Loop)

In `packages/agent/src/agent-loop.ts`, Pi reveals the gold standard of agent loop design: **Nested Dual Loops**.

```
                        ┌────────────────────────────────────────────────────────┐
                        │             🔄 OUTER LOOP (Follow-up Tasks)            │
                        │  Checks: Are there deferred follow-up tasks queued?    │
                        │                                                        │
                        │   ┌────────────────────────────────────────────────┐   │
                        │   │         ⚡ INNER LOOP (Tool Execution Turn)      │   │
                        │   │                                                │   │
                        │   │  1. Check User Steering (Interruption)         │   │
                        │   │  2. Stream LLM Response (Tokens & Tool Calls)  │   │
                        │   │  3. Execute Tools (read, edit, bash)           │   │
                        │   │  4. Append Results to Transcript               │   │
                        │   │                                                │   │
                        │   │  Repeat until:                                 │   │
                        │   │  - No more tool calls requested                │   │
                        │   │  - No immediate steering messages              │   │
                        │   └───────────────────────┬────────────────────────┘   │
                        │                           │                            │
                        │       Inner Loop Done ────┘                            │
                        │       Now checks Follow-up Queue                       │
                        │       If new task exists ──► RESTART INNER LOOP        │
                        └───────────────────────────┬────────────────────────────┘
                                                    │
                                     Both Queues Empty: IDLE
```

### Why two loops? Why not just one `while (true)`?

Let's use a real-life analogy: **The Carpenter and the Foreman**.

#### 1. The Inner Loop (The Carpenter Working on a Board)
* The carpenter is cutting wood.
* **Measure** ➔ **Saw** ➔ **Check measurement** ➔ **Sand down edge**.
* This is fast, atomic, and reactive. Every action immediately produces a physical result that feeds the next millisecond decision.
* **If the user taps the carpenter on the shoulder** (*"Wait! Don't cut that, cut this one instead!"*), this is a **Steering Message**. It interrupts the carpenter between saw strokes.

#### 2. The Outer Loop (The Job Site Checklist)
* Once the board is cut, the carpenter finishes that specific task.
* Now the foreman checks the clipboard: *"Great. Now that the frame is built, our next queued task is installing the hinges."*
* This is the **Follow-Up Queue**. It only triggers when the carpenter has put their tools down and is standing idle.

---

## 🔬 Phase 3: The Code Anatomy of `agent-loop.ts`

Here is the exact implementation logic found in Pi:

### 1. The Outer Loop
```typescript
// OUTER LOOP: Keeps running as long as follow-up work exists
while (true) {
    let hasMoreToolCalls = true;

    // INNER LOOP: Executes tool calls & immediate user steering
    while (hasMoreToolCalls || pendingMessages.length > 0) {
        
        // 1. Check for immediate steering (Interruption)
        // Did the user type something while the agent was running?
        pendingMessages = await config.getSteeringMessages();

        // 2. Stream the LLM response
        const message = await streamAssistantResponse(currentContext, config);

        // 3. Inspect: Did the model request tools?
        const toolCalls = message.content.filter(c => c.type === "toolCall");

        if (toolCalls.length > 0) {
            // 4. Run the tools safely (Read file, run bash, edit code)
            const results = await executeToolCalls(currentContext, toolCalls);
            // Append results so the LLM sees what happened
            currentContext.messages.push(...results);
            hasMoreToolCalls = true; // Continue inner loop!
        } else {
            hasMoreToolCalls = false; // LLM is done talking
        }
    }

    // Agent would stop here. Check for deferred follow-up messages:
    const followUp = await config.getFollowUpMessages();
    if (followUp.length > 0) {
        pendingMessages = followUp;
        continue; // RESTART OUTER LOOP!
    }

    break; // Both queues empty. Agent goes to sleep.
}
```

---

## 🛡️ Phase 4: Why Ordinary LLMs Break & How Agents Prevent Disasters

If you build a naive agent with a simple script, it will fail in 10 minutes. Here is how production agents protect themselves:

### 1. The "Truncated Output" Trap (Token Limits)
* **The Bug:** An LLM generates a tool call to edit a file, but halfway through outputting the code, the model hits its `max_tokens` limit. The JSON is cut in half: `{"file": "app.ts", "code": "function login() { ...`
* **The Disaster:** A naive agent tries to parse the JSON, gets broken syntax, or worse—saves half an empty file, wiping out your project!
* **How Pi Solves It:**
  If `message.stopReason === "length"`, Pi **refuses to execute any tool calls** in that batch. It forcibly marks them as failed errors:
  > *"Tool call was aborted: response hit the token limit so arguments were truncated. Re-issue the tool call with smaller chunks."*

### 2. Surgical Edits vs. Full File Rewriting
* **The Problem:** Asking an AI to *"rewrite this 500-line file with my fix"* leads to lost functions, erased comments, and subtle bugs.
* **The Agent Solution:** Pi uses an exact string match algorithm:
  * LLM must provide the **exact existing lines** (`target`).
  * LLM provides the **new lines** (`replacement`).
  * The tool scans disk: if whitespace or lines don't match 100%, the edit is **rejected**, forcing the LLM to re-read the file.

### 3. Context Compaction (The Infinite Conversation Problem)
* Models have finite context windows (e.g. 128k or 200k tokens).
* As you edit files and run terminal commands, the transcript grows massive.
* In `packages/coding-agent/src/core/compaction`, Pi includes an **automatic compaction engine**:
  1. It monitors token count after every turn.
  2. When it crosses a threshold (e.g. 80% of window), it halts the loop.
  3. It calls a background model to summarize the first 50 messages into a compact snapshot.
  4. It replaces the old messages with the summary, reclaiming 70% of the token space seamlessly without forgetting project state!

---

## 🧩 Phase 5: The Complete Agent State Lifecycle

```
[User Types Message]
       │
       ▼
1. INGESTION ──► Load global settings, authenticate API keys, read .agents/rules
       │
       ▼
2. PROMPT COMPOSITION ──► Assemble: Preamble + System Rules + Skills + Context Files + CWD
       │
       ▼
3. OUTER LOOP START
       │
       ▼
4. INNER LOOP (Turn Step)
       ├── Check Steering (User Interruption)
       ├── Call LLM via SSE Stream (pi-ai)
       ├── Parse Tokens & Tool Calls
       ├── Safety Verification (Schema & Arguments)
       ├── Execute Tools (read / write / edit / bash)
       └── Commit Results to Transcript (JSONL on disk)
       │
       ▼
5. IDLE CHECK
       ├── More tool calls needed? ──► Repeat Inner Loop
       ├── Steering message queued? ──► Repeat Inner Loop
       └── Follow-up task queued? ──► Restart Outer Loop
       │
       ▼
6. COMPLETION ──► Persist transcript, emit `agent_end`, release UI lock
```

---

## 🎯 Summary Checklist for an AI Engineer

If you are building or reverse-engineering an AI agent:

| Component | Why it matters |
|---|---|
| **Dual Loop** | Separates immediate tool resolution from high-level task sequencing. |
| **Steering Queue** | Allows humans to interrupt the agent without killing the process. |
| **Follow-Up Queue** | Enables chaining complex multi-step automations safely. |
| **Atomic File Tools** | Never allow blind file overwrites; always use strict string-diff validation. |
| **Differential TUI** | Render token streams without clearing the screen to prevent flickering. |
| **Durable Storage** | Write every turn to disk (`.jsonl`) so sudden crashes lose zero work. |

