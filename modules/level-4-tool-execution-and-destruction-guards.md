# 📖 Level 4 Module: Tool Execution & Destruction Guards

> **Core Objective:** Understand why LLMs corrupt files when given raw write access, and how surgical edits, truncation aborts, and schema preflights protect codebases.

---

## 1. The Full-Rewrite Fallacy 💥

Early coding agents used a simple tool: `write_file(path, new_full_content)`.

When you ask an AI model to update one line in an 800-line file:
* The model must stream all 800 lines back over the network.
* **Failure Modes:**
  1. It gets lazy and inserts `// ... rest of code unchanged ...`, wiping out 400 lines of your project.
  2. It hallucinates slight changes in other functions.
  3. It misses curly braces or closing brackets, introducing silent syntax errors.

---

## 2. The Solution: Surgical String Replacement (The Scalpel) ✂️

Pi's [`edit.ts`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/coding-agent/src/core/tools/edit.ts) replaces the sledgehammer with a scalpel:

```json
{
  "path": "src/auth.ts",
  "target": "const timeout = 1000;",
  "replacement": "const timeout = 5000;"
}
```

### The Strict Verification Rule:
1. The tool reads the file currently on disk.
2. It looks for `target`.
3. If `target` is found **exactly once**, it replaces it.
4. If `target` appears multiple times, or is not found (even a 1-space indentation mismatch), the tool **rejects the edit**:
   > *"Error: Target snippet not found or matches multiple locations. Re-read the file with line numbers."*

This makes accidental file corruption mathematically near-impossible.

---

## 3. The Truncation Abort Guard 🛡️

What happens if an LLM is outputting a massive JSON tool call, but runs out of output tokens halfway through?

```json
{
  "name": "edit",
  "arguments": {
    "path": "server.ts",
    "target": "app.listen(3000)",
    "replacement": "app.listen(process.env.P
```
*(Token limit reached. Output stream cuts off here.)*

### The Disaster in Naive Agents:
A naive parser tries to "auto-complete" or parse the broken JSON, and writes corrupted garbage into your production file.

### How Pi Protects You ([`agent-loop.ts#L488`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent-loop.ts#L488)):

```typescript
if (message.stopReason === "length") {
    // The response was cut off by output token limit!
    // NONE of the tool calls in this batch are safe to execute.
    return await failToolCallsFromTruncatedMessage(toolCalls, emit);
}
```

Pi forcibly intercepts the tool call, refuses to execute it on disk, and responds to the model:
> *"Tool call was aborted: the model response hit the output token limit so arguments were truncated. Re-issue the call in smaller chunks."*

