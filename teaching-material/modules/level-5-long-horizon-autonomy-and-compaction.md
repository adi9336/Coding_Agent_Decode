# 📖 Level 5 Module: Long-Horizon Autonomy & Context Compaction

> **Core Objective:** Understand how agents manage context token limits over long multi-hour sessions through automated background compaction and branch summarization.

---

## 1. The Finite Memory Ceiling 🧱

Every AI model has a maximum context window:
* Claude 3.5 Sonnet: 200,000 tokens
* GPT-4o: 128,000 tokens

In an active coding session:
* Reading a single large package file or test log can cost 20,000 tokens.
* Within 10 to 15 turns of reading and editing files, the transcript will hit the limit.
* When the limit is hit, the provider API throws an error: `context_window_exceeded`.

---

## 2. Naive Solutions & Why They Fail ❌

| Approach | Why It Fails |
|---|---|
| **Drop oldest messages (FIFO)** | The model forgets what repository it is working in, what the original user request was, and which files it already modified. |
| **Clear history on error** | The user's entire conversation is wiped out. |

---

## 3. The Production Solution: Context Compaction 📦

In [`packages/coding-agent/src/core/compaction`](./pi/packages/coding-agent/src/core/compaction), Pi implements an automated compaction engine:

```
[System Rules] ──► Preserved 100% untouched
[Old Turns 1..15] ──► Summarized into concise 500-token briefing
[Recent Turns 16..20] ──► Preserved word-for-word
```

### The Compaction Flow:
1. **Token Accounting:** After every turn, the engine estimates projected tokens.
2. **Threshold Trigger:** When usage crosses ~80% of the model's capacity, the agent pauses the inner loop before the next turn.
3. **Background Summary Call:** A dedicated summarizer prompt extracts:
   * **Decisions Made:** Architectural choices and accepted trade-offs.
   * **Files Mutated:** Concrete list of files touched and functions changed.
   * **Pending Work:** What is left on the user's checklist.
4. **Atomic Transcript Replacement:** The historical slice is replaced by the summary block.
5. **Reclaimed Context:** Instantly recovers 70%+ of the context window without the user or model losing track of project reality!

