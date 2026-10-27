# 📖 Level 1 Module: The Stateless Brain & The Runtime Illusion

> **Core Objective:** Understand why LLMs have zero memory, how agent harnesses create the illusion of continuity, and how transcript state is managed in production.

---

## 1. The Fundamental Reality: Pure Functions ⚡

An LLM (Large Language Model like Claude, GPT-4, or Gemini) is fundamentally a **stateless mathematical function**:

$$\text{Output} = f(\text{Input})$$

* It does **not** store your previous messages on its servers.
* It does **not** have an ongoing "thought process" while waiting for you to type.
* When an HTTP request completes, all runtime activations inside the model are completely wiped from GPU memory.

---

## 2. The Film Strip Analogy 🎞️

Think of an analog movie projector:

```
[Frame 1: Car starts] ──► [Frame 2: Car accelerates] ──► [Frame 3: Car crashes]
```

1. The **Projector Light Bulb** (= The LLM) has no memory. It simply illuminates whatever film frame is placed in front of it right now.
2. The **Film Strip** (= The Transcript Array) is an ever-growing physical strip of pictures.
3. The **Projectionist** (= The Agent Harness / Pi) cuts and splices new frames onto the end of the film strip before shining the light through it again.

The viewer sees smooth, continuous animation. In reality, it is a sequence of discrete, static snapshots fed one after the other.

---

## 3. The Network Wire: What Actually Happens Turn by Turn 📡

Let us trace the exact payloads that travel over the wire during a coding session:

### Turn 1: Initial Prompt
* **Sent to Model:**
  ```json
  [
    { "role": "system", "content": "You are a coding assistant with access to tools: read, edit, bash." },
    { "role": "user", "content": "Fix the bug in src/auth.ts" }
  ]
  ```
* **Model Output:**
  ```json
  {
    "role": "assistant",
    "content": "I need to inspect the file first.",
    "tool_calls": [{ "name": "read", "arguments": { "path": "src/auth.ts" } }]
  }
  ```

---

### Turn 2: Feeding the Tool Result Back
The model has forgotten Turn 1. The agent runtime executes `read("src/auth.ts")` on local disk, gets the contents, and constructs a new request:

* **Sent to Model:**
  ```json
  [
    { "role": "system", "content": "You are a coding assistant..." },
    { "role": "user", "content": "Fix the bug in src/auth.ts" },
    { "role": "assistant", "content": "I need to inspect the file first.", "tool_calls": [...] },
    { "role": "tool_result", "content": "export function login() { return false; }" }
  ]
  ```
* **Model Output:**
  ```json
  {
    "role": "assistant",
    "content": "Found the issue. Applying the fix now.",
    "tool_calls": [{ "name": "edit", "arguments": { "path": "src/auth.ts", "target": "return false;", "replacement": "return true;" } }]
  }
  ```

---

## 4. Production Ground Truth: Pi Codebase 💻

In [`packages/agent/src/agent.ts`](./pi/packages/agent/src/agent.ts#L83-L106), we see how this memory array is instantiated in production:

```typescript
function createMutableAgentState(initialState?: AgentInitialState): MutableAgentState {
    let tools = initialState?.tools?.slice() ?? [];
    let messages = initialState?.messages?.slice() ?? []; // <--- The Entire Working Memory!
    
    return {
        get systemPrompt() { return getCurrentSystemPrompt(messages); },
        model: initialState?.model ?? DEFAULT_MODEL,
        get tools() { return tools; },
        set tools(nextTools) { tools = nextTools.slice(); },
        get messages() { return messages; },
        set messages(nextMessages) { messages = nextMessages.slice(); },
        isStreaming: false,
        pendingToolCalls: new Set<string>()
    };
}
```

Every turn simply appends to `this._state.messages`.

---

## 5. Architectural Consequences ⚠️

Because memory is simulated by resending the entire transcript every time:

1. **Quadratic Cost Growth ($O(N^2)$ Tokens):**
   * Turn 1 sends 100 tokens.
   * Turn 2 sends Turn 1 + Turn 2 (300 tokens).
   * Turn 10 sends all 10 turns combined (15,000 tokens).
2. **Latency Inflation:** As transcripts grow, Time-To-First-Token (TTFT) slows down because the model must process longer prompt prefixes.
3. **The Hard Context Ceiling:** Eventually, `messages[]` exceeds the context window (e.g., 200,000 tokens). This necessitates **Context Compaction** (covered in Level 5).

