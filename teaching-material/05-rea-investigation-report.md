# 🔬 REA Automated Investigation Report: Pi Agent Core

We ran **REA (`reverse-engineer-anything`)** directly on the Pi Agent core package using the official pipeline:
`npx -y rea-agents@latest analyze-javascript-application packages/agent --json`

REA performed semantic AST reconstruction across all 18 source files and resolved module binding graphs, entry points, and source facts.

---

## 1. Concrete Evidence Recovered by REA

| REA Module ID | Physical Source Path | Structural Role Recovered |
|---|---|---|
| `jsm:18d356c39...` | `src/agent-loop.ts` | **The Execution Engine**: Hosts `runAgentLoop`, `runLoop`, and `streamAssistantResponse`. |
| `jsm:0287cf840...` | `src/agent.ts` | **The State Owner**: Hosts the `Agent` class, `steeringQueue`, `followUpQueue`, and event dispatcher. |
| `jsm:2e3f52427...` | `src/stream-fn.ts` | **Stream Adapter**: Bridges standard LLM event streams into typed `AssistantMessageEventStream`. |
| `jsm:a5c202027...` | `src/proxy.ts` | **Network Proxy/Interception Layer**: Safe request redirection. |
| `jsm:dc73c38cb...` | `src/types.ts` | **The Type Contract**: Exports `AgentEvent`, `AgentTool`, `BeforeToolCallContext`, `QueueMode`. |

---

## 2. Structural Relationships Recovered by REA

```
[src/agent.ts (State Manager)]
    │
    ├── imports ──► [src/agent-loop.ts (Engine)]
    │                  │
    │                  ├── calls ──► [streamAssistantResponse] (LLM Protocol)
    │                  └── executes ─► [executeToolCalls] (Sandbox Boundary)
    │
    ├── manages ──► [PendingMessageQueue: Steering] (Priority Interruption)
    ├── manages ──► [PendingMessageQueue: FollowUp] (Deferred Automation)
    └── emits ────► [AgentEventSink] (UI Updates)
```

---

## 3. What REA Proved (Ground Truths)

1. **Zero Global Variables**: The agent engine has no hidden global state. Everything lives strictly inside `AgentContext` and `MutableAgentState`.
2. **Strict Protocol Boundary**: The agent converts its internal rich messages (`AgentMessage[]`) into LLM-compatible format (`Message[]`) **only at the absolute last millisecond** before hitting the AI API ([`agent-loop.ts#L404`](file:///C:/Users/adi/CODING_AGENT_DECODE/pi/packages/agent/src/agent-loop.ts#L404)).
3. **Fail-Safe Argument Recovery**: REA identified that when token exhaustion occurs, `failToolCallsFromTruncatedMessage` intercepts the truncated JSON before it ever executes on disk.

