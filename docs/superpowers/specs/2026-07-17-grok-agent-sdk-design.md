# grok-agent-sdk: a harness SDK for Grok Build

Status: proposed
Repo for implementation: separate, new (`grok-agent-sdk`, TypeScript)
This spec lives in `grok-build` because the SDK's contract is defined entirely
by `grok-build`'s existing public interfaces (ACP mode, extension methods),
even though the SDK code itself will not live here.

## 1. Problem

`grok-build` ships two ways to drive the agent programmatically today:

- **Headless mode** (`grok -p`): spawn-per-prompt, one-directional output
  (`plain`/`json`/`streaming-json`), documented in
  `crates/codegen/xai-grok-pager/docs/user-guide/14-headless-mode.md`. Good
  for CI/scripting. No persistent process, no interactive permission
  callback, no custom in-process tools, no live hook interception.
- **Agent mode / ACP** (`grok agent stdio`): a persistent process speaking
  the [Agent Client Protocol](https://agentclientprotocol.com), documented in
  `crates/codegen/xai-grok-pager/docs/user-guide/15-agent-mode.md`. Bidirectional
  JSON-RPC: session lifecycle, streamed `session/update` events (message
  chunks, thought chunks, tool calls, plans), interactive permission
  requests, and `x.ai/*` extension methods for fs/git/terminal/search/session
  control. Generic third-party ACP client SDKs exist (TS, Rust, Python, Go,
  Kotlin), but they only give you raw protocol plumbing — the same level of
  ergonomics as hand-rolling JSON-RPC.

Neither is a **harness SDK** in the sense of the Claude Agent SDK: a
language-native package that lets an application embed the coding agent —
spawn it, stream its turns, intercept and approve/deny tool calls
programmatically, register custom tools without a separate server process,
and resume/fork sessions — without the caller hand-rolling JSON-RPC framing
or subprocess management. This spec defines that layer.

This is explicitly **not** an API client SDK (a wrapper over a chat-completion
endpoint). Every operation described here goes through the real agent
process — the same binary and tool-execution runtime the TUI uses — not a
raw model API.

## 2. Decision: build on ACP, not headless

Headless mode cannot support the feature set this SDK needs to expose,
structurally:

| Capability                         | Headless (`grok -p`) | ACP (`grok agent stdio`) |
| ----------------------------------- | --------------------- | ------------------------- |
| Persistent process across turns     | No (spawn per call, unless externally re-invoked with `-r`) | Yes |
| Bidirectional control               | No (stdout stream only) | Yes (JSON-RPC both directions) |
| Interactive permission callback     | No (static `--allow`/`--deny`/`--yolo` only) | Yes (`session/request_permission`) |
| Live tool-call visibility (pre/post)| Only via final JSON/event log | Yes (`tool_call`/`tool_call_update` stream events) |
| Custom tools                        | Static MCP server config file only | `mcpServers` in `session/new`, plus dynamic `x.ai/mcp/upsert` |
| Session fork                        | No | Yes (`x.ai/session/fork`) |
| Plan visibility                     | No | Yes (`plan` update) |

This mirrors why Claude Agent SDK wraps Claude Code's own persistent
`stream-json`/`stream-json` control protocol rather than its one-shot `-p`
mode — hooks, hooks, custom tools, and permission callbacks all require a
standing bidirectional channel. For `grok-build`, ACP is that channel and is
already a stable, documented, externally-consumed interface (Zed, Neovim,
Emacs integrate through it), so building on it doesn't require touching or
duplicating harness internals.

Headless mode remains the right tool for CI one-shots and is out of scope
for this SDK; scripts that just want "run a prompt, get text back, exit"
should keep using `grok -p`.

## 3. Non-goals (v1)

- No websocket/remote transport (`grok agent serve`) — stdio child process
  only. Remote transport is a plausible v1.1 addition since ACP already
  supports it uniformly.
- No Python or Rust package — TypeScript only. The design keeps the
  transport/protocol layer isolated so a Python port is mechanical later,
  but it is not built now.
- No changes to `grok-build` itself. The SDK treats the `grok` binary as a
  black box consuming only its already-public ACP surface. If a needed
  capability doesn't exist on that surface, that's a finding to report
  upstream, not a reason to vendor or fork harness source.
- No attempt to reimplement ACP framing — depend on `@agentclientprotocol/sdk`
  for JSON-RPC/session-type plumbing.

## 4. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Caller application                        │
│         import { query, createSdkMcpServer } from            │
│                  "grok-agent-sdk"                             │
└───────────────────────────┬───────────────────────────────────┘
                             │
┌───────────────────────────▼───────────────────────────────────┐
│                      grok-agent-sdk                            │
│  ┌────────────┐ ┌──────────────┐ ┌────────────────────────┐  │
│  │  query()   │ │ canUseTool / │ │ createSdkMcpServer()    │  │
│  │  (turn     │ │ hooks        │ │ (local loopback MCP     │  │
│  │  loop, msg │ │ (permission +│ │  server for custom      │  │
│  │  typing)   │ │  event taps) │ │  tools, auto-registered) │  │
│  └─────┬──────┘ └──────┬───────┘ └───────────┬──────────────┘  │
│        └───────────────┴─────────────────────┘                 │
│                    @agentclientprotocol/sdk                    │
│              (JSON-RPC framing, ACP session types)             │
└───────────────────────────┬───────────────────────────────────┘
                             │ spawn + stdio JSON-RPC
┌───────────────────────────▼───────────────────────────────────┐
│                    `grok agent stdio` (child process)          │
│         Session Manager · Tool Registry · MCP Servers          │
└─────────────────────────────────────────────────────────────┘
```

The SDK is a client only. It spawns `grok agent stdio`, performs
`initialize` → `session/new`, and from then on translates ACP traffic to and
from a typed TypeScript surface.

## 5. Package layout (new repo)

```
grok-agent-sdk/
├── src/
│   ├── query.ts            # query(), interactive handle
│   ├── process.ts           # child_process spawn/lifecycle/restart
│   ├── messages.ts          # SDKMessage union + translation from session/update
│   ├── permissions.ts       # canUseTool wiring to session/request_permission
│   ├── hooks.ts             # hook matcher/dispatch engine
│   ├── mcp/
│   │   ├── sdkServer.ts     # createSdkMcpServer() — local loopback MCP server
│   │   └── tool.ts          # tool() helper (zod schema + handler)
│   ├── session.ts           # resume/fork/rewind wrappers over x.ai/session/*
│   └── extensions/          # typed x.ai/* method clients (fs, git, terminal, search)
├── test/
├── examples/
└── package.json             # npm: "grok-agent-sdk"
```

## 6. Core API

```ts
function query(input: {
  prompt: string | AsyncIterable<UserMessage>;
  options?: {
    cwd?: string;
    model?: string;
    permissionMode?: "default" | "acceptEdits" | "bypassPermissions" | "plan";
    canUseTool?: (call: ToolCallInfo) => Promise<PermissionDecision>;
    hooks?: Partial<Record<HookEvent, HookCallback[]>>;
    mcpServers?: McpServerConfig[];
    resume?: string;        // session id
    forkSession?: boolean;
    maxTurns?: number;
    abortSignal?: AbortSignal;
  };
}): Query;

interface Query extends AsyncIterable<SDKMessage> {
  send(message: string | UserMessage): void;   // multi-turn, same session
  interrupt(): Promise<void>;
  setPermissionMode(mode: PermissionMode): Promise<void>;
  sessionId: string;
}
```

### Message types (`SDKMessage` union)

| Type            | Sourced from ACP                          | Notes |
| --------------- | ------------------------------------------ | ----- |
| `text_delta`    | `session/update` → `agent_message_chunk`   | streamed response text |
| `thought_delta` | `session/update` → `agent_thought_chunk`   | reasoning stream |
| `tool_call`     | `session/update` → `tool_call`             | new invocation (title, kind, input) |
| `tool_call_update` | `session/update` → `tool_call_update`   | status/result transitions |
| `plan`          | `session/update` → `plan`                  | agent's execution plan |
| `result`        | final `session/prompt` response + `_meta.usage` (`PromptUsage`) | stopReason, usage, cost — same fields as headless `end`/json result |
| `error`         | JSON-RPC error / process failure           | includes process-crash and protocol errors, not just model errors |

This is a direct re-typing of the ACP wire shapes already documented in
`15-agent-mode.md` — the SDK does not invent new semantics here, only types
and an ergonomic async-iterable framing around them (mirrors Claude Agent
SDK's `SDKMessage` union).

## 7. Permission callback — the blocking hook

```ts
type PermissionDecision =
  | { behavior: "allow" }
  | { behavior: "allow"; updatedInput: unknown }
  | { behavior: "deny"; message?: string };

canUseTool?: (call: ToolCallInfo) => Promise<PermissionDecision>;
```

Wired directly to ACP's `session/request_permission` (agent → client
request). This is the only point in the protocol that can actually block a
tool call before it runs — functionally identical in role to Claude Agent
SDK's `canUseTool`/blocking `PreToolUse` hook. If `canUseTool` is omitted,
the SDK defaults to ACP's normal permission-request forwarding behavior
(caller must handle it, or pass `permissionMode: "bypassPermissions"` to
auto-approve, matching `--yolo`).

## 8. Hooks — observation, not blocking

```ts
type HookEvent = "PreToolUse" | "PostToolUse" | "UserPromptSubmit" | "SessionStart" | "SessionEnd";
type HookCallback = (event: HookPayload) => void | Promise<void>;
```

- `PreToolUse` fires on the `tool_call` update (status `pending`/`in_progress`).
  It **cannot** block — blocking is `canUseTool`'s job, matching Claude
  Agent SDK's own split (a `PreToolUse` hook there can technically also
  deny, but grok's ACP protocol only exposes blocking through the
  permission-request channel, so this SDK keeps that as the single source
  of truth to avoid two competing "deny" paths racing each other).
- `PostToolUse` fires on `tool_call_update` reaching a terminal status
  (`completed`/`failed`) — purely observational, the tool has already run.
- `UserPromptSubmit`/`SessionStart`/`SessionEnd` are synthesized locally by
  the SDK around `session/prompt` and `session/new`/close, not sourced from
  a dedicated ACP event (ACP has no direct equivalent) — documented as
  SDK-level conveniences, not protocol guarantees.

## 9. Custom tools — `createSdkMcpServer`

```ts
const server = createSdkMcpServer({
  name: "my-tools",
  tools: [
    tool("get_weather", "Get current weather", z.object({ city: z.string() }),
      async ({ city }) => ({ content: [{ type: "text", text: `... ${city} ...` }] })),
  ],
});

query({ prompt, options: { mcpServers: [server] } });
```

**Important caveat, stated explicitly so it isn't oversold:** ACP's
`mcpServers` (checked against `crates/codegen/xai-grok-shell/src/extensions/mcp.rs`
and `agent/mvp_agent/acp_agent.rs`) only accepts real MCP server connections
(stdio/http/sse) — there is no inline-function/direct-dispatch tool
extension in the protocol. `createSdkMcpServer` therefore starts a small
MCP server *inside the SDK's own process* (using `@modelcontextprotocol/sdk`'s
`Server`) bound to a local loopback HTTP/SSE listener, and registers that
listener's URL via `mcpServers` in `session/new` (or `x.ai/mcp/upsert` for a
session already in flight). From the caller's point of view this behaves
like an in-process tool — define a function, agent can call it, no extra
process to manage — but it is implemented as a same-process local server,
not literal in-SDK function dispatch the way Claude Agent SDK's internal
control protocol does it.

## 10. Session control

Thin typed wrappers over already-documented extensions:

| SDK call                 | ACP method                | 
| ------------------------- | -------------------------- |
| `options.resume`          | `session/load`             |
| `options.forkSession`     | `x.ai/session/fork`        |
| `query.interrupt()`       | `session/cancel`           |
| rewind helpers (v1.1)     | `x.ai/rewind/*`            |

## 11. Process lifecycle & error handling

- One child process per `query()` call by default; the interactive handle
  keeps it alive across `.send()` calls in the same session.
- Crash/exit before a `result` message → the async iterable throws a typed
  `AgentProcessError` (exit code, stderr tail) rather than silently ending.
- `abortSignal` → `session/cancel` then SIGTERM the child if it doesn't
  exit within a grace window (mirrors the leader/client eviction pattern
  already used internally in `crates/codegen/xai-grok-shell/src/leader/mod.rs`,
  reimplemented at the SDK level since the SDK does not talk to the leader
  socket directly, only to a plain `grok agent stdio` child).

## 12. Testing strategy

- Unit tests for message translation (`session/update` → `SDKMessage`) against
  fixture JSON-RPC transcripts — no real `grok` binary required.
- Integration tests spawn a real `grok agent stdio` process (gated behind an
  env var / CI job that has a `grok` binary and auth available), covering:
  basic `query()` round trip, `canUseTool` deny path, custom tool round trip
  via `createSdkMcpServer`, session resume, interrupt/cancel.
- No changes to `grok-build`'s own test suite — this SDK's tests live in its
  own repo entirely.

## 13. Risks / open questions

- **ACP protocol version drift**: the SDK must check `initialize`'s
  negotiated protocol version and `leader_capabilities`-style feature flags
  (the doc notes the `x.ai/*` extension set is "non-exhaustive... discover
  from the agent's `initialize` response") — v1 should fail loudly on
  unsupported versions rather than silently degrading.
- **Loopback MCP server security**: the local HTTP/SSE listener for
  `createSdkMcpServer` must bind to `127.0.0.1` with a per-run random token,
  not an open port — needs explicit design in implementation, not just
  "works on localhost."
- **Auth**: the SDK does not manage `grok login`/API keys; it assumes the
  environment is already authenticated (`XAI_API_KEY` or cached
  `~/.grok/auth.json`), same precondition as headless mode.

## 14. Future work (post-v1)

- `grok agent serve` (websocket) transport for remote/browser use cases.
- Python package mirroring the TS API.
- Typed clients for the remaining `x.ai/*` extension families (git,
  terminal, search, worktree) beyond the minimal set needed for v1.
