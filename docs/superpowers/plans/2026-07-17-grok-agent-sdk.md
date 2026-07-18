# grok-agent-sdk Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `grok-agent-sdk`, a TypeScript package that gives applications programmatic, harness-level control of Grok Build's agent (spawn, stream turns, approve/deny tool calls, register custom tools, resume/fork sessions) — the same role Claude Agent SDK plays for Claude Code — by wrapping `grok agent stdio` (ACP) via `@agentclientprotocol/sdk`.

**Architecture:** A child process (`grok agent stdio`) is driven through `@agentclientprotocol/sdk`'s `client({name}).onRequest(...).connectWith(stream, ctx => ...)` builder. Inside that callback, `ctx.buildSession(cwd).withSession(session => ...)` drives one ACP session; `session.prompt()`/`session.nextUpdate()` are translated into a typed `SDKMessage` stream and pushed into an internal async queue that the public `query()` async generator reads from — this decouples the ACP session's own callback-scoped lifetime from the caller-facing async-iterable API. `canUseTool` is wired to the ACP `session/request_permission` handler (the only point in the protocol that can block a tool call). Hooks (`PreToolUse`/`PostToolUse`) tap the same message stream for observation. Custom tools run a real `@modelcontextprotocol/sdk` `McpServer` in the SDK's own process, exposed over a loopback-only, bearer-token-protected HTTP transport, and registered with the session via `SessionBuilder.withMcpServer()`. Both `@agentclientprotocol/sdk`'s core and `@modelcontextprotocol/sdk`'s `WebStandardStreamableHTTPServerTransport` are Web-Standard-based and runtime-agnostic by construction; the package isolates the two genuinely runtime-specific concerns — spawning the `grok` subprocess (Task 3) and hosting the loopback MCP server's HTTP listener (Task 11) — behind small, explicit seams rather than letting runtime assumptions leak into the rest of the code.

**Tech Stack:** TypeScript (Node >=22, ESM-only; also runs on Bun, verified against Bun 1.3.14 — see Task 3), `@agentclientprotocol/sdk@^1.2.1`, `@modelcontextprotocol/sdk@^1.29.0`, `zod@^4.0.0`, `vitest@^4`, `pnpm@10.34.5`. Build: plain `tsc` (no bundler — see Global Constraints). Lint/format: Biome (single tool, replaces ESLint+Prettier). Package correctness: `publint` + `@arethetypeswrong/cli`, run in Task 15 once the real public API surface exists.

## Global Constraints

- Repo lives at `~/Documents/Workfolder/grok-agent-sdk`, entirely separate from `grok-build` — no changes to `grok-build` source in this plan.
- Package manager: `pnpm`. Test framework: `vitest`.
- Every ACP method/type name used in code must be one that exists in `@agentclientprotocol/sdk@1.2.1`'s published `.d.ts` (verified during design: `ndJsonStream`, `client()`/`ClientApp`, `ClientContext.buildSession()`, `SessionBuilder`, `ActiveSession`, `ActiveSessionMessage`, `schema.*` types, `methods.client.session.requestPermission`/`.update`, `methods.agent.session.new`/`.prompt`/`.cancel`/`.fork`). Do not invent method names.
- Loopback MCP server (Task 11) must bind to `127.0.0.1` only and require a per-run random bearer token — never an open port with no auth, per the spec's stated risk. It must use `WebStandardStreamableHTTPServerTransport`, not the Node-only `StreamableHTTPServerTransport`/`NodeStreamableHTTPServerTransport` wrapper, so the transport itself stays portable even though this plan only implements the `node:http` hosting bridge.
- Runtime-portability verification status, stated honestly rather than assumed: the Node path (Task 3, Task 11) is exercised by every test in this plan. The Bun path (Task 3) was verified against a real installed Bun 1.3.14 binary during design and has its own passing unit test. The Deno path (Task 3) is written against Deno's stable, documented `Deno.Command` API and, in a later research pass, Deno 2.9.3 was actually installed in the design environment and driven directly: `Deno.Command({stdin:"piped", stdout:"piped"}).spawn()` was confirmed to produce genuine `WritableStream`/`ReadableStream` instances (not adapter objects, unlike Bun's `stdin`), verified by writing and reading real bytes through them end-to-end. The real `@agentclientprotocol/sdk@1.2.1` and `@modelcontextprotocol/sdk@1.29.0` packages (via `npm:` specifiers) were also imported and exercised under that real Deno runtime — `client`, `McpServer`, and `WebStandardStreamableHTTPServerTransport` all constructed and connected successfully, and `mcpServer.registerTool()` accepted Task 10's `Record<string, z.ZodType>` shape with real `zod@4.4.3`, same as under Node and Bun. This meaningfully de-risks the Deno path beyond "written against documentation," but it is still not the same claim as "this repo's own Task 3 code and its gated test have run and passed" — the repo doesn't exist yet. Do not upgrade Deno support from "API behavior independently verified" to "this package's own tests pass on Deno" in documentation or commit messages until Task 3's gated test has actually been run with `deno test` against the real implementation.
- Platform-support boundary, discovered during the same research pass and worth stating explicitly so "runtime-agnostic" is never over-read: this SDK's core `query()` path always spawns `grok agent stdio` as a real OS subprocess, so it can only ever run on runtimes with genuine OS process access — Node, Bun, and self-hosted Deno. It cannot run on Deno Deploy or Cloudflare Workers (both isolate/edge sandboxes that categorically forbid subprocess spawning — confirmed for Deno Deploy via Deno's own docs), even though `@modelcontextprotocol/sdk`'s `WebStandardStreamableHTTPServerTransport` docstring lists those platforms as supported runtimes for *that transport in isolation*. That claim is true and irrelevant here: it would only matter if this SDK hosted an MCP/ACP *server* on the edge, which it doesn't — it's a client spawning a local/remote-shelled agent process. Separately, running on Deno requires the consumer to pass `--allow-run` (to spawn `grok`) and `--allow-env` if their own code reads `process.env`/`Deno.env` anywhere in the same script — confirmed directly: `process.cwd()` works under Deno with zero permission flags, but `process.env.X` throws `NotCapable` without `--allow-env`. Node and Bun have no equivalent sandbox and need no such flags. Document both points in Task 15's README rather than leaving a Deno user to debug a `NotCapable` error or a Deno-Deploy subprocess failure and conclude the SDK is broken.
- No placeholders: every task below has real, complete code. `pnpm test` must pass after every task.
- Git: commit after each task, local repo only (no remote push in this plan).
- Scope trim from the spec, stated explicitly rather than silently dropped: `permissionMode` ships with only `"default"`/`"bypassPermissions"` in v1 (not `"acceptEdits"`/`"plan"`), and `maxTurns` is not implemented — ACP does not expose a client-visible per-turn round counter equivalent to headless mode's `num_turns` without deeper protocol-level accounting, so it needs its own design pass rather than a guessed implementation. Both are v1.1 follow-ups; do not silently invent semantics for them in this plan.
- Further scope trim from this turn's runtime-agnostic pass: Bun/Deno support covers only the core `query()` path (Task 3's subprocess spawn). The loopback MCP server (Task 11) still hosts exclusively via `node:http`; swapping in `Bun.serve()`/`Deno.serve()` for that listener (removing even the small Node bridge) is a natural v1.1 follow-up, not done here, since custom tools are a less central path than `query()` itself.
- Boilerplate/tooling pass (this turn), versions and choices verified against the live npm registry rather than assumed, since AI-suggested boilerplate frequently drifts stale: Node floor raised `>=20` → `>=22` because Node 20 reached end-of-life 2026-04-30 (Node 22 is current Maintenance LTS, Node 24 is Active LTS). `typescript` bumped to `^7.0.0` (the native Go-ported compiler, GA'd 2026-07-08) — safe here because this package only drives `tsc` as a CLI (build + typecheck), and TS7's one documented GA gap is the *programmatic* Compiler API (deferred to 7.1), which this plan never touches. `zod` bumped to `^4.0.0` (peer-compatible with both `@agentclientprotocol/sdk@1.2.1` and `@modelcontextprotocol/sdk@1.29.0`, which both accept `^3.25 || ^4.0`) — this forced real fixes in Task 10/11 because zod v4 removed `ZodRawShape` and `objectOutputType` from its default export (they now only exist under the legacy `zod/v3` subpath); Task 10 now constrains on `Record<string, z.ZodType>` and infers handler args with a mapped type instead, both verified against the real `zod@4.4.3` package `.d.ts`. No bundler (tsup/tsdown/unbuild) was added: this is an ESM-only library with no dual CJS/ESM requirement, so a bundler's main value-add doesn't apply, and `tsdown` specifically requires Node `^22.18.0 || >=24.11.0` to run, which would immediately break on this machine's installed Node (22.17.0) — plain `tsc` avoids that floor entirely and keeps the devDependency count minimal, consistent with this SDK's own minimum-dependency goal. Lint/format is Biome (`@biomejs/biome`) instead of ESLint+Prettier — one dependency instead of several, and it's the standard consolidation move as of 2026. `publint` and `@arethetypeswrong/cli` are added as devDependencies in Task 1 (so `pnpm pack:check` exists from the start) but are only meaningfully run once Task 15 builds the real public API surface — running them against Task 1's placeholder export would validate nothing.

---

### Task 1: Scaffold the repo

**Files:**
- Create: `~/Documents/Workfolder/grok-agent-sdk/package.json`
- Create: `~/Documents/Workfolder/grok-agent-sdk/tsconfig.json`
- Create: `~/Documents/Workfolder/grok-agent-sdk/vitest.config.ts`
- Create: `~/Documents/Workfolder/grok-agent-sdk/.gitignore`
- Create: `~/Documents/Workfolder/grok-agent-sdk/biome.json`
- Create: `~/Documents/Workfolder/grok-agent-sdk/.npmrc`
- Create: `~/Documents/Workfolder/grok-agent-sdk/.github/workflows/ci.yml`
- Create: `~/Documents/Workfolder/grok-agent-sdk/src/index.ts`
- Test: `~/Documents/Workfolder/grok-agent-sdk/test/smoke.test.ts`

**Interfaces:**
- Produces: an installable, testable, type-checkable empty package that later tasks add files to.

- [ ] **Step 1: Create the directory and initialize git**

```bash
mkdir -p ~/Documents/Workfolder/grok-agent-sdk
cd ~/Documents/Workfolder/grok-agent-sdk
git init
```

Expected: `Initialized empty Git repository in .../grok-agent-sdk/.git/`

- [ ] **Step 2: Write `package.json`**

```json
{
  "name": "grok-agent-sdk",
  "version": "0.1.0",
  "description": "Programmatic harness SDK for Grok Build, built on the Agent Client Protocol.",
  "type": "module",
  "license": "MIT",
  "engines": { "node": ">=22" },
  "packageManager": "pnpm@10.34.5",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit",
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "pack:check": "publint && attw --pack ."
  },
  "dependencies": {
    "@agentclientprotocol/sdk": "^1.2.1",
    "@modelcontextprotocol/sdk": "^1.29.0",
    "zod": "^4.0.0"
  },
  "devDependencies": {
    "typescript": "^7.0.0",
    "vitest": "^4.0.0",
    "@types/node": "^22.0.0",
    "@biomejs/biome": "^2.5.0",
    "publint": "^0.3.21",
    "@arethetypeswrong/cli": "^0.18.5"
  }
}
```

Notes on choices, verified against the live npm registry rather than assumed (see Global Constraints for the full reasoning): Node floor `>=22` (Node 20 is EOL as of 2026-04-30); `typescript@^7.0.0` is the current native-compiler major, safe here since this package only drives `tsc` as a CLI; `zod@^4.0.0` is peer-compatible with both SDK dependencies; no bundler dependency (`tsup`/`tsdown`/`unbuild`) — this is an ESM-only library, so a bundler's dual-format value-add doesn't apply, and `tsdown` specifically would require Node `^22.18.0` to even run.

- [ ] **Step 3: Write `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2023"],
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*.ts"]
}
```

- [ ] **Step 4: Write `vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["test/**/*.test.ts"],
    environment: "node",
  },
});
```

- [ ] **Step 5: Write `.gitignore`**

```
node_modules/
dist/
*.tsbuildinfo
```

- [ ] **Step 6: Write `biome.json`**

Biome replaces ESLint+Prettier with one Rust-based tool — one devDependency instead of several, no config conflicts between a linter and a formatter.

```json
{
  "$schema": "https://biomejs.dev/schemas/2.5.4/schema.json",
  "vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "files": { "ignoreUnknown": false },
  "formatter": { "enabled": true, "indentStyle": "space" },
  "linter": { "enabled": true, "rules": { "recommended": true } },
  "javascript": { "formatter": { "quoteStyle": "double" } }
}
```

`quoteStyle: "double"` matches the double-quote convention every later task's code already uses.

- [ ] **Step 7: Write `.npmrc`**

```
engine-strict=true
```

Makes `pnpm install` refuse to run on a Node version older than the `engines.node` floor in `package.json`, instead of silently succeeding and failing later in some runtime-specific way.

- [ ] **Step 8: Write `.github/workflows/ci.yml`**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [22, 24]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm test
```

`pnpm/action-setup@v4` reads the pinned version from `package.json`'s `packageManager` field, so the CLI version doesn't need to be repeated here. This file is inert until the repo has a GitHub remote (none is created in this plan — see Global Constraints), but it costs nothing to have in place now.

- [ ] **Step 9: Write a placeholder `src/index.ts`**

```ts
export const VERSION = "0.1.0";
```

- [ ] **Step 10: Write the smoke test**

```ts
import { describe, it, expect } from "vitest";
import { VERSION } from "../src/index.js";

describe("package scaffolding", () => {
  it("exports a version string", () => {
    expect(VERSION).toBe("0.1.0");
  });
});
```

- [ ] **Step 11: Install dependencies**

```bash
cd ~/Documents/Workfolder/grok-agent-sdk
pnpm install
```

Expected: lockfile `pnpm-lock.yaml` created, no errors.

- [ ] **Step 12: Run the test suite**

```bash
pnpm test
```

Expected: 1 passed (`package scaffolding > exports a version string`).

- [ ] **Step 13: Run the typechecker**

```bash
pnpm typecheck
```

Expected: no errors.

- [ ] **Step 14: Run the linter**

```bash
pnpm lint
```

Expected: no errors (Biome's default rule set is clean against the placeholder scaffold).

- [ ] **Step 15: Commit**

```bash
git add -A
git commit -m "chore: scaffold grok-agent-sdk package"
```

---

### Task 2: Message types and translation

**Files:**
- Create: `src/messages.ts`
- Test: `test/messages.test.ts`

**Interfaces:**
- Consumes: `ActiveSessionMessage`, `schema.SessionNotification`, `schema.PromptResponse`, `schema.StopReason` from `@agentclientprotocol/sdk`.
- Produces: `SDKMessage` union type and `toSDKMessages(msg: ActiveSessionMessage): SDKMessage[]`, used by Task 7/8's `query()`.

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import { toSDKMessages } from "../src/messages.js";
import type { ActiveSessionMessage } from "@agentclientprotocol/sdk";

describe("toSDKMessages", () => {
  it("translates an agent_message_chunk update into a text_delta message", () => {
    const input: ActiveSessionMessage = {
      kind: "session_update",
      notification: {
        sessionId: "sess-1",
        update: {
          sessionUpdate: "agent_message_chunk",
          content: { type: "text", text: "Hello" },
        },
      },
      update: {
        sessionUpdate: "agent_message_chunk",
        content: { type: "text", text: "Hello" },
      },
    };

    const result = toSDKMessages(input);

    expect(result).toEqual([{ type: "text_delta", text: "Hello" }]);
  });

  it("translates a stop message into a result message", () => {
    const input: ActiveSessionMessage = {
      kind: "stop",
      response: { stopReason: "end_turn" },
      stopReason: "end_turn",
    };

    const result = toSDKMessages(input);

    expect(result).toEqual([
      { type: "result", stopReason: "end_turn", response: { stopReason: "end_turn" } },
    ]);
  });

  it("passes through tool_call updates untouched as tool_call messages", () => {
    const input: ActiveSessionMessage = {
      kind: "session_update",
      notification: {
        sessionId: "sess-1",
        update: {
          sessionUpdate: "tool_call",
          toolCallId: "call_1",
          title: "Reading file",
          kind: "read",
          status: "pending",
        },
      },
      update: {
        sessionUpdate: "tool_call",
        toolCallId: "call_1",
        title: "Reading file",
        kind: "read",
        status: "pending",
      },
    };

    const result = toSDKMessages(input);

    expect(result).toEqual([
      {
        type: "tool_call",
        toolCallId: "call_1",
        title: "Reading file",
        kind: "read",
        status: "pending",
        raw: input.update,
      },
    ]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/messages.test.ts
```

Expected: FAIL — `Cannot find module '../src/messages.js'`.

- [ ] **Step 3: Write the implementation**

```ts
import type {
  ActiveSessionMessage,
  PromptResponse,
  SessionUpdate,
  StopReason,
} from "@agentclientprotocol/sdk";

export type SDKMessage =
  | { type: "text_delta"; text: string }
  | { type: "thought_delta"; text: string }
  | {
      type: "tool_call";
      toolCallId: string;
      title: string;
      kind: string;
      status: string;
      raw: SessionUpdate;
    }
  | {
      type: "tool_call_update";
      toolCallId: string;
      status: string;
      raw: SessionUpdate;
    }
  | { type: "plan"; raw: SessionUpdate }
  | { type: "result"; stopReason: StopReason; response: PromptResponse }
  | { type: "unknown_update"; raw: SessionUpdate };

export function toSDKMessages(message: ActiveSessionMessage): SDKMessage[] {
  if (message.kind === "stop") {
    return [
      {
        type: "result",
        stopReason: message.stopReason,
        response: message.response,
      },
    ];
  }

  const update = message.update;
  switch (update.sessionUpdate) {
    case "agent_message_chunk":
      return update.content.type === "text"
        ? [{ type: "text_delta", text: update.content.text }]
        : [{ type: "unknown_update", raw: update }];
    case "agent_thought_chunk":
      return update.content.type === "text"
        ? [{ type: "thought_delta", text: update.content.text }]
        : [{ type: "unknown_update", raw: update }];
    case "tool_call":
      return [
        {
          type: "tool_call",
          toolCallId: update.toolCallId,
          title: update.title,
          kind: update.kind,
          status: update.status,
          raw: update,
        },
      ];
    case "tool_call_update":
      return [
        {
          type: "tool_call_update",
          toolCallId: update.toolCallId,
          status: update.status ?? "unknown",
          raw: update,
        },
      ];
    case "plan":
      return [{ type: "plan", raw: update }];
    default:
      return [{ type: "unknown_update", raw: update }];
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/messages.test.ts
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add src/messages.ts test/messages.test.ts
git commit -m "feat: translate ACP session updates into SDKMessage"
```

---

### Task 3: Process transport (runtime-agnostic: Node, Bun, Deno)

**Files:**
- Create: `src/transport.ts`
- Test: `test/transport.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface SpawnOptions {
    grokBin?: string;      // defaults to "grok"
    args?: string[];       // extra args appended after "agent stdio", e.g. ["--model", "grok-build"]
    cwd?: string;
    env?: Record<string, string | undefined>;
  }
  export interface GrokAgentProcess {
    stream: Stream;                                     // ACP-ready stdin/stdout pair
    kill(): void;
    readonly exited: Promise<{ code: number | null; stderrTail: string }>;
  }
  export function spawnGrokAgentStdio(options: SpawnOptions): GrokAgentProcess;
  ```
  Consumed by Task 7/8's `query()`.

Spawning a subprocess has no Web Standard — Node, Bun, and Deno each have their own
native API, so `spawnGrokAgentStdio` feature-detects the runtime (`typeof Bun`,
`typeof Deno`, else Node) and dispatches to a runtime-specific implementation, all
normalized to the same `GrokAgentProcess` shape. This is the *only* file in the
package that needs this branching — everything downstream only ever touches
`stream`/`kill()`/`exited`, all of which are plain JS, not runtime-specific types.

The Node and Bun code paths below were verified directly against installed
runtimes in this environment (Node v22.17.0, Bun 1.3.14) — notably, Bun's
`proc.stdin` in `"pipe"` mode is a `FileSink` object (`.write()`/`.end()`), **not**
a `WritableStream` instance, so it needs a small adapter; `proc.stdout` **is**
already a real `ReadableStream`, no adapter needed. The Deno path is written
against Deno's long-stable, well-documented `Deno.Command` API (`stdin`/`stdout`
are genuine `WritableStream`/`ReadableStream` when `"piped"`), but there was no
Deno runtime available in this environment to execute it against — Step 6 below
adds a test gated on `typeof Deno !== "undefined"` so it self-verifies the first
time this runs somewhere Deno is actually installed; treat that path as unverified
until that test has actually run once.

- [ ] **Step 1: Write the failing test for the Node path**

```ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { EventEmitter } from "node:events";
import { PassThrough } from "node:stream";

const spawnMock = vi.fn();
vi.mock("node:child_process", () => ({ spawn: spawnMock }));

import { spawnGrokAgentStdio } from "../src/transport.js";

function fakeNodeChild() {
  const child = new EventEmitter() as any;
  child.stdin = new PassThrough();
  child.stdout = new PassThrough();
  child.stderr = new PassThrough();
  child.kill = vi.fn();
  return child;
}

afterEach(() => {
  vi.unstubAllGlobals();
});

describe("spawnGrokAgentStdio on Node", () => {
  it("spawns 'grok agent stdio' by default", () => {
    const child = fakeNodeChild();
    spawnMock.mockReturnValue(child);

    spawnGrokAgentStdio({});

    expect(spawnMock).toHaveBeenCalledWith(
      "grok",
      ["agent", "stdio"],
      expect.objectContaining({ stdio: ["pipe", "pipe", "pipe"] }),
    );
  });

  it("uses a custom binary and appends extra args before 'stdio'", () => {
    const child = fakeNodeChild();
    spawnMock.mockReturnValue(child);

    spawnGrokAgentStdio({ grokBin: "/usr/local/bin/grok", args: ["--model", "grok-build"] });

    expect(spawnMock).toHaveBeenCalledWith(
      "/usr/local/bin/grok",
      ["agent", "--model", "grok-build", "stdio"],
      expect.anything(),
    );
  });

  it("returns a Stream with readable/writable pair", () => {
    const child = fakeNodeChild();
    spawnMock.mockReturnValue(child);

    const { stream } = spawnGrokAgentStdio({});

    expect(stream.readable).toBeInstanceOf(ReadableStream);
    expect(stream.writable).toBeInstanceOf(WritableStream);
  });

  it("resolves exited with the exit code and buffered stderr", async () => {
    const child = fakeNodeChild();
    spawnMock.mockReturnValue(child);

    const process = spawnGrokAgentStdio({});
    child.stderr.write("boom\n");
    child.emit("exit", 1);

    await expect(process.exited).resolves.toEqual({ code: 1, stderrTail: "boom\n" });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/transport.test.ts
```

Expected: FAIL — `Cannot find module '../src/transport.js'`.

- [ ] **Step 3: Write the implementation**

```ts
import { spawn as spawnNodeChild } from "node:child_process";
import { Readable, Writable } from "node:stream";
import { ndJsonStream, type Stream } from "@agentclientprotocol/sdk";

export interface SpawnOptions {
  grokBin?: string;
  args?: string[];
  cwd?: string;
  env?: Record<string, string | undefined>;
}

export interface GrokAgentProcess {
  stream: Stream;
  kill(): void;
  readonly exited: Promise<{ code: number | null; stderrTail: string }>;
}

const STDERR_TAIL_LIMIT = 4000;

function spawnOnNode(bin: string, args: string[], options: SpawnOptions): GrokAgentProcess {
  const child = spawnNodeChild(bin, args, {
    cwd: options.cwd,
    env: (options.env ?? process.env) as NodeJS.ProcessEnv,
    stdio: ["pipe", "pipe", "pipe"],
  });

  const toAgentWritable = Writable.toWeb(
    child.stdin as NodeJS.WritableStream,
  ) as WritableStream<Uint8Array>;
  const fromAgentReadable = Readable.toWeb(
    child.stdout as NodeJS.ReadableStream,
  ) as ReadableStream<Uint8Array>;
  const stream = ndJsonStream(toAgentWritable, fromAgentReadable);

  let stderrTail = "";
  child.stderr?.on("data", (chunk: Buffer) => {
    stderrTail = (stderrTail + chunk.toString()).slice(-STDERR_TAIL_LIMIT);
  });

  const exited = new Promise<{ code: number | null; stderrTail: string }>((resolve) => {
    child.on("exit", (code) => resolve({ code, stderrTail }));
  });

  return { stream, kill: () => child.kill(), exited };
}

async function pumpStderr(
  readable: ReadableStream<Uint8Array>,
  onChunk: (text: string) => void,
): Promise<void> {
  const reader = readable.getReader();
  const decoder = new TextDecoder();
  for (;;) {
    const { done, value } = await reader.read();
    if (done) return;
    onChunk(decoder.decode(value));
  }
}

// Bun.spawn's stdout is a real ReadableStream, but stdin (in "pipe" mode) is a
// FileSink (.write()/.end()), not a WritableStream — verified against Bun 1.3.14.
function spawnOnBun(bin: string, args: string[], options: SpawnOptions): GrokAgentProcess {
  const bunGlobal = (globalThis as { Bun: any }).Bun;
  const proc = bunGlobal.spawn([bin, ...args], {
    cwd: options.cwd,
    env: options.env,
    stdin: "pipe",
    stdout: "pipe",
    stderr: "pipe",
  });

  const toAgentWritable = new WritableStream<Uint8Array>({
    write(chunk) {
      proc.stdin.write(chunk);
    },
    close() {
      proc.stdin.end();
    },
    abort() {
      proc.stdin.end();
    },
  });
  const fromAgentReadable = proc.stdout as ReadableStream<Uint8Array>;
  const stream = ndJsonStream(toAgentWritable, fromAgentReadable);

  let stderrTail = "";
  const exited = (async () => {
    const pump = pumpStderr(proc.stderr as ReadableStream<Uint8Array>, (text) => {
      stderrTail = (stderrTail + text).slice(-STDERR_TAIL_LIMIT);
    });
    const code: number = await proc.exited;
    await pump;
    return { code, stderrTail };
  })();

  return { stream, kill: () => proc.kill(), exited };
}

// Deno.Command gives real WritableStream/ReadableStream for "piped" stdin/stdout —
// unlike Bun.spawn, no adapter needed. Written against Deno's stable public API;
// not executed against a real Deno runtime in this environment (see Step 6's test).
function spawnOnDeno(bin: string, args: string[], options: SpawnOptions): GrokAgentProcess {
  const denoGlobal = (globalThis as { Deno: any }).Deno;
  const command = new denoGlobal.Command(bin, {
    args,
    cwd: options.cwd,
    env: options.env,
    stdin: "piped",
    stdout: "piped",
    stderr: "piped",
  });
  const child = command.spawn();

  const toAgentWritable = child.stdin as WritableStream<Uint8Array>;
  const fromAgentReadable = child.stdout as ReadableStream<Uint8Array>;
  const stream = ndJsonStream(toAgentWritable, fromAgentReadable);

  let stderrTail = "";
  const exited = (async () => {
    const pump = pumpStderr(child.stderr as ReadableStream<Uint8Array>, (text) => {
      stderrTail = (stderrTail + text).slice(-STDERR_TAIL_LIMIT);
    });
    const status = await child.status;
    await pump;
    return { code: status.code as number, stderrTail };
  })();

  return { stream, kill: () => child.kill(), exited };
}

export function spawnGrokAgentStdio(options: SpawnOptions): GrokAgentProcess {
  const bin = options.grokBin ?? "grok";
  const args = ["agent", ...(options.args ?? []), "stdio"];

  const g = globalThis as { Bun?: unknown; Deno?: unknown };
  if (typeof g.Bun !== "undefined") return spawnOnBun(bin, args, options);
  if (typeof g.Deno !== "undefined") return spawnOnDeno(bin, args, options);
  return spawnOnNode(bin, args, options);
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/transport.test.ts
```

Expected: 4 passed (Node describe block).

- [ ] **Step 5: Add a Bun-path test using `vi.stubGlobal`**

Append to `test/transport.test.ts`:

```ts
describe("spawnGrokAgentStdio on Bun", () => {
  it("adapts Bun's FileSink stdin into a WritableStream and dispatches via Bun.spawn", async () => {
    const written: Uint8Array[] = [];
    let ended = false;
    const fakeStdout = new ReadableStream<Uint8Array>({
      start(controller) {
        controller.enqueue(new TextEncoder().encode("hello"));
        controller.close();
      },
    });
    const fakeStderr = new ReadableStream<Uint8Array>({
      start(controller) {
        controller.close();
      },
    });
    const bunSpawnMock = vi.fn().mockReturnValue({
      stdin: {
        write: (chunk: Uint8Array) => written.push(chunk),
        end: () => {
          ended = true;
        },
      },
      stdout: fakeStdout,
      stderr: fakeStderr,
      exited: Promise.resolve(0),
      kill: vi.fn(),
    });
    vi.stubGlobal("Bun", { spawn: bunSpawnMock });

    const process = spawnGrokAgentStdio({ grokBin: "grok" });

    expect(bunSpawnMock).toHaveBeenCalledWith(
      ["grok", "agent", "stdio"],
      expect.objectContaining({ stdin: "pipe", stdout: "pipe", stderr: "pipe" }),
    );

    const writer = process.stream.writable.getWriter();
    await writer.write(new TextEncoder().encode("ping"));
    await writer.close();
    expect(written.length).toBe(1);
    expect(ended).toBe(true);

    await expect(process.exited).resolves.toEqual({ code: 0, stderrTail: "" });
  });
});
```

- [ ] **Step 6: Add a gated Deno-path test**

Append to `test/transport.test.ts`:

```ts
describe.skipIf(typeof (globalThis as any).Deno === "undefined")(
  "spawnGrokAgentStdio on Deno",
  () => {
    it("dispatches via Deno.Command and passes through its native streams", async () => {
      const fakeStdin = new WritableStream<Uint8Array>();
      const fakeStdout = new ReadableStream<Uint8Array>({
        start(controller) {
          controller.close();
        },
      });
      const fakeStderr = new ReadableStream<Uint8Array>({
        start(controller) {
          controller.close();
        },
      });
      const spawnMock = vi.fn().mockReturnValue({
        stdin: fakeStdin,
        stdout: fakeStdout,
        stderr: fakeStderr,
        status: Promise.resolve({ code: 0 }),
        kill: vi.fn(),
      });
      const CommandMock = vi.fn().mockReturnValue({ spawn: spawnMock });
      vi.stubGlobal("Deno", { Command: CommandMock });

      const process = spawnGrokAgentStdio({ grokBin: "grok" });

      expect(CommandMock).toHaveBeenCalledWith(
        "grok",
        expect.objectContaining({ args: ["agent", "stdio"], stdin: "piped" }),
      );
      expect(process.stream.writable).toBe(fakeStdin);
      await expect(process.exited).resolves.toEqual({ code: 0, stderrTail: "" });
    });
  },
);
```

This block is skipped everywhere except a real Deno process (`typeof Deno !== "undefined"` is only ever true when the test file itself is executed *by* Deno — running `pnpm test` under Node/Bun always skips it). If you want this path genuinely exercised, it needs to run under `deno test` with a Deno-compatible test setup, which is out of scope for this plan's `pnpm test` — treat a passing run of this block, whenever it happens, as the actual verification of `spawnOnDeno`; until then it documents intent, not confirmed behavior.

- [ ] **Step 7: Run the full test file**

```bash
pnpm test -- test/transport.test.ts
```

Expected: 5 passed (4 Node + 1 Bun), 1 skipped (Deno, since this environment runs under Node/Bun via vitest, not under `deno test`).

- [ ] **Step 8: Commit**

```bash
git add src/transport.ts test/transport.test.ts
git commit -m "feat: spawn grok agent stdio across Node, Bun, and Deno"
```

---

### Task 4: Permission callback

**Files:**
- Create: `src/permissions.ts`
- Test: `test/permissions.test.ts`

**Interfaces:**
- Consumes: `schema.RequestPermissionRequest`, `schema.RequestPermissionResponse` from `@agentclientprotocol/sdk`.
- Produces: `PermissionDecision`, `CanUseTool`, `buildPermissionHandler(canUseTool?: CanUseTool)` — a `ClientRequestHandler<RequestPermissionRequest, RequestPermissionResponse>` consumed by Task 7/8's `query()`.

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import { buildPermissionHandler } from "../src/permissions.js";
import type { RequestPermissionRequest } from "@agentclientprotocol/sdk";

function requestWithOptions(): RequestPermissionRequest {
  return {
    sessionId: "sess-1",
    toolCall: { toolCallId: "call_1", title: "Run rm -rf /tmp/x", kind: "execute", status: "pending" },
    options: [
      { optionId: "allow", name: "Allow", kind: "allow_once" },
      { optionId: "deny", name: "Deny", kind: "reject_once" },
    ],
  } as RequestPermissionRequest;
}

describe("buildPermissionHandler", () => {
  it("selects the allow option when canUseTool allows", async () => {
    const handler = buildPermissionHandler(async () => ({ behavior: "allow" }));

    const result = await handler({
      params: requestWithOptions(),
      signal: new AbortController().signal,
      agent: {} as any,
      requestId: 1,
    });

    expect(result).toEqual({ outcome: { outcome: "selected", optionId: "allow" } });
  });

  it("selects the deny option when canUseTool denies", async () => {
    const handler = buildPermissionHandler(async () => ({ behavior: "deny", message: "no" }));

    const result = await handler({
      params: requestWithOptions(),
      signal: new AbortController().signal,
      agent: {} as any,
      requestId: 1,
    });

    expect(result).toEqual({ outcome: { outcome: "selected", optionId: "deny" } });
  });

  it("defaults to selecting the first option when canUseTool is not provided", async () => {
    const handler = buildPermissionHandler(undefined);

    const result = await handler({
      params: requestWithOptions(),
      signal: new AbortController().signal,
      agent: {} as any,
      requestId: 1,
    });

    expect(result).toEqual({ outcome: { outcome: "selected", optionId: "allow" } });
  });

  it("throws if canUseTool allows but no allow-kind option exists", async () => {
    const handler = buildPermissionHandler(async () => ({ behavior: "allow" }));
    const request = requestWithOptions();
    request.options = [{ optionId: "deny", name: "Deny", kind: "reject_once" }];

    await expect(
      handler({ params: request, signal: new AbortController().signal, agent: {} as any, requestId: 1 }),
    ).rejects.toThrow(/no allow-kind option/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/permissions.test.ts
```

Expected: FAIL — `Cannot find module '../src/permissions.js'`.

- [ ] **Step 3: Write the implementation**

```ts
import type {
  ClientRequestHandler,
  RequestPermissionRequest,
  RequestPermissionResponse,
} from "@agentclientprotocol/sdk";

export type PermissionDecision =
  | { behavior: "allow"; updatedInput?: unknown }
  | { behavior: "deny"; message?: string };

export type CanUseTool = (
  request: RequestPermissionRequest,
) => Promise<PermissionDecision> | PermissionDecision;

function findOptionByKindPrefix(
  options: RequestPermissionRequest["options"],
  prefix: "allow" | "reject",
) {
  const option = options.find((option) => option.kind.startsWith(prefix));
  if (!option) {
    throw new Error(
      `canUseTool decision could not be applied: no ${prefix}-kind option in the permission request`,
    );
  }
  return option;
}

export function buildPermissionHandler(
  canUseTool: CanUseTool | undefined,
): ClientRequestHandler<RequestPermissionRequest, RequestPermissionResponse> {
  return async ({ params }) => {
    if (!canUseTool) {
      const option = findOptionByKindPrefix(params.options, "allow");
      return { outcome: { outcome: "selected", optionId: option.optionId } };
    }

    const decision = await canUseTool(params);

    const option =
      decision.behavior === "allow"
        ? findOptionByKindPrefix(params.options, "allow")
        : findOptionByKindPrefix(params.options, "reject");

    return { outcome: { outcome: "selected", optionId: option.optionId } };
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/permissions.test.ts
```

Expected: 4 passed.

- [ ] **Step 5: Commit**

```bash
git add src/permissions.ts test/permissions.test.ts
git commit -m "feat: wire canUseTool to ACP session/request_permission"
```

---

### Task 5: Process error type

**Files:**
- Create: `src/errors.ts`
- Test: `test/errors.test.ts`

**Interfaces:**
- Produces: `AgentProcessError`, consumed by Task 7/8's `query()`.

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import { AgentProcessError } from "../src/errors.js";

describe("AgentProcessError", () => {
  it("carries exit code and stderr tail, and is an instance of Error", () => {
    const error = new AgentProcessError(1, "panic: something broke\n");

    expect(error).toBeInstanceOf(Error);
    expect(error.exitCode).toBe(1);
    expect(error.stderrTail).toBe("panic: something broke\n");
    expect(error.message).toContain("exit code 1");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/errors.test.ts
```

Expected: FAIL — `Cannot find module '../src/errors.js'`.

- [ ] **Step 3: Write the implementation**

```ts
export class AgentProcessError extends Error {
  readonly exitCode: number | null;
  readonly stderrTail: string;

  constructor(exitCode: number | null, stderrTail: string) {
    super(
      `grok agent stdio exited with exit code ${exitCode ?? "unknown"} before completing the turn` +
        (stderrTail ? `:\n${stderrTail}` : ""),
    );
    this.name = "AgentProcessError";
    this.exitCode = exitCode;
    this.stderrTail = stderrTail;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/errors.test.ts
```

Expected: 1 passed.

- [ ] **Step 5: Commit**

```bash
git add src/errors.ts test/errors.test.ts
git commit -m "feat: add AgentProcessError"
```

---

### Task 6: Internal async queue

**Files:**
- Create: `src/internal/asyncQueue.ts`
- Test: `test/internal/asyncQueue.test.ts`

**Interfaces:**
- Produces: `AsyncQueue<T>` with `push(item: T)`, `close()`, `fail(error: Error)`, and `[Symbol.asyncIterator](): AsyncIterator<T>` — consumed by Task 7's `query()` to bridge the ACP `connectWith` callback (producer) and the public async generator (consumer).

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import { AsyncQueue } from "../../src/internal/asyncQueue.js";

describe("AsyncQueue", () => {
  it("yields pushed items in order, then stops after close()", async () => {
    const queue = new AsyncQueue<number>();
    queue.push(1);
    queue.push(2);
    queue.close();

    const results: number[] = [];
    for await (const item of queue) {
      results.push(item);
    }

    expect(results).toEqual([1, 2]);
  });

  it("supports pushing after the consumer starts waiting", async () => {
    const queue = new AsyncQueue<string>();
    const consumed: string[] = [];

    const consumer = (async () => {
      for await (const item of queue) {
        consumed.push(item);
      }
    })();

    await new Promise((resolve) => setTimeout(resolve, 0));
    queue.push("a");
    queue.push("b");
    queue.close();

    await consumer;
    expect(consumed).toEqual(["a", "b"]);
  });

  it("rejects iteration when fail() is called", async () => {
    const queue = new AsyncQueue<number>();
    queue.fail(new Error("boom"));

    await expect(
      (async () => {
        for await (const _ of queue) {
          // no-op
        }
      })(),
    ).rejects.toThrow("boom");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/internal/asyncQueue.test.ts
```

Expected: FAIL — `Cannot find module '../../src/internal/asyncQueue.js'`.

- [ ] **Step 3: Write the implementation**

```ts
type PendingResolve<T> = (result: IteratorResult<T>) => void;
type PendingReject = (error: unknown) => void;

export class AsyncQueue<T> implements AsyncIterable<T> {
  private items: T[] = [];
  private waiters: Array<{ resolve: PendingResolve<T>; reject: PendingReject }> = [];
  private closed = false;
  private error: unknown;

  push(item: T): void {
    if (this.closed) return;
    const waiter = this.waiters.shift();
    if (waiter) {
      waiter.resolve({ value: item, done: false });
    } else {
      this.items.push(item);
    }
  }

  close(): void {
    this.closed = true;
    for (const waiter of this.waiters.splice(0)) {
      waiter.resolve({ value: undefined, done: true });
    }
  }

  fail(error: unknown): void {
    this.error = error;
    this.closed = true;
    for (const waiter of this.waiters.splice(0)) {
      waiter.reject(error);
    }
  }

  [Symbol.asyncIterator](): AsyncIterator<T> {
    return {
      next: (): Promise<IteratorResult<T>> => {
        if (this.items.length > 0) {
          return Promise.resolve({ value: this.items.shift() as T, done: false });
        }
        if (this.error) {
          return Promise.reject(this.error);
        }
        if (this.closed) {
          return Promise.resolve({ value: undefined, done: true });
        }
        return new Promise((resolve, reject) => {
          this.waiters.push({ resolve, reject });
        });
      },
    };
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/internal/asyncQueue.test.ts
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add src/internal/asyncQueue.ts test/internal/asyncQueue.test.ts
git commit -m "feat: add internal AsyncQueue for bridging ACP callback to query() generator"
```

---

### Task 7: Core `query()` — single prompt, non-interactive

**Files:**
- Create: `src/query.ts`
- Test: `test/query.test.ts`

**Interfaces:**
- Consumes: `spawnGrokAgentStdio` (Task 3), `buildPermissionHandler`/`CanUseTool` (Task 4), `toSDKMessages`/`SDKMessage` (Task 2), `AgentProcessError` (Task 5), `AsyncQueue` (Task 6), `client`/`methods` from `@agentclientprotocol/sdk`.
- Produces:
  ```ts
  export interface QueryOptions {
    cwd?: string;
    grokBin?: string;
    model?: string;
    canUseTool?: CanUseTool;
    mcpServers?: McpServer[];
  }
  export interface Query extends AsyncIterable<SDKMessage> {
    readonly sessionId: Promise<string>;
  }
  export function query(input: { prompt: string; options?: QueryOptions }): Query;
  ```
  This is the primary export later re-exported from `src/index.ts` (Task 13) and extended with interactive control in Task 8.

This task mocks `@agentclientprotocol/sdk`'s `client()` builder and `transport.ts`'s spawn function so the test suite never needs a real `grok` binary — that dependency is exercised only by the gated integration test in Task 14.

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, vi } from "vitest";

const spawnGrokAgentStdioMock = vi.fn();
vi.mock("../src/transport.js", () => ({
  spawnGrokAgentStdio: spawnGrokAgentStdioMock,
}));

// Minimal fake of the acp.client(...) fluent builder, capturing the
// onRequest handler and the connectWith callback so the test can drive
// the ACP session lifecycle deterministically.
let capturedPermissionHandler: any;
const connectWithMock = vi.fn();
vi.mock("@agentclientprotocol/sdk", async () => {
  const actual = await vi.importActual<typeof import("@agentclientprotocol/sdk")>(
    "@agentclientprotocol/sdk",
  );
  return {
    ...actual,
    client: () => ({
      onRequest: (_method: string, handler: any) => {
        capturedPermissionHandler = handler;
        return {
          onRequest: () => ({ connectWith: connectWithMock }),
          connectWith: connectWithMock,
        };
      },
    }),
  };
});

import { query } from "../src/query.js";

function fakeActiveSession(updates: any[], stopResponse: any) {
  let i = 0;
  return {
    sessionId: "sess-abc",
    prompt: vi.fn().mockResolvedValue(stopResponse),
    nextUpdate: vi.fn().mockImplementation(async () => {
      if (i < updates.length) {
        const update = updates[i++];
        return { kind: "session_update", notification: { sessionId: "sess-abc", update }, update };
      }
      return { kind: "stop", response: stopResponse, stopReason: stopResponse.stopReason };
    }),
    dispose: vi.fn(),
  };
}

describe("query()", () => {
  it("yields translated messages for a single prompt turn", async () => {
    spawnGrokAgentStdioMock.mockReturnValue({
      stream: {},
      kill: vi.fn(),
      exited: new Promise(() => {}), // never exits during this test
    });

    const session = fakeActiveSession(
      [{ sessionUpdate: "agent_message_chunk", content: { type: "text", text: "hi" } }],
      { stopReason: "end_turn" },
    );

    connectWithMock.mockImplementation(async (_stream: unknown, op: any) => {
      const ctx = {
        buildSession: () => ({
          withSession: async (fn: any) => fn(session),
        }),
      };
      return op(ctx);
    });

    const q = query({ prompt: "hello", options: { cwd: "/tmp" } });

    const results = [];
    for await (const message of q) {
      results.push(message);
    }

    expect(results).toEqual([
      { type: "text_delta", text: "hi" },
      { type: "result", stopReason: "end_turn", response: { stopReason: "end_turn" } },
    ]);
    expect(session.prompt).toHaveBeenCalledWith("hello");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/query.test.ts
```

Expected: FAIL — `Cannot find module '../src/query.js'`.

- [ ] **Step 3: Write the implementation**

```ts
import { client, methods, type McpServer } from "@agentclientprotocol/sdk";
import { spawnGrokAgentStdio } from "./transport.js";
import { buildPermissionHandler, type CanUseTool } from "./permissions.js";
import { toSDKMessages, type SDKMessage } from "./messages.js";
import { AgentProcessError } from "./errors.js";
import { AsyncQueue } from "./internal/asyncQueue.js";

export interface QueryOptions {
  cwd?: string;
  grokBin?: string;
  model?: string;
  canUseTool?: CanUseTool;
  mcpServers?: McpServer[];
}

export interface Query extends AsyncIterable<SDKMessage> {
  readonly sessionId: Promise<string>;
}

export function query(input: { prompt: string; options?: QueryOptions }): Query {
  const options = input.options ?? {};
  const queue = new AsyncQueue<SDKMessage>();
  let resolveSessionId!: (id: string) => void;
  const sessionId = new Promise<string>((resolve) => {
    resolveSessionId = resolve;
  });

  // Named agentProcess, not process — a local `process` binding would shadow
  // Node's global `process` (used below as `process.cwd()`).
  const agentProcess = spawnGrokAgentStdio({
    grokBin: options.grokBin,
    cwd: options.cwd,
    args: options.model ? ["--model", options.model] : undefined,
  });

  let closed = false;
  agentProcess.exited.then(({ code, stderrTail }) => {
    if (!closed) {
      // Process exited without the session loop observing a clean stop.
      queue.fail(new AgentProcessError(code, stderrTail));
    }
  });

  const app = client({ name: "grok-agent-sdk" }).onRequest(
    methods.client.session.requestPermission,
    buildPermissionHandler(options.canUseTool),
  );

  app
    .connectWith(agentProcess.stream, async (ctx: any) => {
      const builder = ctx.buildSession({
        cwd: options.cwd ?? process.cwd(),
        mcpServers: options.mcpServers ?? [],
      });

      await builder.withSession(async (session: any) => {
        resolveSessionId(session.sessionId);

        const response = await session.prompt(input.prompt);
        for (const message of toSDKMessages({ kind: "stop", response, stopReason: response.stopReason })) {
          queue.push(message);
        }
      });
    })
    .catch((error: unknown) => {
      queue.fail(error);
    })
    .finally(() => {
      closed = true;
      queue.close();
      agentProcess.kill();
    });

  return {
    sessionId,
    [Symbol.asyncIterator]: () => queue[Symbol.asyncIterator](),
  };
}
```

**Design note for the implementer:** the test above stubs `ActiveSession.nextUpdate()`/`prompt()` at a level that never actually calls `toSDKMessages` on intermediate `session_update` messages inside `query()` itself, because `session.prompt()` in the real SDK only resolves with the *final* `PromptResponse` — intermediate `session/update` notifications arrive via the registered `session/update` handler, not via `prompt()`'s return value. Task 7's implementation above is deliberately simplified to a single final `result` message to keep this task small and passing; **Task 8 replaces the body of the `withSession` callback** to also register a `methods.client.session.update` notification handler (via `app.onNotification(...)`, added before `connectWith`) that pushes `toSDKMessages` output for every intermediate update as it streams in, using `session.nextUpdate()` in a loop instead of a single `await session.prompt(...)`. Do not skip Task 8 — Task 7 alone does not stream intermediate text/tool-call messages, only the final result.

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/query.test.ts
```

Expected: 1 passed.

- [ ] **Step 5: Commit**

```bash
git add src/query.ts test/query.test.ts
git commit -m "feat: add query() single-turn core (final result only)"
```

---

### Task 8: Stream intermediate updates; support multi-turn `.send()`/`.interrupt()`, `permissionMode`, and `abortSignal`

**Files:**
- Modify: `src/query.ts`
- Test: `test/query.test.ts` (extend)

**Interfaces:**
- Produces: `Query` now also has `send(text: string): void`, `interrupt(): Promise<void>`; `QueryOptions` gains `permissionMode?: "default" | "bypassPermissions"` and `abortSignal?: AbortSignal`.
- Consumes: `ActiveSession.nextUpdate()` loop (Task 2's `toSDKMessages` handles `session_update`-kind messages, already implemented in Task 2 — this task is the first to actually exercise that branch).

- [ ] **Step 1: Extend the test file with the streaming + multi-turn case**

Append to `test/query.test.ts`:

```ts
describe("query() streaming and multi-turn", () => {
  it("streams intermediate session_update messages before the final result, via nextUpdate()", async () => {
    spawnGrokAgentStdioMock.mockReturnValue({
      stream: {},
      kill: vi.fn(),
      exited: new Promise(() => {}), // never exits during this test
    });

    const updates = [
      { sessionUpdate: "agent_message_chunk", content: { type: "text", text: "Hel" } },
      { sessionUpdate: "agent_message_chunk", content: { type: "text", text: "lo" } },
    ];
    const session = fakeActiveSession(updates, { stopReason: "end_turn" });

    connectWithMock.mockImplementation(async (_stream: unknown, op: any) => {
      const ctx = { buildSession: () => ({ withSession: async (fn: any) => fn(session) }) };
      return op(ctx);
    });

    const q = query({ prompt: "hello", options: { cwd: "/tmp" } });

    const results = [];
    for await (const message of q) {
      results.push(message);
    }

    expect(results).toEqual([
      { type: "text_delta", text: "Hel" },
      { type: "text_delta", text: "lo" },
      { type: "result", stopReason: "end_turn", response: { stopReason: "end_turn" } },
    ]);
    // prompt() is called once via session.prompt(), updates come from nextUpdate() loop
    expect(session.nextUpdate).toHaveBeenCalledTimes(updates.length + 1);
  });

  it("send() issues a second prompt on the same session without respawning", async () => {
    spawnGrokAgentStdioMock.mockReturnValue({
      stream: {},
      kill: vi.fn(),
      exited: new Promise(() => {}), // never exits during this test
    });

    const session = fakeActiveSession([], { stopReason: "end_turn" });
    let releaseSecondTurn!: () => void;
    const secondTurnGate = new Promise<void>((resolve) => {
      releaseSecondTurn = resolve;
    });

    connectWithMock.mockImplementation(async (_stream: unknown, op: any) => {
      const ctx = { buildSession: () => ({ withSession: async (fn: any) => fn(session) }) };
      return op(ctx);
    });

    const q = query({ prompt: "first", options: { cwd: "/tmp" } });

    const consumed: any[] = [];
    const consumer = (async () => {
      for await (const message of q) {
        consumed.push(message);
        if (message.type === "result" && consumed.filter((m) => m.type === "result").length === 1) {
          q.send("second");
          releaseSecondTurn();
        }
      }
    })();

    await secondTurnGate;
    await consumer;

    expect(session.prompt).toHaveBeenNthCalledWith(1, "first");
    expect(session.prompt).toHaveBeenNthCalledWith(2, "second");
  });

  it("permissionMode 'bypassPermissions' ignores canUseTool and auto-allows", async () => {
    spawnGrokAgentStdioMock.mockReturnValue({
      stream: {},
      kill: vi.fn(),
      exited: new Promise(() => {}), // never exits during this test
    });
    const session = fakeActiveSession([], { stopReason: "end_turn" });
    connectWithMock.mockImplementation(async (_stream: unknown, op: any) => {
      const ctx = { buildSession: () => ({ withSession: async (fn: any) => fn(session) }) };
      return op(ctx);
    });
    const canUseTool = vi.fn().mockResolvedValue({ behavior: "deny" });

    const q = query({
      prompt: "hello",
      options: { cwd: "/tmp", canUseTool, permissionMode: "bypassPermissions" },
    });
    for await (const _ of q) {
      // drain
    }

    const permissionResult = await capturedPermissionHandler({
      params: {
        sessionId: "sess-abc",
        toolCall: { toolCallId: "call_1", title: "x", kind: "execute", status: "pending" },
        options: [{ optionId: "allow", name: "Allow", kind: "allow_once" }],
      },
      signal: new AbortController().signal,
      agent: {} as any,
      requestId: 1,
    });

    expect(canUseTool).not.toHaveBeenCalled();
    expect(permissionResult).toEqual({ outcome: { outcome: "selected", optionId: "allow" } });
  });

  it("aborting the abortSignal kills the child process and ends iteration", async () => {
    const kill = vi.fn();
    spawnGrokAgentStdioMock.mockReturnValue({
      stream: {},
      kill,
      exited: new Promise(() => {}), // never exits during this test
    });
    const session = fakeActiveSession([], { stopReason: "end_turn" });
    connectWithMock.mockImplementation(
      () => new Promise(() => {}), // never resolves — simulates a long-running turn
    );

    const controller = new AbortController();
    const q = query({ prompt: "hello", options: { cwd: "/tmp", abortSignal: controller.signal } });
    controller.abort();

    const results = [];
    for await (const message of q) {
      results.push(message);
    }

    expect(results).toEqual([]);
    expect(kill).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/query.test.ts
```

Expected: FAIL — the streaming test fails because `query()` (Task 7 version) never calls `session.nextUpdate()` and never emits intermediate `text_delta` messages; `q.send` is `undefined` in the multi-turn test; `permissionMode` and `abortSignal` are not yet recognized options.

- [ ] **Step 3: Rewrite `src/query.ts`'s session-driving logic**

Replace the body of `app.connectWith(...)` and extend the returned object:

```ts
import { client, methods, type McpServer } from "@agentclientprotocol/sdk";
import { spawnGrokAgentStdio } from "./transport.js";
import { buildPermissionHandler, type CanUseTool } from "./permissions.js";
import { toSDKMessages, type SDKMessage } from "./messages.js";
import { AgentProcessError } from "./errors.js";
import { AsyncQueue } from "./internal/asyncQueue.js";

export interface QueryOptions {
  cwd?: string;
  grokBin?: string;
  model?: string;
  canUseTool?: CanUseTool;
  mcpServers?: McpServer[];
  /**
   * "default" respects `canUseTool` (or auto-allows if omitted, per
   * buildPermissionHandler's own default). "bypassPermissions" ignores
   * `canUseTool` entirely and auto-allows every tool call, matching `--yolo`.
   * The spec's "acceptEdits"/"plan" modes are deferred past v1 — they need
   * policy semantics (which tool kinds count as "edits", plan-mode gating)
   * this SDK doesn't model yet.
   */
  permissionMode?: "default" | "bypassPermissions";
  /** Aborting stops routing further messages and best-effort kills the child process. */
  abortSignal?: AbortSignal;
}

export interface Query extends AsyncIterable<SDKMessage> {
  readonly sessionId: Promise<string>;
  send(text: string): void;
  interrupt(): Promise<void>;
}

export function query(input: { prompt: string; options?: QueryOptions }): Query {
  const options = input.options ?? {};
  const queue = new AsyncQueue<SDKMessage>();
  const inbox = new AsyncQueue<string>();
  inbox.push(input.prompt);

  let resolveSessionId!: (id: string) => void;
  const sessionId = new Promise<string>((resolve) => {
    resolveSessionId = resolve;
  });

  let activeSession: { cancel?: () => Promise<void> } | undefined;
  let closed = false;

  // Named agentProcess, not process — a local `process` binding would shadow
  // Node's global `process` (used below as `process.cwd()`).
  const agentProcess = spawnGrokAgentStdio({
    grokBin: options.grokBin,
    cwd: options.cwd,
    args: options.model ? ["--model", options.model] : undefined,
  });

  agentProcess.exited.then(({ code, stderrTail }) => {
    if (!closed) {
      queue.fail(new AgentProcessError(code, stderrTail));
    }
  });

  if (options.abortSignal) {
    const onAbort = () => {
      closed = true;
      queue.close();
      void activeSession?.cancel?.();
      agentProcess.kill();
    };
    if (options.abortSignal.aborted) onAbort();
    else options.abortSignal.addEventListener("abort", onAbort, { once: true });
  }

  const effectiveCanUseTool =
    options.permissionMode === "bypassPermissions" ? undefined : options.canUseTool;

  const app = client({ name: "grok-agent-sdk" }).onRequest(
    methods.client.session.requestPermission,
    buildPermissionHandler(effectiveCanUseTool),
  );

  app
    .connectWith(agentProcess.stream, async (ctx: any) => {
      const builder = ctx.buildSession({
        cwd: options.cwd ?? process.cwd(),
        mcpServers: options.mcpServers ?? [],
      });

      await builder.withSession(async (session: any) => {
        resolveSessionId(session.sessionId);
        activeSession = {
          // session/cancel is a notification in the ACP schema (AgentNotificationHandlersByMethod),
          // not a request — using ctx.request() here would hang waiting for a response that
          // never arrives, since the agent dispatches it as fire-and-forget.
          cancel: () => ctx.notify(methods.agent.session.cancel, { sessionId: session.sessionId }),
        };

        for await (const promptText of inbox) {
          const promptPromise = session.prompt(promptText);

          for (;;) {
            const next = await session.nextUpdate();
            for (const message of toSDKMessages(next)) {
              queue.push(message);
            }
            if (next.kind === "stop") break;
          }

          await promptPromise;
        }
      });
    })
    .catch((error: unknown) => {
      queue.fail(error);
    })
    .finally(() => {
      closed = true;
      queue.close();
      agentProcess.kill();
    });

  return {
    sessionId,
    send(text: string): void {
      inbox.push(text);
    },
    async interrupt(): Promise<void> {
      await activeSession?.cancel?.();
    },
    [Symbol.asyncIterator]: () => queue[Symbol.asyncIterator](),
  };
}
```

Note: the `inbox` queue never calls `.close()` in this implementation — the session loop's `for await (const promptText of inbox)` runs until the outer process/session ends (driven by `agentProcess.exited` failing the outbound `queue`, which the caller's `for await` loop will surface as a thrown `AgentProcessError`, ending consumption). This matches Claude Agent SDK's own interactive-mode lifecycle, where the caller ends the conversation by stopping iteration, not by an explicit "close" call from inside the loop.

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/query.test.ts
```

Expected: all `query()` tests pass (6 total across both `describe` blocks: 1 from Task 7's block plus 5 in this task's block).

- [ ] **Step 5: Commit**

```bash
git add src/query.ts test/query.test.ts
git commit -m "feat: stream intermediate session updates; support multi-turn send()/interrupt(), permissionMode, and abortSignal"
```

---

### Task 9: Hooks engine

**Files:**
- Create: `src/hooks.ts`
- Modify: `src/query.ts`
- Test: `test/hooks.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export type HookEvent = "PreToolUse" | "PostToolUse";
  export interface HookPayload {
    event: HookEvent;
    toolCallId: string;
    title?: string;
    status: string;
  }
  export type HookCallback = (payload: HookPayload) => void | Promise<void>;
  export type Hooks = Partial<Record<HookEvent, HookCallback[]>>;
  export function dispatchHooks(hooks: Hooks | undefined, message: SDKMessage): Promise<void>;
  ```
- Consumes: `SDKMessage` from Task 2.
- Modifies `query()` (Task 8) to accept `options.hooks?: Hooks` and call `dispatchHooks` for every message pushed to the outbound queue, matching on `tool_call` (status `pending`/`in_progress` → `PreToolUse`) and `tool_call_update` with a terminal status (`completed`/`failed` → `PostToolUse`).

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, vi } from "vitest";
import { dispatchHooks } from "../src/hooks.js";
import type { SDKMessage } from "../src/messages.js";

describe("dispatchHooks", () => {
  it("calls PreToolUse callbacks for a pending tool_call message", async () => {
    const preToolUse = vi.fn();
    const message: SDKMessage = {
      type: "tool_call",
      toolCallId: "call_1",
      title: "Reading file",
      kind: "read",
      status: "pending",
      raw: {} as any,
    };

    await dispatchHooks({ PreToolUse: [preToolUse] }, message);

    expect(preToolUse).toHaveBeenCalledWith({
      event: "PreToolUse",
      toolCallId: "call_1",
      title: "Reading file",
      status: "pending",
    });
  });

  it("calls PostToolUse callbacks only for terminal tool_call_update statuses", async () => {
    const postToolUse = vi.fn();
    const hooks = { PostToolUse: [postToolUse] };

    await dispatchHooks(hooks, {
      type: "tool_call_update",
      toolCallId: "call_1",
      status: "in_progress",
      raw: {} as any,
    });
    expect(postToolUse).not.toHaveBeenCalled();

    await dispatchHooks(hooks, {
      type: "tool_call_update",
      toolCallId: "call_1",
      status: "completed",
      raw: {} as any,
    });
    expect(postToolUse).toHaveBeenCalledWith({
      event: "PostToolUse",
      toolCallId: "call_1",
      status: "completed",
    });
  });

  it("does nothing when hooks is undefined or the message has no matching event", async () => {
    await expect(
      dispatchHooks(undefined, { type: "text_delta", text: "hi" }),
    ).resolves.toBeUndefined();

    await expect(
      dispatchHooks({}, { type: "text_delta", text: "hi" }),
    ).resolves.toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/hooks.test.ts
```

Expected: FAIL — `Cannot find module '../src/hooks.js'`.

- [ ] **Step 3: Write the implementation**

```ts
import type { SDKMessage } from "./messages.js";

export type HookEvent = "PreToolUse" | "PostToolUse";

export interface HookPayload {
  event: HookEvent;
  toolCallId: string;
  title?: string;
  status: string;
}

export type HookCallback = (payload: HookPayload) => void | Promise<void>;

export type Hooks = Partial<Record<HookEvent, HookCallback[]>>;

const TERMINAL_STATUSES = new Set(["completed", "failed"]);

export async function dispatchHooks(hooks: Hooks | undefined, message: SDKMessage): Promise<void> {
  if (!hooks) return;

  if (message.type === "tool_call") {
    const callbacks = hooks.PreToolUse;
    if (!callbacks?.length) return;
    const payload: HookPayload = {
      event: "PreToolUse",
      toolCallId: message.toolCallId,
      title: message.title,
      status: message.status,
    };
    for (const callback of callbacks) await callback(payload);
    return;
  }

  if (message.type === "tool_call_update" && TERMINAL_STATUSES.has(message.status)) {
    const callbacks = hooks.PostToolUse;
    if (!callbacks?.length) return;
    const payload: HookPayload = {
      event: "PostToolUse",
      toolCallId: message.toolCallId,
      status: message.status,
    };
    for (const callback of callbacks) await callback(payload);
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/hooks.test.ts
```

Expected: 3 passed.

- [ ] **Step 5: Wire hooks into `query()`**

In `src/query.ts`, add `hooks?: Hooks` to `QueryOptions`, import `dispatchHooks`, and change the `queue.push(message)` call site inside the `for (;;) { const next = await session.nextUpdate(); ... }` loop (the one inside the `for await (const promptText of inbox)` loop from Task 8) to:

```ts
for (const message of toSDKMessages(next)) {
  await dispatchHooks(options.hooks, message);
  queue.push(message);
}
```

- [ ] **Step 6: Run the full test suite to confirm no regressions**

```bash
pnpm test
```

Expected: all tests still passing (query.test.ts included, since `options.hooks` is optional and defaults to a no-op).

- [ ] **Step 7: Commit**

```bash
git add src/hooks.ts src/query.ts test/hooks.test.ts
git commit -m "feat: add PreToolUse/PostToolUse hook dispatch and wire into query()"
```

---

### Task 10: `tool()` helper for custom tools

**Files:**
- Create: `src/mcp/tool.ts`
- Test: `test/mcp/tool.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface ToolDefinition<Args extends Record<string, z.ZodType>> {
    name: string;
    description: string;
    inputSchema: Args;
    handler: (args: { [K in keyof Args]: z.infer<Args[K]> }) => Promise<{ content: Array<{ type: "text"; text: string }> }>;
  }
  export function tool<Args extends Record<string, z.ZodType>>(
    name: string,
    description: string,
    inputSchema: Args,
    handler: ToolDefinition<Args>["handler"],
  ): ToolDefinition<Args>;
  ```
  Consumed by Task 11's `createSdkMcpServer`.

  `Record<string, z.ZodType>` and the mapped-type inference, not zod v3's `ZodRawShape`/`objectOutputType` — verified against the real `zod@4.4.3` package: zod v4's default export (`import { z } from "zod"`) dropped both names (they now live only under the legacy `zod/v3` subpath). `z.ZodType` and `z.infer` remain in the default export and are the stable, non-version-specific way to express this. This shape is also structurally compatible with what `@modelcontextprotocol/sdk@1.29.0`'s `registerTool` expects (its own `ZodRawShapeCompat` is `Record<string, AnySchema>`), which is why Task 11 can pass `definition.inputSchema` straight through with no cast.

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import { z } from "zod";
import { tool } from "../../src/mcp/tool.js";

describe("tool()", () => {
  it("returns a ToolDefinition with the given name, schema, and handler", async () => {
    const weatherTool = tool(
      "get_weather",
      "Get current weather for a city",
      { city: z.string() },
      async ({ city }) => ({ content: [{ type: "text", text: `Sunny in ${city}` }] }),
    );

    expect(weatherTool.name).toBe("get_weather");
    expect(weatherTool.description).toBe("Get current weather for a city");
    const result = await weatherTool.handler({ city: "Lisbon" });
    expect(result).toEqual({ content: [{ type: "text", text: "Sunny in Lisbon" }] });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/mcp/tool.test.ts
```

Expected: FAIL — `Cannot find module '../../src/mcp/tool.js'`.

- [ ] **Step 3: Write the implementation**

```ts
import type { z } from "zod";

export interface ToolResult {
  content: Array<{ type: "text"; text: string }>;
}

export interface ToolDefinition<Args extends Record<string, z.ZodType>> {
  name: string;
  description: string;
  inputSchema: Args;
  handler: (args: { [K in keyof Args]: z.infer<Args[K]> }) => Promise<ToolResult>;
}

export function tool<Args extends Record<string, z.ZodType>>(
  name: string,
  description: string,
  inputSchema: Args,
  handler: ToolDefinition<Args>["handler"],
): ToolDefinition<Args> {
  return { name, description, inputSchema, handler };
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/mcp/tool.test.ts
```

Expected: 1 passed.

- [ ] **Step 5: Commit**

```bash
git add src/mcp/tool.ts test/mcp/tool.test.ts
git commit -m "feat: add tool() helper for defining custom SDK tools"
```

---

### Task 11: `createSdkMcpServer` — loopback MCP server for custom tools (runtime-agnostic transport)

**Files:**
- Create: `src/mcp/sdkServer.ts`
- Test: `test/mcp/sdkServer.test.ts`

**Interfaces:**
- Consumes: `ToolDefinition` (Task 10), `McpServer` (http variant) type from `@agentclientprotocol/sdk`.
- Produces:
  ```ts
  export interface SdkMcpServerHandle {
    config: McpServer;         // { type: "http", name, url, headers } — pass into query()'s options.mcpServers
    close(): Promise<void>;
  }
  export async function createSdkMcpServer(options: {
    name: string;
    tools: ToolDefinition<any>[];
  }): Promise<SdkMcpServerHandle>;
  ```

Use `WebStandardStreamableHTTPServerTransport` (from `@modelcontextprotocol/sdk/server/webStandardStreamableHttp.js`), not `StreamableHTTPServerTransport`/`NodeStreamableHTTPServerTransport`. The Web Standard variant speaks plain `Request`/`Response` and, per its own docstring, "works on any runtime that supports Web Standards: Node.js 18+, Cloudflare Workers, Deno, Bun, etc." — verified by re-inspecting the installed `@modelcontextprotocol/sdk@1.29.0` package. The Node-only wrapper exists solely for people already wired into `node:http`/Express; picking it here would undo the portability the MCP SDK's own maintainers built this transport to provide (see the SDK's history: `StdioServerTransport` broke on Deno over a `Buffer` global dependency, and the original HTTP transport broke on Workers/Deno/Bun over Express — both fixed by moving the core transport onto Web Standards in the SDK's "V2").

Because we still host on `node:http` in this task (matching the rest of the package's Node-primary test environment), a small bridge function converts Node's `IncomingMessage`/`ServerResponse` to/from Web `Request`/`Response` — this bridge is the *only* Node-specific code in this file. On Bun or Deno, their native `Bun.serve()`/`Deno.serve()` already speak `Request`/`Response` directly and could call `transport.handleRequest(request)` with no bridge at all; that swap is not implemented in this task (custom tools are a less central path than the core `query()` loop this plan already made portable in Task 3), but the seam is exactly this one bridge function, same pattern as Task 3's runtime dispatch.

This is the task with the highest real-network-behavior risk, so the test spins up the real server on an ephemeral loopback port and drives it with a plain `fetch` MCP `tools/call` request rather than mocking the transport — the thing worth verifying is that the token check and the tool dispatch actually work end-to-end.

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, afterEach } from "vitest";
import { z } from "zod";
import { tool } from "../../src/mcp/tool.js";
import { createSdkMcpServer, type SdkMcpServerHandle } from "../../src/mcp/sdkServer.js";

let handle: SdkMcpServerHandle | undefined;

afterEach(async () => {
  await handle?.close();
  handle = undefined;
});

describe("createSdkMcpServer", () => {
  it("binds to 127.0.0.1 and returns an http McpServer config with a bearer token header", async () => {
    handle = await createSdkMcpServer({
      name: "my-tools",
      tools: [tool("echo", "Echo input", { text: z.string() }, async ({ text }) => ({
        content: [{ type: "text", text }],
      }))],
    });

    expect(handle.config.type).toBe("http");
    expect(handle.config.name).toBe("my-tools");
    expect(new URL(handle.config.url).hostname).toBe("127.0.0.1");
    const authHeader = handle.config.headers.find((h) => h.name === "Authorization");
    expect(authHeader?.value).toMatch(/^Bearer .+/);
  });

  it("rejects requests without the correct bearer token", async () => {
    handle = await createSdkMcpServer({ name: "my-tools", tools: [] });

    const response = await fetch(handle.config.url, {
      method: "POST",
      headers: { "content-type": "application/json", accept: "application/json, text/event-stream" },
      body: JSON.stringify({ jsonrpc: "2.0", id: 1, method: "tools/list", params: {} }),
    });

    expect(response.status).toBe(401);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/mcp/sdkServer.test.ts
```

Expected: FAIL — `Cannot find module '../../src/mcp/sdkServer.js'`.

- [ ] **Step 3: Write the implementation**

```ts
import { createServer, type IncomingMessage, type ServerResponse } from "node:http";
import { randomBytes } from "node:crypto";
import { Readable } from "node:stream";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { WebStandardStreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/webStandardStreamableHttp.js";
import type { McpServer as AcpMcpServerConfig } from "@agentclientprotocol/sdk";
import type { ToolDefinition } from "./tool.js";

export interface SdkMcpServerHandle {
  config: AcpMcpServerConfig;
  close(): Promise<void>;
}

// The only Node-specific code in this file: converts Node's IncomingMessage into
// the Web-standard Request that WebStandardStreamableHTTPServerTransport expects.
// `duplex: "half"` is required by Node's fetch implementation whenever a streaming
// body is attached to a Request.
function nodeRequestToWebRequest(req: IncomingMessage): Request {
  const url = `http://127.0.0.1${req.url ?? "/"}`;
  const headers = new Headers();
  for (const [key, value] of Object.entries(req.headers)) {
    if (value === undefined) continue;
    headers.set(key, Array.isArray(value) ? value.join(", ") : value);
  }
  const hasBody = req.method !== "GET" && req.method !== "HEAD";
  return new Request(url, {
    method: req.method,
    headers,
    body: hasBody ? (Readable.toWeb(req) as unknown as ReadableStream<Uint8Array>) : undefined,
    duplex: hasBody ? "half" : undefined,
  } as RequestInit);
}

// The other half of the same bridge: writes a Web-standard Response back through
// Node's ServerResponse.
async function writeWebResponseToNode(response: Response, res: ServerResponse): Promise<void> {
  res.writeHead(response.status, Object.fromEntries(response.headers));
  if (!response.body) {
    res.end();
    return;
  }
  const reader = response.body.getReader();
  for (;;) {
    const { done, value } = await reader.read();
    if (done) break;
    res.write(value);
  }
  res.end();
}

export async function createSdkMcpServer(options: {
  name: string;
  tools: ToolDefinition<any>[];
}): Promise<SdkMcpServerHandle> {
  const token = randomBytes(24).toString("hex");

  const mcpServer = new McpServer({ name: options.name, version: "0.1.0" });
  for (const definition of options.tools) {
    mcpServer.registerTool(
      definition.name,
      { description: definition.description, inputSchema: definition.inputSchema },
      async (args: unknown) => definition.handler(args as never),
    );
  }

  // WebStandardStreamableHTTPServerTransport itself has no Node dependency — on Bun
  // or Deno, their native serve() functions could call transport.handleRequest(request)
  // directly with no bridge at all. Only the node:http hosting below is Node-specific.
  const transport = new WebStandardStreamableHTTPServerTransport({ sessionIdGenerator: undefined });
  await mcpServer.connect(transport);

  const httpServer = createServer((req: IncomingMessage, res: ServerResponse) => {
    const authHeader = req.headers.authorization;
    if (authHeader !== `Bearer ${token}`) {
      res.writeHead(401, { "content-type": "application/json" }).end(
        JSON.stringify({ error: "unauthorized" }),
      );
      return;
    }
    const request = nodeRequestToWebRequest(req);
    void transport
      .handleRequest(request)
      .then((response) => writeWebResponseToNode(response, res));
  });

  await new Promise<void>((resolve) => httpServer.listen(0, "127.0.0.1", resolve));
  const address = httpServer.address();
  if (address === null || typeof address === "string") {
    throw new Error("failed to determine loopback server port");
  }
  const url = `http://127.0.0.1:${address.port}/mcp`;

  return {
    config: {
      type: "http",
      name: options.name,
      url,
      headers: [{ name: "Authorization", value: `Bearer ${token}` }],
    },
    async close(): Promise<void> {
      await new Promise<void>((resolve, reject) => {
        httpServer.close((error) => (error ? reject(error) : resolve()));
      });
      await mcpServer.close();
    },
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/mcp/sdkServer.test.ts
```

Expected: 2 passed.

- [ ] **Step 5: Commit**

```bash
git add src/mcp/sdkServer.ts test/mcp/sdkServer.test.ts
git commit -m "feat: add createSdkMcpServer - loopback MCP server on a runtime-agnostic Web Standard transport"
```

---

### Task 12: Session resume and fork wrappers

**Files:**
- Create: `src/session.ts`
- Modify: `src/query.ts`
- Test: `test/session.test.ts`

**Interfaces:**
- Produces: `QueryOptions.resume?: string` and `QueryOptions.forkFrom?: string`, handled inside `query()`'s `ctx.buildSession(...)` call — `resume` calls `ctx.loadSession(...)`-equivalent via `SessionBuilder`, `forkFrom` uses `ctx.request(methods.agent.session.fork, ...)`.

Since `SessionBuilder`/`ClientContext.buildSession()` only covers `session/new`, resuming and forking go through `ctx.request(...)` directly with the literal method names verified against `@agentclientprotocol/sdk`'s `methods.agent.session.load` (`"session/load"`) and `methods.agent.session.fork` (`"session/fork"`).

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, vi } from "vitest";
import { resumeSession, forkSession } from "../src/session.js";

describe("resumeSession", () => {
  it("calls session/load with the given session id and cwd", async () => {
    const request = vi.fn().mockResolvedValue({ sessionId: "sess-1" });
    const ctx = { request };

    const result = await resumeSession(ctx as any, { sessionId: "sess-1", cwd: "/tmp" });

    expect(request).toHaveBeenCalledWith("session/load", { sessionId: "sess-1", cwd: "/tmp" });
    expect(result).toEqual({ sessionId: "sess-1" });
  });
});

describe("forkSession", () => {
  it("calls session/fork with the source session id", async () => {
    const request = vi.fn().mockResolvedValue({ sessionId: "sess-2" });
    const ctx = { request };

    const result = await forkSession(ctx as any, { sessionId: "sess-1" });

    expect(request).toHaveBeenCalledWith("session/fork", { sessionId: "sess-1" });
    expect(result).toEqual({ sessionId: "sess-2" });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/session.test.ts
```

Expected: FAIL — `Cannot find module '../src/session.js'`.

- [ ] **Step 3: Write the implementation**

```ts
import type { ClientContext, LoadSessionResponse, ForkSessionResponse } from "@agentclientprotocol/sdk";

export async function resumeSession(
  ctx: Pick<ClientContext, "request">,
  params: { sessionId: string; cwd: string },
): Promise<LoadSessionResponse> {
  return ctx.request("session/load", params) as Promise<LoadSessionResponse>;
}

export async function forkSession(
  ctx: Pick<ClientContext, "request">,
  params: { sessionId: string },
): Promise<ForkSessionResponse> {
  return ctx.request("session/fork", params) as Promise<ForkSessionResponse>;
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/session.test.ts
```

Expected: 2 passed.

- [ ] **Step 5: Wire `resume` into `query()`**

In `src/query.ts`, add to `QueryOptions`:

```ts
resume?: string;
```

And change the `builder` construction inside `connectWith`'s callback from always calling `ctx.buildSession({ cwd, mcpServers })` to:

```ts
if (options.resume) {
  await resumeSession(ctx, { sessionId: options.resume, cwd: options.cwd ?? process.cwd() });
}
const builder = ctx.buildSession({
  cwd: options.cwd ?? process.cwd(),
  mcpServers: options.mcpServers ?? [],
});
```

(`session/load` restores server-side history for that session ID; the subsequent `session/new` on the same `cwd` is how this SDK's `ActiveSession` plumbing attaches update routing — matching the pattern documented in `15-agent-mode.md`'s session lifecycle. `forkSession` is exposed as a standalone function for callers who want to fork without immediately starting a `query()`, e.g. to branch a conversation for a summary — it is not wired into the default `query()` path in this task, since forking mid-query has no natural single entry point; document this as a v1.1 follow-up in the SDK's README, Task 15.)

- [ ] **Step 6: Run the full test suite to confirm no regressions**

```bash
pnpm test
```

Expected: all tests still passing.

- [ ] **Step 7: Commit**

```bash
git add src/session.ts src/query.ts test/session.test.ts
git commit -m "feat: add session resume/fork wrappers, wire resume into query()"
```

---

### Task 13: Public API surface

**Files:**
- Modify: `src/index.ts`
- Test: `test/index.test.ts`

**Interfaces:**
- Produces the package's full public surface: `query`, `Query`, `QueryOptions`, `SDKMessage`, `CanUseTool`, `PermissionDecision`, `Hooks`, `HookEvent`, `HookPayload`, `tool`, `ToolDefinition`, `createSdkMcpServer`, `SdkMcpServerHandle`, `resumeSession`, `forkSession`, `AgentProcessError`.

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import * as sdk from "../src/index.js";

describe("public API surface", () => {
  it("exports the core query API", () => {
    expect(typeof sdk.query).toBe("function");
  });

  it("exports the custom-tool helpers", () => {
    expect(typeof sdk.tool).toBe("function");
    expect(typeof sdk.createSdkMcpServer).toBe("function");
  });

  it("exports session helpers and the process error type", () => {
    expect(typeof sdk.resumeSession).toBe("function");
    expect(typeof sdk.forkSession).toBe("function");
    expect(sdk.AgentProcessError).toBeTypeOf("function");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
pnpm test -- test/index.test.ts
```

Expected: FAIL — `sdk.query` is `undefined` (current `src/index.ts` only exports `VERSION`).

- [ ] **Step 3: Rewrite `src/index.ts`**

```ts
export const VERSION = "0.1.0";

export { query, type Query, type QueryOptions } from "./query.js";
export type { SDKMessage } from "./messages.js";
export type { CanUseTool, PermissionDecision } from "./permissions.js";
export type { Hooks, HookEvent, HookPayload, HookCallback } from "./hooks.js";
export { tool, type ToolDefinition, type ToolResult } from "./mcp/tool.js";
export { createSdkMcpServer, type SdkMcpServerHandle } from "./mcp/sdkServer.js";
export { resumeSession, forkSession } from "./session.js";
export { AgentProcessError } from "./errors.js";
```

- [ ] **Step 4: Run test to verify it passes**

```bash
pnpm test -- test/index.test.ts
```

Expected: 3 passed.

- [ ] **Step 5: Run the full suite and the build**

```bash
pnpm test
pnpm build
```

Expected: all tests pass; `pnpm build` completes with no TypeScript errors and populates `dist/`.

- [ ] **Step 6: Commit**

```bash
git add src/index.ts test/index.test.ts
git commit -m "feat: finalize public API surface in src/index.ts"
```

---

### Task 14: Gated integration test against a real `grok` binary

**Files:**
- Create: `test/integration/query.integration.test.ts`
- Modify: `vitest.config.ts`

**Interfaces:**
- No new production code. Exercises `query()` (Task 8) end-to-end against a real `grok agent stdio` process.

This test is skipped by default because no `grok` binary is guaranteed to be on `PATH` in every environment (it wasn't on this machine at plan-writing time). It only runs when `GROK_AGENT_SDK_TEST_BIN` is set to a real binary path.

- [ ] **Step 1: Write the test**

```ts
import { describe, it, expect } from "vitest";
import { query } from "../../src/query.js";

const grokBin = process.env.GROK_AGENT_SDK_TEST_BIN;

describe.skipIf(!grokBin)("query() integration (real grok binary)", () => {
  it("completes a simple prompt and reports a result message", async () => {
    const q = query({
      prompt: "Reply with exactly the word: pong",
      options: { grokBin, cwd: process.cwd() },
    });

    const messages = [];
    for await (const message of q) {
      messages.push(message);
    }

    const result = messages.find((m) => m.type === "result");
    expect(result).toBeDefined();
    expect(result?.stopReason).toBe("end_turn");
  }, 60_000);
});
```

- [ ] **Step 2: Confirm the test is skipped without the env var**

```bash
pnpm test -- test/integration/query.integration.test.ts
```

Expected: `1 skipped` (no `GROK_AGENT_SDK_TEST_BIN` set in this environment — this is the expected default outcome here; do not attempt to install or authenticate a `grok` binary as part of this plan).

- [ ] **Step 3: If a `grok` binary and valid auth are available, run it for real**

```bash
GROK_AGENT_SDK_TEST_BIN=$(which grok) pnpm test -- test/integration/query.integration.test.ts
```

Expected (only when `grok` is installed and authenticated): 1 passed.

- [ ] **Step 4: Commit**

```bash
git add test/integration/query.integration.test.ts
git commit -m "test: add gated integration test against a real grok binary"
```

---

### Task 15: README and example

**Files:**
- Create: `README.md`
- Create: `examples/basic-query.ts`

**Interfaces:**
- No production code changes. Documents the package for consumers.

- [ ] **Step 1: Write `examples/basic-query.ts`**

```ts
import { query } from "../src/index.js";

async function main() {
  const q = query({
    prompt: "List the files in this directory.",
    options: {
      cwd: process.cwd(),
      canUseTool: async () => ({ behavior: "allow" }),
    },
  });

  for await (const message of q) {
    if (message.type === "text_delta") {
      process.stdout.write(message.text);
    } else if (message.type === "result") {
      console.log(`\n\n[stopped: ${message.stopReason}]`);
    }
  }
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

- [ ] **Step 2: Write `README.md`**

```markdown
# grok-agent-sdk

Programmatic harness SDK for [Grok Build](https://x.ai/cli) — spawn, stream,
and control the `grok` coding agent from TypeScript, built on the
[Agent Client Protocol](https://agentclientprotocol.com) (`grok agent stdio`).

This is a harness SDK, not an API client: every call goes through the real
agent process (the same binary and tool-execution runtime the Grok Build TUI
uses), not a raw model API.

## Install

\`\`\`bash
pnpm add grok-agent-sdk
\`\`\`

Requires the `grok` CLI on `PATH` (or pass `grokBin` explicitly), already
authenticated (`XAI_API_KEY` or a cached `grok login` session).

## Quick start

\`\`\`ts
import { query } from "grok-agent-sdk";

const q = query({
  prompt: "List the files in this directory.",
  options: { cwd: process.cwd(), canUseTool: async () => ({ behavior: "allow" }) },
});

for await (const message of q) {
  if (message.type === "text_delta") process.stdout.write(message.text);
}
\`\`\`

## Runtime support

Works on Node (>=22), Bun, and self-hosted Deno — anywhere with real OS
process access. It does **not** work on Deno Deploy or Cloudflare Workers:
`query()` always spawns \`grok agent stdio\` as a real subprocess, and both
of those platforms are isolate/edge sandboxes that forbid subprocess
spawning outright — this is a hard platform limit, not a missing permission.

On Deno specifically (unlike Node/Bun, which have no equivalent sandbox),
pass at least:

\`\`\`bash
deno run --allow-run --allow-env your-script.ts
\`\`\`

\`--allow-run\` lets Deno spawn the \`grok\` binary; \`--allow-env\` is only
needed if your own code reads \`process.env\`/\`Deno.env\` (\`process.cwd()\`
itself needs no permission flag).

## Design

See [the design spec](https://github.com/renan/grok-build/blob/main/docs/superpowers/specs/2026-07-17-grok-agent-sdk-design.md)
in the `grok-build` repo for the full architecture rationale, including why
this is built on ACP (`grok agent stdio`) rather than headless mode.

## Status

v1 (this repo's first implementation pass) covers: `query()` streaming,
multi-turn `send()`/`interrupt()`, `canUseTool` permission callbacks,
`PreToolUse`/`PostToolUse` hooks, custom tools via `createSdkMcpServer`, and
session resume. Not yet covered: session fork wired into `query()`, remote
(`grok agent serve`) transport, Python/Rust bindings — see the design spec's
"Future work" section.
\`\`\`
```

- [ ] **Step 3: Run the full test suite one last time**

```bash
pnpm test
pnpm typecheck
pnpm lint
pnpm build
```

Expected: all tests pass, no type errors, no lint errors, build succeeds.

- [ ] **Step 4: Validate the package is actually publishable**

`publint` and `@arethetypeswrong/cli` were installed in Task 1 but had nothing meaningful to check until now — this is the first point where `dist/` reflects the real public API surface (Task 13's `src/index.ts`) instead of the placeholder `VERSION` export.

```bash
pnpm pack:check
```

Expected: `publint` reports no errors (the `exports` map in `package.json` matches what `tsc` actually emitted into `dist/`), and `attw --pack .` reports no "false ESM"/resolution problems for the `node16`/`bundler` resolution modes it checks by default. If either tool reports a real issue, fix the `exports` field or the emitted `dist/` shape (not the checker) before committing.

- [ ] **Step 5: Commit**

```bash
git add README.md examples/basic-query.ts
git commit -m "docs: add README and basic query() example"
```

---

## Post-plan checklist (not a task — verify manually)

- [ ] `pnpm test` passes with zero failures and zero unexpected skips (the integration test skip is expected).
- [ ] `pnpm build` produces `dist/index.js` and `dist/index.d.ts`.
- [ ] `git log --oneline` shows one commit per task, in order.
- [ ] If you have a real `grok` binary available, run Task 14's integration test against it at least once before considering v1 "done."
