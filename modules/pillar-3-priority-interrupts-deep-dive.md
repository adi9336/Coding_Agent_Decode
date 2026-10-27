# 🛑 Pillar 3 Deep Dive: Priority Interrupts & Steering

> **Core Theme:** Why naive agents trap the user in locked terminals, how production harnesses make agents **interruptible without crashing**, and the mathematics of the **Steering vs. Follow-Up Queues**.

---

## 1. The Human-Agent Dilemma 🚧

Imagine you are in a self-driving taxi.
You tell the car: *"Take me to the airport."*

Two minutes into the trip, you look out the window and notice the car is heading towards the seaport instead.
* **In a Naive System:** The car doors are locked. The steering wheel cannot be touched while in motion. Your only option to stop the car is **pulling the emergency handbrake at 60 mph** (hitting `Ctrl+C` in your terminal).
  * The engine dies.
  * The tires screech.
  * You are stranded on the side of the highway, and you have to start the trip completely over from scratch.

* **In a Production Harness:** You gently tap the driver on the shoulder: *"Hey, take exit 4 instead."*
  * The car doesn't crash.
  * It finishes its current lane change safely.
  * It smoothly recalculates the route and takes exit 4.

That is the difference between a toy agent and a **production agent harness**.

---

## 2. Why Simple Agents Are Unusable in Real Work

In 99% of tutorial code or early open-source agents:

```python
# The Naive Flaw
def run_agent(prompt):
    while True:
        response = call_llm(history)
        if not response.tool_calls:
            break
        # Terminal is completely blocked here!
        # The user CANNOT type anything.
        execute_tool(response.tool_calls) 
```

### What happens when things go wrong?
1. The agent enters a loop trying to inspect a 2-gigabyte database file.
2. The user sees this on their screen and screams: *"No, don't read that!"*
3. The user presses `Ctrl+C`.
4. **The entire Node/Python process abruptly terminates.**
5. The session is broken, half-written temp files litter the project, and all previous context is lost.

---

## 3. The Dual-Queue Architecture: `Steer` vs. `Follow-Up`

A real agent harness separates user input into **two distinct priority lanes**:

```
                       [User Types in Terminal]
                                   │
               Does the user want action NOW or LATER?
                                   │
            ┌──────────────────────┴──────────────────────┐
            │                                             │
      [Urgent Interrupt]                           [Deferred Task]
            │                                             │
            ▼                                             ▼
   ⚡ STEERING QUEUE                             📋 FOLLOW-UP QUEUE
   (Drained inside Inner Loop)                  (Drained inside Outer Loop)
   "Interrupt between tool calls"               "Execute only after current job ends"
```

---

## 4. Lane 1: The Steering Queue (`steer()`) ⚡

### When is it used?
When you want to redirect the agent **while it is actively working**.

### Concrete Example:
* You tell the agent: *"Refactor `user.service.ts`."*
* The agent starts up, calls the model, and the model outputs: `tool_call: read("legacy_user.service.ts")`.
* You see this in your terminal. You immediately type:
  > *"Stop. Don't touch legacy files, only look at the new `src/v2/` folder."*

### How the Harness executes this without crashing:

```mermaid
sequenceDiagram
    autonumber
    actor User as 👤 You
    participant Loop as ⚡ Inner Loop Engine
    participant Tool as 🛠️ File System Tool
    participant LLM as 🤖 AI Model

    Loop->>LLM: Stream request
    LLM-->>Loop: Tool call: read("legacy_user.service.ts")
    Loop->>Tool: Begin reading file...
    
    Note over User,Loop: You type in terminal: "Don't touch legacy, use src/v2!"
    User->>Loop: steer("Don't touch legacy, use src/v2!")
    
    Tool-->>Loop: File read finishes (allowed to settle cleanly)
    
    rect rgb(30, 41, 59)
        Note over Loop: 🚨 STEERING DRAIN POINT
        Loop->>Loop: Poll Steering Queue -> 1 message found!
        Loop->>LLM: Injects user steering message BEFORE next model call
    end
    
    LLM-->>Loop: "Understood! Switching to src/v2/user.service.ts..."
```

> [!IMPORTANT]
> **Key Engineering Insight:**
> The harness does **NOT** kill the active tool execution midway (which could corrupt files or leave hanging network sockets). It allows the currently running tool to settle cleanly, but **before the LLM can generate its next decision**, the steering message is injected right in front of it!

---

## 5. Lane 2: The Follow-Up Queue (`followUp()`) 📋

### When is it used?
When you have a task that must run **only after the agent completely finishes its current focus**.

### Concrete Example:
* You tell the agent: *"Implement the Stripe webhook endpoint."*
* While it is writing the endpoint, you remember something else:
  > *"When you finish writing that endpoint, run the full test suite and verify coverage."*

If this message were injected as a steering message, the agent might stop writing the endpoint immediately and try to run tests on unfinished code!

### How the Harness handles it:
1. The message is pushed to `followUpQueue`.
2. The **Inner Loop** completely ignores it. The agent finishes writing the endpoint, edits files, and settles.
3. The model finishes talking: `hasMoreToolCalls = false`.
4. The **Outer Loop** wakes up, drains the `followUpQueue`, and boots up the inner loop for the next mission!

---

## 6. The Code Ground Truth in Pi Agent

Let's look at the exact code from [`packages/agent/src/agent.ts`](./pi/packages/agent/src/agent.ts#L149-L180):

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

    peek(): AgentMessage[] {
        if (this.mode === "all") return this.messages.slice();
        const first = this.messages[0];
        return first ? [first] : [];
    }

    drain(): AgentMessage[] {
        const drained = this.peek();
        this.messages = this.messages.slice(drained.length);
        return drained;
    }
}
```

And how the loop consumes it inside [`agent-loop.ts`](./pi/packages/agent/src/agent-loop.ts#L185):

```typescript
// INNER LOOP: Polled on EVERY turn before streaming to LLM
let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) || [];

// ... [Tool execution happens] ...

// OUTER LOOP: Polled ONLY when inner loop completely stops
const followUpMessages = (await config.getFollowUpMessages?.()) || [];
if (followUpMessages.length > 0) {
    pendingMessages = followUpMessages;
    continue; // Restarts the loop for the next major task!
}
```

---

## 7. The Queue Drain Modes: `one-at-a-time` vs. `all`

Notice the `QueueMode` in the code above? Why does that exist?

* **`one-at-a-time` (Default):**
  If you typed 3 messages while waiting, the agent consumes **only the oldest one first**. It gives the AI a chance to react to that instruction before dumping the rest on it.
* **`all`:**
  Drains all queued messages into a single prompt block immediately.

---

## 🧠 Summary: Why Pillar 3 Matters

Without priority interrupt queues:
1. **Agents run away:** You cannot correct bad decisions without terminating the session.
2. **Pipelines break:** You cannot queue sequential goals cleanly.
3. **Developers lose trust:** An agent that locks your keyboard feels like a black box you cannot control.

With Pillar 3, the agent becomes a **collaborative partner**: fast and autonomous, but always responsive to your steering wheel.
