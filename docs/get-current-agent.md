# getCurrentAgent()

## Automatic Context for Custom Methods

**All custom methods automatically have full agent context!** The framework automatically detects and wraps your custom methods during initialization, ensuring `getCurrentAgent()` works seamlessly everywhere.

## How It Works

```typescript
import { AIChatAgent } from "@cloudflare/ai-chat";
import { getCurrentAgent } from "agents";

export class MyAgent extends AIChatAgent {
  async customMethod() {
    const { agent } = getCurrentAgent<MyAgent>();
    // ✅ agent is automatically available!
    console.log(agent.name);
  }

  async anotherMethod() {
    // ✅ This works too - no setup needed!
    const { agent } = getCurrentAgent<MyAgent>();
    return agent.state;
  }
}
```

**Zero configuration required!** The framework automatically:

1. Scans your agent class for custom methods
2. Wraps them with agent context during initialization
3. Ensures `getCurrentAgent()` works in all external functions called from your methods

## Real-World Example

```typescript
import { AIChatAgent } from "@cloudflare/ai-chat";
import { getCurrentAgent } from "agents";
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

// External utility function that needs agent context
async function processWithAI(prompt: string) {
  const { agent } = getCurrentAgent<MyAgent>();
  // ✅ External functions can access the current agent!

  return await generateText({
    model: openai("gpt-4"),
    prompt: `Agent ${agent?.name}: ${prompt}`
  });
}

export class MyAgent extends AIChatAgent {
  async customMethod(message: string) {
    // Use this.* to access agent properties directly
    console.log("Agent name:", this.name);
    console.log("Agent state:", this.state);

    // External functions automatically work!
    const result = await processWithAI(message);
    return result.text;
  }
}
```

### Built-in vs Custom Methods

- **Built-in methods** (onRequest, onEmail, onStateChanged): Already have context
- **Custom methods** (your methods): Automatically wrapped during initialization
- **External functions**: Access context through `getCurrentAgent()`

### The Context Flow

```typescript
// When you call a custom method:
agent.customMethod()
  → automatically wrapped with agentContext.run()
  → your method executes with full context
  → external functions can use getCurrentAgent()
```

## Common Use Cases

### Working with AI SDK Tools

```typescript
import { AIChatAgent } from "@cloudflare/ai-chat";
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

export class MyAgent extends AIChatAgent {
  async generateResponse(prompt: string) {
    // AI SDK tools automatically work
    const response = await generateText({
      model: openai("gpt-4"),
      prompt,
      tools: {
        // Tools that use getCurrentAgent() work perfectly
      }
    });

    return response.text;
  }
}
```

### Calling External Libraries

```typescript
import { AIChatAgent } from "@cloudflare/ai-chat";
import { getCurrentAgent } from "agents";

async function saveToDatabase(data: any) {
  const { agent } = getCurrentAgent<MyAgent>();
  // Can access agent info for logging, context, etc.
  console.log(`Saving data for agent: ${agent?.name}`);
}

export class MyAgent extends AIChatAgent {
  async processData(data: any) {
    // External functions automatically have context
    await saveToDatabase(data);
  }
}
```

## When Context Is Lost: `runWithAgentContext()`

The agent context is carried by `AsyncLocalStorage`, which only propagates
along the call tree of the original invocation. Code reached **outside** that
call tree starts with an empty context, so `getCurrentAgent()` returns
`undefined` there. Common cases:

- A host-side callback invoked via **RPC from a Worker Loader child isolate**
  (e.g. sandboxed tool execution / codemode)
- Calls arriving over a **service binding** or **Durable Object RPC**
- **Queue consumers** and other entrypoints that hold an agent reference

If such code only calls public methods on your agent (`agent.someMethod()`),
nothing is needed — those are wrapped automatically. For arbitrary closures
that rely on `getCurrentAgent()` internally, re-enter the context explicitly:

```typescript
import { runWithAgentContext } from "agents";
import { RpcTarget } from "cloudflare:workers";

class HostCallbackBridge extends RpcTarget {
  constructor(private agent: MyMcpAgent) {
    super();
  }

  // Invoked via RPC from a Worker Loader child isolate — no ALS ancestry,
  // so the agent context must be re-entered before using context-dependent
  // APIs like getCurrentAgent().
  async invoke() {
    return runWithAgentContext(this.agent, async () => {
      const { agent } = getCurrentAgent<MyMcpAgent>();
      // ✅ agent is available again
    });
  }
}
```

Context entered this way has `connection`, `request`, and `email` unset: it is
not tied to any live client I/O. If the given agent is already the current
agent, the function runs unchanged.

> Note: server-initiated MCP requests (`elicitInput`, `createMessage`,
> `listRoots`) on `McpAgent` do **not** require this wrapping — the MCP
> transport resolves its agent independently of the calling context.

## API Reference

The agents package exports two functions for context management:

### `getCurrentAgent<T>()`

Gets the current agent from any context where it's available.

**Returns:**

```typescript
{
  agent: T | undefined,
  connection: Connection | undefined,
  request: Request | undefined
}
```

**Usage:**

```typescript
import { AIChatAgent } from "@cloudflare/ai-chat";
import { getCurrentAgent } from "agents";

export class MyAgent extends AIChatAgent {
  async customMethod() {
    const { agent, connection, request } = getCurrentAgent<MyAgent>();
    // agent is properly typed as MyAgent
    // connection and request available if called from a request handler
  }
}
```

### `runWithAgentContext<T>(agent, fn)`

Runs `fn` inside the given agent's context so `getCurrentAgent()` resolves the
agent. Use it in callbacks reached outside the original invocation's call tree
(cross-isolate RPC, service bindings, queue consumers — see above).

**Parameters:**

- `agent` — the agent instance to run within
- `fn` — the function to execute; its return value is returned

**Usage:**

```typescript
import { getCurrentAgent, runWithAgentContext } from "agents";

const result = runWithAgentContext(agent, () => {
  const { agent: current } = getCurrentAgent();
  // current === agent
  return doSomethingContextDependent();
});
```
