# 🔄 The Complete Inner Loop & Outer Loop Workflow

> A step-by-step, mechanical breakdown of how production coding agents (like Pi) coordinate the **Reflex Engine (Inner Loop)** and the **Goal Sequencer (Outer Loop)**.

---

## 🏗️ 1. High-Level Architecture Flowchart

```mermaid
flowchart TD
    Start(["User submits prompt / Session starts"]) --> InitState["Initialize Mutable State (messages[], tools[])"]
    
    %% OUTER LOOP BOUNDARY
    subgraph OuterBoundary ["🔄 THE OUTER LOOP: while (true) — Goal Sequencer"]
        InitState --> CheckFollowUp{"Are there tasks to run?"}
        
        %% INNER LOOP BOUNDARY
        subgraph InnerBoundary ["⚡ THE INNER LOOP: while (hasMoreToolCalls || steering.length > 0) — Atomic Reflexes"]
            CheckFollowUp -->|"Yes"| DrainSteer["1. Poll Steering Queue (Urgent user interrupts)"]
            DrainSteer --> CheckCompaction{"Context > 80% tokens?"}
            CheckCompaction -->|"Yes"| AutoCompact["Run Background Compaction (Summarize old turns)"]
            CheckCompaction -->|"No"| StreamLLM["2. Stream LLM Response (Tokens + Tool Calls)"]
            AutoCompact --> StreamLLM
            
            StreamLLM --> TruncCheck{"Did response hit token limit? (stopReason === 'length')"}
            TruncCheck -->|"Yes: Truncated JSON"| AbortTool["Abort Tool Calls (Protect disk from broken JSON)"]
            TruncCheck -->|"No: Valid response"| InspectCalls{"Did model request tool calls?"}
            
            InspectCalls -->|"Yes: read, edit, bash"| ExecTools["3. Execute Tools (Surgical edits, safe shell)"]
            ExecTools --> AppendResult["4. Append Tool Results to Transcript JSONL"]
            AbortTool --> AppendResult
            AppendResult --> HasMore["Set hasMoreToolCalls = true"]
            HasMore --> DrainSteer
            
            InspectCalls -->|"No tool calls (Model finished talking)"| SettleInner["Set hasMoreToolCalls = false"]
        end
        
        SettleInner --> InnerDone{"Steering Queue empty?"}
        InnerDone -->|"No: user typed mid-turn"| DrainSteer
        InnerDone -->|"Yes"| CheckOuterFollowUp["Poll Follow-Up Queue (Deferred tasks)"]
        
        CheckOuterFollowUp --> HasFollowUp{"Found follow-up tasks?"}
        HasFollowUp -->|"Yes"| ReArmOuter["Move follow-up to pendingMessages & continue Outer Loop"]
        ReArmOuter --> DrainSteer
    end

    HasFollowUp -->|"No: Both queues drained"| EmitEnd["Emit agent_end & persist session.jsonl"]
    EmitEnd --> IdleState(["Agent becomes IDLE (Ready for next prompt)"])

    style OuterBoundary fill:#1a102f,stroke:#c084fc,stroke-width:2px,color:#fff
    style InnerBoundary fill:#0d1b2a,stroke:#38bdf8,stroke-width:2px,color:#fff
    style DrainSteer fill:#d97706,stroke:#f59e0b,color:#fff
    style CheckOuterFollowUp fill:#7e22ce,stroke:#a855f7,color:#fff
```

---

## 🔬 2. Step-by-Step Execution Lifecycle

Let's trace what happens when you give an agent a task:
> *"Fix the authentication bug in `auth.ts`, and when you're done, run the integration test suite."*

Here is the exact lifecycle across both loops:

### 📍 Phase A: Session Ingestion & Setup
1. **User input arrives:**
   * Immediate task: *"Fix the authentication bug in `auth.ts`"* (enters **pendingMessages**).
   * Deferred task: *"Run integration test suite"* (enters **followUpQueue**).
2. **System Prompt Assembled:** Behavioral rules + available tools (`read`, `edit`, `bash`) + project directory context.
3. **The Outer Loop begins:** `while (true)`.

---

### 📍 Phase B: The Inner Loop (Atomic Reflexes)
The inner loop evaluates: `while (hasMoreToolCalls || pendingMessages.length > 0)`.

#### Cycle 1: Inspection
* **Step 1 (Steering Poll):** Checks if the user typed an urgent mid-flight correction. (None found).
* **Step 2 (LLM Stream):** Model receives transcript. It outputs:
  ```json
  "tool_calls": [{ "name": "read", "arguments": { "path": "src/auth.ts" } }]
  ```
* **Step 3 (Tool Execution):** The tool executor reads `auth.ts` line-by-line from disk.
* **Step 4 (Append Result):** The file contents are pushed into `messages[]`.
* **State Check:** `hasMoreToolCalls = true`. The inner loop repeats immediately!

#### Cycle 2: The Surgical Fix
* **Step 1 (Steering Poll):** No interruption.
* **Step 2 (LLM Stream):** Model reads the file contents from context and outputs:
  ```json
  "tool_calls": [{ "name": "edit", "arguments": { "path": "src/auth.ts", "target": "token.exp < now", "replacement": "token.exp <= now" } }]
  ```
* **Step 3 (Tool Execution):** Pi verifies the exact target text on disk, performs the atomic replacement, and saves the file.
* **Step 4 (Append Result):** Pushes `"Edit applied successfully"` to `messages[]`.
* **State Check:** `hasMoreToolCalls = true`. The inner loop repeats!

#### Cycle 3: The Completion Response
* **Step 1 (Steering Poll):** No interruption.
* **Step 2 (LLM Stream):** The model sees the edit succeeded. It outputs plain text:
  > *"I have fixed the token expiration comparison in `auth.ts`."*
* **Step 3 (Tool Inspection):** No tool calls requested!
* **State Check:** `hasMoreToolCalls = false` AND `pendingMessages.length === 0`.
* **Exit:** **The Inner Loop terminates.**

---

### 📍 Phase C: The Outer Loop Handoff
Now that the inner loop has put its tools down:

1. The outer loop wakes up.
2. It polls the **Follow-Up Queue**:
   * It finds the queued task: *"Run integration test suite"*.
3. The outer loop moves that task into `pendingMessages`.
4. It calls `continue` on the outer loop!
5. **The Inner Loop boots up again:**
   * It calls the model with *"Run integration test suite"*.
   * The model invokes `bash("npm test")`.
   * Tool runs tests ➔ outputs results ➔ model summarizes pass/fail.

---

### 📍 Phase D: Quiescence (Going to Sleep)
1. The inner loop finishes the test summary.
2. The outer loop polls the **Follow-Up Queue** again ➔ **Empty!**
3. Both loops terminate.
4. Pi flushes the final `.jsonl` transcript to disk and returns terminal control to you.

---

## ⚡ 3. The Human Interruption: How `steer()` Intercepts

What if you notice the model reading the wrong file in Cycle 1?

```mermaid
sequenceDiagram
    autonumber
    actor User as 👤 Developer
    participant Outer as 🔄 Outer Loop
    participant Inner as ⚡ Inner Loop
    participant Tool as 🛠️ Tool Executor
    participant LLM as 🤖 AI Model

    Outer->>Inner: Start Turn with Prompt
    Inner->>LLM: Stream request
    LLM-->>Inner: Tool call: read("legacy_auth.ts")
    Inner->>Tool: Execute read("legacy_auth.ts")
    
    Note over User: You notice the wrong file on your terminal screen!
    User->>Inner: steer("Stop! Use src/v2/auth.ts instead")
    
    Tool-->>Inner: File read finishes (settles cleanly)
    Note over Inner: Inner loop drains Steering Queue BEFORE next model call
    Inner->>LLM: Injects: "User steering: Stop! Use src/v2/auth.ts instead"
    LLM-->>Inner: "Understood. Switching to src/v2/auth.ts..."
```

---

## 📊 4. Comparison Summary: Inner vs. Outer Loop

| Feature | The Inner Loop | The Outer Loop |
|---|---|---|
| **Loop Condition** | `while (hasMoreToolCalls \|\| pendingMessages.length > 0)` | `while (true)` |
| **Responsibility** | Immediate atomic reflexes (read, edit, execute) | Long-horizon project and follow-up sequencing |
| **Queue Checked** | **Steering Queue** (`steer()`) — Immediate priority | **Follow-Up Queue** (`followUp()`) — Deferred |
| **When it runs** | Between individual tool calls and model responses | Only after all tool calls settle and the model stops |
| **Failure Handling** | Catches broken tools, token truncations, and syntax errors | Decides whether to continue multi-step jobs or halt |
| **Real-Life Analogy** | The **Carpenter** actively cutting wood at the bench | The **Foreman** holding the job site checklist |
