# 📖 Level 3 Module: Priority Interrupts & Steering

> **Core Objective:** Understand how production agents allow human interruption without corrupting the state machine or killing the process.

---

## 1. The Interruption Dilemma 🛑

In naive agents, when an agent begins executing a long task (e.g. running an extensive build or reading 40 files), the terminal is **locked**.

If you notice the agent going down the wrong rabbit hole, your only option is pressing `Ctrl+C`.
* **The Consequence:** The Node/Python process abruptly dies.
* Transcripts are half-written.
* Temporary files are left on disk.
* The conversation context is lost.

---

## 2. The Solution: Dual Priority Queues 🚦

Pi resolves this by introducing two distinct message queues inside [`packages/agent/src/agent.ts`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent.ts#L149-L180):

| Queue | Method | Drain Point | Real-World Analogy |
|---|---|---|---|
| **Steering Queue** | `steer(message)` | **Inside the Inner Loop**, immediately before the next model call. | Tapping the driver on the shoulder: *"Take the next exit instead!"* |
| **Follow-Up Queue** | `followUp(message)` | **Inside the Outer Loop**, only after all tool calls settle. | Leaving a note on the desk: *"When you finish this, run the tests."* |

---

## 3. How Steering Works Under the Hood ⚙️

```mermaid
sequenceDiagram
    autonumber
    actor User as 👤 You
    participant Loop as ⚡ Inner Loop
    participant LLM as 🤖 Model

    Loop->>LLM: Stream turn (Model calls read("wrong_file.ts"))
    Note over User,Loop: You see the tool call on screen and type: "Stop! Don't read that, read config.ts"
    User->>Loop: steer("Read config.ts instead")
    Note over Loop: Tool finishes executing.
    Loop->>Loop: Poll Steering Queue -> Found 1 message!
    Loop->>LLM: Injects user steering message BEFORE next assistant generation
    LLM-->>Loop: "Understood. Switching to config.ts..."
```

Notice that the tool execution was allowed to settle cleanly, but before the model could take another wrong action, the user's message was injected directly in front of it.

---

## 4. Source Code Blueprint: `PendingMessageQueue` 💻

```typescript
class PendingMessageQueue {
    private messages: AgentMessage[] = [];
    public mode: QueueMode; // "all" | "one-at-a-time"

    constructor(mode: QueueMode) {
        this.mode = mode;
    }

    enqueue(message: AgentMessage): void {
        this.messages.push(message);
    }

    drain(): AgentMessage[] {
        if (this.mode === "all") {
            const all = this.messages.slice();
            this.messages = [];
            return all;
        }
        const first = this.messages.shift();
        return first ? [first] : [];
    }
}
```

* **`one-at-a-time` (Default):** Injects only the oldest queued message per turn so the agent can react incrementally.
* **`all`:** Drains every pending message at once into the prompt.

