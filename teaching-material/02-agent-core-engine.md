# 🧠 Module 2: The Core Engine (The Brain & The Loop)

Now let's zoom in on the heart of Pi: **`@earendil-works/pi-agent-core`**.

If Pi were a human worker, this is the brain and nervous system. It decides what happens next, remembers what was done, and knows how to use tools.

---

## 1. The Core Loop: The "Think - Act - Observe" Cycle 🔄

Whenever you send a prompt, Pi doesn't just give one answer and stop. It enters a loop:

```mermaid
sequenceDiagram
    autonumber
    actor User as 👤 You
    participant Loop as 🔄 Agent Loop (`agent-loop.ts`)
    participant LLM as 🤖 AI Model (pi-ai)
    participant Tool as 🛠️ Tool Executor (read/bash/edit)

    User->>Loop: "Add a login button in index.html"
    rect rgb(240, 245, 255)
        Note over Loop,LLM: Step 1: THINK
        Loop->>LLM: Here is the history + prompt + available tools
        LLM-->>Loop: "I should inspect index.html first. Calling `read(index.html)`"
    end
    rect rgb(255, 245, 240)
        Note over Loop,Tool: Step 2: ACT
        Loop->>Tool: Execute read("index.html")
        Tool-->>Loop: Contents of index.html...
    end
    rect rgb(245, 255, 245)
        Note over Loop,LLM: Step 3: OBSERVE & REPEAT
        Loop->>LLM: Tool Result: "index.html has <html>...</html>"
        LLM-->>Loop: "Now editing file. Calling `edit(index.html)`"
        Loop->>Tool: Execute edit(...)
        Tool-->>Loop: File updated successfully!
        Loop->>LLM: Tool Result: Success!
    end
    LLM-->>Loop: "I have added the button to index.html!"
    Loop->>User: "Done! Your button is ready."
```

---

## 2. Real-Life Analogy: The Detective 🕵️‍♂️

Think of the Agent Loop like a detective solving a case:

1. **The Briefing (User Prompt)**: The client comes in: *"Find out who stole the artifact."*
2. **First Thought (LLM Call)**: The detective thinks: *"I cannot answer yet. I need clues. Let me dust the crime scene for fingerprints."*
3. **Action (Tool Call)**: The detective uses their fingerprint kit (`fingerprint_tool`).
4. **Observation (Tool Result)**: Fingerprint belongs to person X.
5. **Next Thought (LLM Call)**: *"Now let me check the city database for person X's address."*
6. **Action (Tool Call)**: Searches database (`database_tool`).
7. **Resolution (Final Answer)**: *"Found him at 42 Baker Street."*

The detective **never guesses** if they can verify with tools.

---

## 3. Tool Execution: Sequential vs. Parallel ⚡

When an AI model says *"I want to read 3 files at once"*, how does Pi handle it?

In [types.ts](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/types.ts#L47), Pi defines two modes:
- **`parallel` (Default)**: Preflights all tool calls sequentially for safety, then runs independent operations concurrently via `Promise.all()`. Fast!
- **`sequential`**: Runs tool 1 → waits for finish → runs tool 2 → waits. Safer for destructive file edits.

> [!NOTE]
> Even in parallel mode, if an individual tool specifies `executionMode: "sequential"` (e.g. bash commands or terminal scripts), Pi automatically steps down to sequential execution so steps don't collide.

---

## 4. Steering & Follow-ups: The Steering Wheel 🚗

What if the agent is in the middle of a 10-step task, and you notice it's going the wrong way?

Pi has two special priority queues in [agent.ts](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent.ts#L149):

| Queue | What it does | Real-life Analogy |
|---|---|---|
| **`steer()`** | Injects an instruction right into the current turn before the next AI request. | Tapping the driver on the shoulder: *"Take the next left instead!"* |
| **`followUp()`** | Queues an instruction that waits until the agent has completely finished its current task. | Leaving a sticky note: *"When you finish cleaning the dishes, take out the trash."* |

---

## 5. Lifecycle Events: How the UI knows what's happening

The agent emits events at every stage so the terminal UI (`pi-tui`) or web UI can render spinners, streaming tokens, or diffs:

```
agent_start
  └─► turn_start
        ├─► message_start
        ├─► message_update (streaming text/thinking chunks)
        ├─► message_end
        ├─► tool_execution_start (tool name, args)
        ├─► tool_execution_update (progress)
        ├─► tool_execution_end (result, duration)
        └─► turn_end
  └─► agent_end
```

Every single spinner or colored box you see in your terminal is listening to one of these events!
