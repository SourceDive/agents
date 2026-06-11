---
"agents": patch
---

Fix `Agent was not found in send` when an `McpAgent` issues server-initiated MCP requests (`elicitInput`, `createMessage`, `listRoots`) from code with no agent context on its call stack — e.g. a host-side callback invoked via RPC from a Worker Loader child isolate (sandboxed tool execution / codemode), a service binding, or a queue consumer. `StreamableHTTPServerTransport` now captures its owning agent at construction instead of recovering it from `AsyncLocalStorage` at send time, so server-initiated sends work regardless of how the calling code was reached. (#1490)

For other context-dependent APIs in such callbacks, route the callback through a public method on your agent class — custom methods are automatically wrapped and re-enter the agent's context (see `docs/get-current-agent.md`).
