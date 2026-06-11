---
"agents": patch
---

Fix `Agent was not found in send` when an `McpAgent` issues server-initiated MCP requests (`elicitInput`, `createMessage`, `listRoots`) from code with no agent context on its call stack — e.g. a host-side callback invoked via RPC from a Worker Loader child isolate (sandboxed tool execution / codemode), a service binding, or a queue consumer. `StreamableHTTPServerTransport` now captures its owning agent at construction instead of recovering it from `AsyncLocalStorage` at send time, so server-initiated sends work regardless of how the calling code was reached. (#1490)

Also adds a stable public `runWithAgentContext(agent, fn)` export that re-enters the given agent's context so `getCurrentAgent()` works inside arbitrary callbacks reached outside the original invocation's call tree. This replaces the unsupported workaround of importing `__DO_NOT_USE_WILL_BREAK__agentContext` directly. Context entered this way carries no `connection`, `request`, or `email` — it is not tied to any live client I/O.
