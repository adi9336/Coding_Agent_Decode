# 📖 Level 2 Module: The Dual-Loop State Machine

> **Core Objective:** Master the inner vs. outer loop architecture that powers autonomous agent reflexes and high-level goal sequencing.

---

## 1. Why One Simple Loop Fails ❌

Most simple agent tutorials suggest writing code like this:

```python
# Naive approach: Single loop
while True:
    response = call_llm(history)
    if not response.tool_calls:
        break
    execute_tools(response.tool_calls)
```

### Why this breaks in the real world:
* **No Interruption Surface:** If the agent gets stuck in a loop reading 50 files, you cannot steer it or inject corrections without killing the entire process.
* **No Multi-Phase Automation:** After fixing a bug, the agent cannot automatically proceed to "run integration tests", because the loop exits the moment the tool calls stop.

---

## 2. The Solution: Dual Nested Loops 🔄

Production coding agents (like Pi) decouple immediate **tool reflexes** from **long-term task sequencing**:

```
┌────────────────────────────────────────────────────────────────────────┐
│  🔄 OUTER LOOP: Goal Sequencer (while true)                            │
│  Monitors follow-up queues. Runs across entire multi-step projects.     │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │  ⚡ INNER LOOP: Immediate Reflex Turn (while hasMoreToolCalls)  │   │
│   │                                                                │   │
│   │  1. Check steering messages (Interruption queue)               │   │
│   │  2. Stream LLM tokens & parse tool calls                       │   │
│   │  3. Execute tools (read / edit / bash)                         │   │
│   │  4. Append results to transcript                               │   │
│   │                                                                │   │
│   │  Continues until model stops requesting tools.                 │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │                                    │
│       Inner Loop Finishes ────────┘                                    │
│       Checks: "Are there queued follow-up tasks?"                      │
│       If YES ──► Re-arm and restart Inner Loop!                        │
│       If NO  ──► Settle run and go IDLE.                               │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Real-Life Analogy: The Carpenter & The Foreman 👷‍♂️

| Role | Agent Equivalent | What It Does |
|---|---|---|
| **The Carpenter** | **The Inner Loop** | Working hands-on at the bench: measure, saw, sand. Fast and reactive. |
| **The Foreman** | **The Outer Loop** | Standing back with the blueprint checklist: *"Cabinet frame is done. Next task: attach the hinges."* |

The carpenter does not leave the workshop until the cut is finished. The foreman does not assign the next task until the current one is settled.

---

## 4. Reverse-Engineered Source: `agent-loop.ts` 🔬

Look at the exact lines in [`packages/agent/src/agent-loop.ts`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent-loop.ts#L188-L327):

```typescript
// OUTER LOOP: Keeps running as long as follow-up work exists
while (true) {
    let hasMoreToolCalls = true;

    // INNER LOOP: Resolves atomic tool executions & immediate steering
    while (hasMoreToolCalls || pendingMessages.length > 0) {
        
        // 1. Drain immediate steering messages (user interruption)
        pendingMessages = (await config.getSteeringMessages?.()) || [];

        // 2. Stream assistant response from provider
        const message = await streamAssistantResponse(currentContext, config, signal, emit, streamFunction);
        newMessages.push(message);

        // 3. Inspect for tool calls
        const toolCalls = message.content.filter((c) => c.type === "toolCall");

        if (toolCalls.length > 0) {
            // 4. Execute tool calls and push results
            const executedToolBatch = await executeToolCalls(currentContext, message, config, signal, emit);
            for (const result of executedToolBatch.messages) {
                currentContext.messages.push(result);
                newMessages.push(result);
            }
            hasMoreToolCalls = !executedToolBatch.terminate;
        } else {
            hasMoreToolCalls = false; // No more tools needed this turn
        }
    }

    // Inner loop finished! Agent would stop here.
    // Check for deferred follow-up messages:
    const followUpMessages = (await config.getFollowUpMessages?.()) || [];
    if (followUpMessages.length > 0) {
        pendingMessages = followUpMessages;
        continue; // RESTART OUTER LOOP for the next task!
    }

    break; // Both queues empty. Agent safely sleeps.
}
```

---

## 5. Architectural Checklist for AI Engineers 📋

1. **Inner Loop Termination:** Must terminate when `toolCalls.length === 0` OR when a tool explicitly returns `terminate: true`.
2. **Atomic Context Mutation:** Tool result messages must be appended to context *before* the next inner loop turn begins.
3. **Queue Polling Separation:** Steering messages are polled **inside** the inner loop; Follow-up messages are polled **only between** outer loop cycles.

