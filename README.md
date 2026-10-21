# Mu

Mu is a TypeScript CLI coding agent. You point it at a directory and give it a task in plain English; it reads files, searches code, runs shell commands, and streams a response back — all while enforcing approval gates, managing context, and delegating subtasks to specialised subagents.

> **Reference implementation.** Mu is designed as a production-oriented reference for developers building their own coding agents. The architecture prioritises the patterns that matter at scale: a clean sandbox abstraction that swaps execution backends without touching tool logic, a discriminated-union approval system that is safe by default and auditable by design, explicit context management to keep long sessions from degrading, and a composable tool registry with wrapping hooks for observability. Use Mu as a starting point or as a blueprint — every design decision here is something you will eventually face in a real agent harness.

## Table of Contents

- [Requirements](#requirements)
- [Setup](#setup)
- [Usage](#usage)
- [Architecture Overview](#architecture-overview)
- [Entry Point — `index.ts`](#entry-point--indexts)
- [Sandbox Abstraction — `src/sandbox.ts`](#sandbox-abstraction--srcsandboxts)
  - [Local Sandbox](#local-sandbox--srcsandbox-localts)
  - [Just-Bash Sandbox](#just-bash-sandbox--srcsandbox-just-bashts)
  - [Sandbox Lifecycle](#sandbox-lifecycle)
- [Tools — `src/tools.ts`](#tools--srctoolsts)
  - [read](#read)
  - [grep](#grep)
  - [bash](#bash)
  - [askUser](#askuser)
  - [todo](#todo)
  - [loadSkill](#loadskill)
  - [task — subagent delegation](#task--subagent-delegation)
- [Approval Gates — `src/approval.ts`](#approval-gates--srcapprovalts)
- [Tool Registry — `src/registry.ts`](#tool-registry--srcregistryts)
- [System Prompt — `src/system.ts`](#system-prompt--srcsystemts)
- [Context Management — `src/cache.ts`](#context-management--srccachets)
- [Verification Gates — `src/verification.ts`](#verification-gates--srcverificationts)
- [Skills — `src/skills.ts`](#skills--srcskillsts)
- [Project Structure](#project-structure)
- [Running Tests](#running-tests)
- [Design Decisions](#design-decisions)

## Requirements

- [Bun](https://bun.sh/) 1.x — used for running, testing, and type-checking
- An [OpenAI API key](https://platform.openai.com/api_keys) with available credits
- Node.js 20.18.1+ — required only for `--sandbox=just-bash` (Bun hands that mode off to Node automatically)

## Setup

```bash
bun install
cp -n .env.example .env
# Add OPENAI_API_KEY=<your key> to .env — never commit this file
```

## Usage

```bash
# Run the agent against a directory with a task
bun run start <directory> "<task>"

# Examples
bun run start . "What does the approval system do?"
bun run start ./my-project "Add error handling to the fetch call in src/api.ts"

# Use the in-memory sandbox (writes never touch disk)
bun run index.ts --sandbox=just-bash . "Create a file called scratch.txt with hello"

# Use a different model (default is gpt-5.5)
bun run index.ts --model=gpt-5-mini . "Refactor the auth module"

# Type-check the project
bun run typecheck
```

**Output routing:**
- Agent text responses → `stdout`
- Tool calls, step telemetry, and errors → `stderr`

You can pipe agent output cleanly: `bun run start . "summarise the API" > summary.txt`

**AGENTS.md convention:** if the target directory contains an `AGENTS.md` file, its contents are injected into the system prompt as project-specific instructions. Use this to give the agent standing rules about a codebase — preferred patterns, forbidden operations, test commands, etc.

## Architecture Overview

```
index.ts                     ← CLI entry point; wires everything together
│
├── src/sandbox.ts           ← Sandbox + SandboxLifecycle interfaces
├── src/sandbox-local.ts     ← Local filesystem backend (Node fs + execSync)
├── src/sandbox-just-bash.ts ← In-memory copy-on-write backend (just-bash)
│
├── src/tools.ts             ← All tool factory functions
├── src/approval.ts          ← Approval gate logic
├── src/registry.ts          ← Tool registry, registerBuiltins, wrapTool
│
├── src/system.ts            ← buildSystemPrompt — agent instruction builder
├── src/cache.ts             ← addCacheControl — prompt cache markers
├── src/verification.ts      ← discoverGates — auto-detect verification commands
├── src/skills.ts            ← discoverSkills — on-demand skill file loader
│
└── skills/                  ← Skill definitions (each a directory with SKILL.md)
    └── auth-patterns/
        └── SKILL.md
```

The agent loop is provided by `ToolLoopAgent` from the `ai` package (AI SDK v6). Mu's job is to configure and compose the pieces around it.

## Entry Point — `index.ts`

`index.ts` is the full startup sequence. On every invocation it:

**1. Parses CLI flags**
`--sandbox` (default: `local`) and `--model` (default: `gpt-5.5`), plus positional args `<directory>` and `<task>`.

**2. Handles the just-bash handoff**
The just-bash sandbox requires Node, not Bun. When `--sandbox=just-bash` is detected inside a Bun process, the entry point re-spawns itself under `node --import tsx` and forwards all arguments, then exits. This is transparent to the caller.

**3. Reads AGENTS.md**
Checks for `AGENTS.md` in the target directory and reads it as `projectContext`.

**4. Creates the sandbox**
Calls `sandboxFromFlag` to produce either a `local` or `just-bash` `Sandbox`.

**5. Registers shutdown handlers**
`SandboxLifecycle.beforeStop` and `sandbox.stop()` are called in a `try/finally` and also wired to `SIGINT`, so sandbox cleanup always happens even on crash or Ctrl-C.

**6. Discovers gates and skills**
`discoverGates` reads `package.json` to find verification commands. `discoverSkills` scans `<cwd>/skills` and `~/.harness/skills` for skill directories.

**7. Builds the tool registry**
`createRegistry()` + `registerBuiltins()` wires up all standard tools. A `now` tool is added inline as an example of registering a one-off custom tool.

**8. Builds the system prompt**
`buildSystemPrompt` receives the full `PromptContext` and returns the instruction string.

**9. Runs the agent**
`ToolLoopAgent` is configured with: the model, instructions, tools, a 10-step cap, step telemetry via `onStepFinish`, and a `prepareStep` hook that prunes old tool calls from message history and applies cache-control markers on every step.

**10. Streams output**
`agent.stream({ prompt })` returns a `fullStream` async iterator. `text-delta` chunks go to `stdout`; `tool-call` and `tool-result` chunks go to `stderr`.

## Sandbox Abstraction — `src/sandbox.ts`

All tools interact with the filesystem and shell through a `Sandbox` interface rather than calling Node APIs directly. This decouples tool logic from execution environment.

```ts
export interface Sandbox {
  type: string;
  workingDirectory: string;
  readFile(path: string): Promise<string>;
  exec(command: string): Promise<{ stdout: string; exitCode: number }>;
  stop(): Promise<void>;
  expiresAt?: number;                            // Unix ms when the sandbox expires
  snapshot?(): Promise<{ snapshotId: string }>;  // create a point-in-time snapshot
}
```

**Key contracts:**
- `exec` must **never throw** — it always returns `{ stdout, exitCode }`. Errors surface as non-zero exit codes, not exceptions.
- `readFile` receives a path relative to `workingDirectory`. Implementations resolve it internally.
- `expiresAt` and `snapshot` are optional — backends opt in to the capabilities they can support.

```ts
export interface SandboxLifecycle {
  afterStart?(sandbox: Sandbox): Promise<void>;
  beforeStop?(sandbox: Sandbox): Promise<void>;
  onTimeout?(sandbox: Sandbox): Promise<void>;
}
```

`afterStart` and `beforeStop` are called manually by the harness. `onTimeout` fires when `expiresAt` is reached. The lifecycle object in `index.ts` is always declared even when empty — it's a clean slot for attaching hooks without modifying call sites.

### Local Sandbox — `src/sandbox-local.ts`

Uses `fs.readFileSync` and `child_process.execSync`. Writes go directly to disk. `stop` is a no-op. Use for local development.

`exec` catches `execSync` exceptions and normalises them into `{ stdout, exitCode }`, including capturing `stderr` as `stdout` when the process fails without output.

### Just-Bash Sandbox — `src/sandbox-just-bash.ts`

Uses the `just-bash` package, which mounts the target directory as an overlay: reads come from real disk, writes go to an in-memory copy-on-write layer. When `stop()` is called the layer is discarded — the real filesystem is untouched.

```ts
const MOUNT = "/home/user/project";  // fixed mount point inside just-bash
```

The working directory inside just-bash is always `/home/user/project` regardless of the real path on disk. The `exec` implementation uses the just-bash command-handle pattern: `runCommand` → `wait()` → `output()`.

`createJustBashSandbox` is **async** (unlike `createLocalSandbox` which is sync) because the just-bash overlay requires async initialisation.

### Sandbox Lifecycle

Lifecycle hooks provide injection points for setup and teardown without modifying tool code:

```ts
const lifecycle: SandboxLifecycle = {
  afterStart: async (sandbox) => {
    // e.g. install deps, warm caches
  },
  beforeStop: async (sandbox) => {
    // e.g. flush logs, persist state
  },
};
```

In `index.ts`, `beforeStop` is called inside `try/finally` around the entire agent run — it fires even if the agent throws.

## Tools — `src/tools.ts`

All tools are created by factory functions that accept a `Sandbox` (and sometimes an approval function). This keeps tool logic testable in isolation — inject a mock sandbox, test the tool without a real filesystem.

Every tool description follows a 5-section contract:

```
<First line: what the tool does>

WHEN TO USE: <positive trigger cases>
WHEN NOT TO USE: <redirect to a better tool>
DO NOT USE FOR: <hard exclusions>
USAGE: <parameter notes, output caps, defaults>
```

This wording directly steers which tool the model picks. Weak descriptions cause the model to default to `bash` for everything ("bash gravity").

### `read`

Reads a file and returns numbered lines. Path is relative to `workingDirectory`.

- Capped at **500 lines**. Truncated output ends with `... (truncated at 500 lines)`.
- `offset` (1-indexed start line) and `limit` (max lines) let the model read targeted sections of large files.
- Intended use: `grep` to locate a target, then `read` with `offset`/`limit` to read only the relevant section.

### `grep`

Searches files using a regex pattern via `grep -rn -E`. Returns matching lines with file path and line number.

- Capped at **50 matches**. When truncated, reports the total count so the model can narrow the search pattern.
- `path` restricts the search directory (default: working directory). `glob` filters by filename pattern (e.g. `*.ts`).
- `node_modules` and `.git` are always excluded.
- Shell-quoting of pattern and path (`shellQuote`) handles special characters safely.

### `bash`

Executes a shell command in `workingDirectory` via `sandbox.exec`.

- Commands requiring approval return `Blocked: "<command>" requires approval.` and the sandbox is never called.
- Output capped at **5,000 characters**, keeping the **tail** on truncation — useful because errors typically appear at the end of build/test output.
- Non-zero exit codes surface as `Exit <code>: <output>` rather than throwing.
- Empty successful output returns `(no output)`.

### `askUser`

Presents a multiple-choice question to the user via `stderr` and returns a pending-response string to the model.

- Accepts **2–4 options** (enforced by Zod schema).
- Used for resolving ambiguity before acting: `"add auth"` → ask OAuth or JWT.
- The system prompt instructs: search first, then ask if still uncertain, then act.

### `todo`

An in-process task list for tracking multi-step work within a single agent session.

- Actions: `add` (creates an item with a random 8-char ID), `start` (marks `in_progress`), `complete`, `list`.
- Only **one item can be `in_progress`** at a time. Starting a second is rejected until the active one completes.
- The system prompt requires `todo` before touching files for any task with 3+ steps, multiple files, or ordered dependencies.

### `loadSkill`

Loads the full markdown content of a named skill on demand.

- The system prompt lists skill names and one-line descriptions only. `loadSkill` is called when the task matches a skill.
- Content capped at **4,000 characters**.
- Returns `Unknown skill: <name>` for unrecognised names.

### `task` — subagent delegation

Delegates a self-contained task to a specialised subagent. Two roles:

| Role | Model | Step budget | Tools | Writes? |
|---|---|---|---|---|
| **Explorer** (default) | GPT-5-mini | 5 | `read`, `grep` | No |
| **Executor** | GPT-5.5 | 15 | `read`, `grep`, `bash` | No (delegated bash only) |

The executor's `bash` tool uses `mode: "delegated"` approval — it can only run commands explicitly in the parent's trust list (`bun test`, `bun run typecheck`, `npx tsc --noEmit` by default). Shell operator injection is also blocked.

Results are returned as `[Explorer: N steps]\n<text>` or `[Executor: N steps]\n<text>`, giving the parent visibility into how much work was done.

```ts
// Delegate research
task({ description: "Find all files that import from src/auth.ts", subagentType: "explorer" })

// Delegate verification
task({ description: "Run the test suite and report failures", subagentType: "executor" })
```

## Approval Gates — `src/approval.ts`

The approval system decides whether a `bash` command runs automatically or is blocked. It is configured once at startup via a discriminated union:

```ts
type ApprovalConfig =
  | { mode: "interactive" }              // local dev: safe read-only prefixes auto-approved
  | { mode: "background" }               // CI/automation: everything auto-approved
  | { mode: "delegated"; trust: string[] } // subagents: only explicit trust list approved
```

```ts
export function createApproval(config: ApprovalConfig) {
  return ({ command }: { command: string }) => boolean  // true = needs approval (blocked)
}
```

**Interactive mode** auto-approves: `ls`, `cat`, `echo`, `pwd`, `which`, `head`, `tail`, `wc`, `git log`, `git status`, `git diff`. Everything else is blocked until a human approves.

**Background mode** auto-approves everything — for CI pipelines or sandboxed automation where no human is present.

**Delegated mode** (used for executor subagents) only auto-approves commands that exactly match or prefix-match an entry in the `trust` list. Shell operators (`;&|<>\`$\n\r`) in the command string cause it to be rejected even if the prefix matches — preventing injection via chaining (e.g. `bun test && rm -rf .`).

The return value is `true` (needs approval = blocked) or `false` (runs). A blocked command never reaches the sandbox's `exec`.

## Tool Registry — `src/registry.ts`

The registry is a thin ordered `Map<string, Tool>` wrapper:

```ts
export interface ToolRegistry {
  register(name: string, value: Tool): void;
  get(name: string): Tool | undefined;
  list(): string[];
  entries(): [string, Tool][];
}
```

`registerBuiltins` wires all standard tools in a fixed order. `read` and `grep` are registered first because `task` depends on them as `parentTools`. The `bash` approval mode is set based on `sandbox.type` — `just-bash` sandboxes use `background` mode since writes are discarded.

**`wrapTool`** intercepts a tool's inputs and outputs without modifying the original:

```ts
export function wrapTool<Input, Output>(
  base: Tool<Input, Output>,
  hooks: {
    beforeExecute?: (input: Input) => Input | Promise<Input>;
    afterExecute?: (output: Output) => Output | Promise<Output>;
  },
): Tool<Input, Output>
```

The base tool reference is unchanged — other consumers continue using the original. Wrapping creates a new tool object. Use it for logging, telemetry, path normalisation, or output transformation.

## System Prompt — `src/system.ts`

`buildSystemPrompt(ctx: PromptContext): string` is a pure function. It takes a context object and returns the agent's full instruction string as joined sections.

```ts
export interface PromptContext {
  workingDirectory: string;
  sandboxType: string;
  toolNames: string[];
  gitBranch?: string;
  projectContext?: string;            // from AGENTS.md
  verificationCommands?: string[];    // from discoverGates
  skills?: { name: string; description: string }[];
}
```

The prompt has eight sections:

| Section | Purpose |
|---|---|
| **Role line** | Grounds the agent: working directory and sandbox type |
| **Agency** | Commands the agent to use tools rather than speculate |
| **Guardrails** | Keeps changes minimal; no new dependencies without asking |
| **Skills** | Lists available skills by name + description (not content) |
| **Handling Ambiguity** | "Search, ask, act" — search first, then `askUser`, then act |
| **Planning** | Requires `todo` before multi-step implementation |
| **Verification** | Lists discovered gates; requires honest, scoped reporting |
| **Project Instructions** | Injects `AGENTS.md` content verbatim |

**Tool preferences appear in both tool descriptions and the Agency section.** The redundancy is intentional — cross-layer repetition reinforces routing without either layer being a single point of failure.

**Verification is scoped, not aspirational.** The prompt says "run these gates and report what passed and what didn't." The model is required to distinguish failures it caused from pre-existing ones, and must not claim "tests pass" without running them.

## Context Management — `src/cache.ts`

As a conversation grows, accumulated tool call/result pairs push important content out of the model's attention window. Two mechanisms keep context bounded:

**Pruning** (applied in `index.ts` via `prepareStep`):

```ts
prepareStep: ({ messages }) => {
  const pruned = pruneMessages({
    messages,
    toolCalls: "before-last-3-messages",
  });
  return { messages: addCacheControl(pruned) };
},
```

`pruneMessages` removes tool call/result pairs from all but the last 3 messages. The original user prompt always survives. This keeps history bounded as the agent takes more steps.

**Cache control** (`src/cache.ts`):

```ts
export function addCacheControl(messages: ModelMessage[]): ModelMessage[] {
  const lastStableIndex = messages.length - 3;
  return messages.map((message, index) => {
    if (index !== 0 && index !== lastStableIndex) return message;
    return {
      ...message,
      providerOptions: {
        ...message.providerOptions,
        anthropic: { cacheControl: { type: "ephemeral" } },
      },
    };
  });
}
```

Marks two positions with cache-control headers: the **first message** (system prompt anchor, never changes) and the **last stable message** (3 from the end — stable enough to cache, recent enough to be useful). The 2 most recent messages are not marked because they're about to be replaced next step.

**Order matters:** prune first, then mark cache positions. Marking before pruning would misidentify the stable boundary.

## Verification Gates — `src/verification.ts`

`discoverGates(sandbox)` reads `package.json` from the sandbox and returns an ordered list of verification commands for the agent to run after making changes:

1. **Typecheck** — `bun run typecheck` / `npm run typecheck` / `npx tsc --noEmit` (in preference order)
2. **Lint** — if a `lint` script exists
3. **Test** — `bun test` / `npm test`
4. **Build** — if a `build` script exists

Bun detection checks the `packageManager` field and looks for `bun.lock` or `bun.lockb` lockfiles, falling back to npm if neither is found.

The discovered commands are injected into the system prompt under `# Verification`. If no commands are discovered (no `package.json`, unrecognised project), the prompt notes this explicitly rather than silently omitting the section.

## Skills — `src/skills.ts`

Skills are on-demand knowledge documents loaded only when relevant to the current task. Each skill lives in its own directory:

```
skills/
└── auth-patterns/
    └── SKILL.md
```

`SKILL.md` format:
```markdown
---
description: One-line description shown in the system prompt listing
---
# Skill Title

Full guidance loaded on demand...
```

`discoverSkills(dirs: string[])` scans an array of directories in priority order. **The first directory wins** for deduplication — a project-local skill overrides a user-global skill of the same name.

Two directories are scanned by default:
- `<target-project>/skills` — project-level skills
- `~/.harness/skills` — user-global skills

The system prompt lists only names and one-line descriptions (minimal token cost). Full content is loaded lazily via `loadSkill` when the task matches. This avoids spending context budget on skills unrelated to the current task.

## Project Structure

```
Mu/
├── index.ts                      # CLI entry point
├── package.json
├── tsconfig.json
├── .env.example
├── .tool-versions                # Bun version pin
│
├── src/
│   ├── sandbox.ts                # Sandbox + SandboxLifecycle interfaces
│   ├── sandbox-local.ts          # Local filesystem backend
│   ├── sandbox-just-bash.ts      # In-memory copy-on-write backend
│   ├── approval.ts               # createApproval — command approval logic
│   ├── cache.ts                  # addCacheControl — prompt cache markers
│   ├── registry.ts               # createRegistry, registerBuiltins, wrapTool
│   ├── skills.ts                 # discoverSkills — skill file loader
│   ├── system.ts                 # buildSystemPrompt — agent instruction builder
│   ├── tools.ts                  # All tool factory functions
│   └── verification.ts           # discoverGates — auto-detect verification commands
│
├── tests/
│   ├── approval.test.ts
│   ├── cache.test.ts
│   ├── registry.test.ts
│   ├── sandbox-just-bash.test.ts
│   ├── skills.test.ts
│   ├── system.test.ts
│   ├── tools.test.ts
│   └── verification.test.ts
│
└── skills/
    └── auth-patterns/
        └── SKILL.md              # Example skill
```

## Running Tests

```bash
bun test                           # run all tests
bun test tests/tools.test.ts       # run a specific file
bun run typecheck                  # TypeScript type-check only
```

The just-bash integration test (`tests/sandbox-just-bash.test.ts`) spawns a Node subprocess and requires Node 20.18.1+. It auto-skips on older versions. Set `MU_TEST_NODE` to override the Node binary used.

## Design Decisions

**Sandbox interface over direct Node APIs.** Tools never call `fs` or `child_process` directly. This makes them portable across execution backends and trivially testable with a stub `Sandbox` that returns controlled values.

**Discriminated union for approval config.** Three modes are mutually exclusive and type-safe. Adding a fourth mode is one union member, not a scattered conditional. The mode is set once at startup and never changes mid-run.

**Shell operator rejection in delegated mode.** `matchesTrustedCommand` rejects any command containing `;&|<>\`$\n\r` even if the prefix matches. This prevents injection attacks like `bun test && rm -rf .` from bypassing the trust list.

**`wrapTool` preserves the base tool.** Consumers that hold a reference to the unwrapped tool continue to behave identically. Wrapping produces a new object — it never mutates the original.

**Prune before cache-mark.** `prepareStep` always prunes first, then marks cache positions on the pruned array. Marking first would identify the wrong stable boundary because the message count shifts during pruning.

**Skills loaded on demand.** Only names and descriptions appear in the base system prompt. Loading full skill content upfront would consume context for skills irrelevant to the current task.

**`exec` never throws.** Every sandbox backend returns `{ stdout, exitCode }` even for failed commands. Tools handle non-zero exit codes explicitly rather than catching exceptions — predictable and test-friendly.

**Explorer uses GPT-5-mini, Executor uses GPT-5.5.** Explorers only read and grep — high volume, low stakes, latency-sensitive. Executors make decisions and run commands — worth the slower, more capable model.
