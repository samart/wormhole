# Claude Code CLI — Architecture & Design Document

> **Version analyzed:** 2.1.25 (build 2026-01-29)
> **Package:** `@anthropic-ai/claude-code`
> **Location:** `/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js`
> **Bundle format:** Single-file minified ESM bundle (~11.7 MB, ~6,437 lines)

---

## 1. High-Level Overview

Claude Code is an agentic coding assistant that runs in the terminal. It is a Node.js application distributed as a single minified JavaScript bundle (`cli.js`) that wraps the **Anthropic Messages API** in an agentic loop with tool use, sub-agent spawning, permission controls, and a React/Ink terminal UI.

```
┌─────────────────────────────────────────────────────────────────┐
│                       cli.js (Entry Point)                      │
├──────────┬──────────┬──────────┬──────────┬─────────────────────┤
│  CLI     │  React/  │  Agent   │  Tool    │  MCP                │
│  Parser  │  Ink UI  │  Loop    │  System  │  Integration        │
│(Commander│          │          │          │                     │
│  .js)    │          │          │          │                     │
├──────────┴──────────┴──────────┴──────────┴─────────────────────┤
│           Anthropic Messages API (Streaming)                    │
├─────────────────────────────────────────────────────────────────┤
│   Session / State Management  │  Permission System  │  Hooks   │
├───────────────────────────────┴─────────────────────┴──────────┤
│  Vendor: ripgrep │ sharp │ tree-sitter │ resvg │ highlight.js  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Bundle Structure & Build

The application is built with **Bun** (evidenced by `bun.lock`) and bundled into a single ESM file using a custom build pipeline. Key characteristics:

| Property | Value |
|---|---|
| **Runtime** | Node.js >= 18.0.0 |
| **Module system** | ESM (`"type": "module"`) |
| **Minification** | Heavily minified with identifier mangling |
| **Lazy loading** | Uses `k()` wrapper for lazy module initialization |
| **Dynamic imports** | Sub-entry points loaded via `Promise.resolve().then()` |

### File inventory

```
cli.js                          # 11.7 MB - Main bundle
package.json                    # Package metadata
sdk-tools.d.ts                  # 67 KB  - TypeScript type definitions for tool inputs
resvg.wasm                      # 2.5 MB - SVG rendering (for image output)
tree-sitter.wasm                # 205 KB - Tree-sitter parser core
tree-sitter-bash.wasm           # 1.4 MB - Bash language grammar
vendor/ripgrep/                 # Platform-specific rg + ripgrep.node binaries
node_modules/@img/sharp-*       # Native image processing (optional dep)
```

### Embedded libraries (bundled into cli.js)

From the bundle analysis, the following major libraries are embedded:

- **Ink / React** — Terminal UI framework (JSX rendering via `React.createElement`)
- **Commander.js** — CLI argument parsing
- **Anthropic SDK** — API client (`new Anthropic({...})`, `messages.create`, `messages.stream`)
- **Marked** — Markdown parser/renderer
- **highlight.js** — Syntax highlighting (full language grammar set, ~50% of bundle)
- **Sentry** — Error tracking/telemetry
- **Lodash** (partial) — Utility functions (deep equal, memoize, etc.)
- **undici** — HTTP client
- **ono** — Error chaining
- **JSON Schema tools** — $ref resolution for schema validation
- **Zod (`U`)** — Runtime schema validation (aliased as `U` in minified code)
- **execa** — Child process execution
- **OpenTelemetry** — Observability (metrics, counters, tracing)
- **AWS SDK components** — For Bedrock authentication
- **Google Auth** — For Vertex AI authentication

---

## 3. Entry Points & Boot Sequence

The CLI has multiple entry points controlled by command-line arguments:

```
cli.js
├── --version / -v              → Fast-path version print
├── --mcp-cli                   → MCP CLI inspector mode
├── --ripgrep                   → Delegate to vendor ripgrep binary
├── --claude-in-chrome-mcp      → Chrome extension MCP server
├── --chrome-native-host        → Chrome native messaging host
├── --tmux                      → Tmux teammate mode
└── (default)                   → Main interactive/non-interactive agent
```

### Boot sequence (default path)

```
1.  cli_entry                    → Parse process.argv
2.  cli_imports_loaded           → Dynamic import of main module
3.  main()                       → Commander.js program setup
4.  run_before_parse             → Hooks: pre-parse lifecycle
5.  parseAsync(process.argv)     → Route to appropriate command handler
6.  action_after_input_prompt    → Process initial prompt/stdin
7.  action_tools_loaded          → Initialize tool registry
8.  action_before_setup          → Run setup() (auth, config, etc.)
9.  action_after_setup           → Load commands and agent definitions
10. action_commands_loaded        → Load custom skills from project
11. action_mcp_configs_loaded    → Connect MCP servers
12. action_after_plugins_init    → Initialize plugin system
13. Render Ink app               → Mount React component tree
```

---

## 4. Session & Global State Management

A singleton state object (`f6`) acts as the session-scoped global store:

```typescript
// Reconstructed from minified code
interface SessionState {
  // Identity
  sessionId: string;              // UUID per session
  parentSessionId?: string;       // For sub-agent correlation
  originalCwd: string;
  projectRoot: string;
  cwd: string;

  // Cost tracking
  totalCostUSD: number;
  totalAPIDuration: number;
  totalToolDuration: number;
  totalLinesAdded: number;
  totalLinesRemoved: number;
  modelUsage: Record<string, ModelUsageEntry>;

  // Configuration
  clientType: "cli" | "claude-vscode" | string;
  isInteractive: boolean;
  mainLoopModelOverride?: string;
  initialMainLoopModel: string | null;
  sdkBetas?: string;

  // Authentication
  sessionIngressToken?: string;
  oauthTokenFromFd?: string;
  apiKeyFromFd?: string;

  // Permission state
  sessionBypassPermissionsMode: boolean;
  sessionTrustAccepted: boolean;
  hasExitedPlanMode: boolean;
  hasExitedDelegateMode: boolean;

  // Hooks & plugins
  registeredHooks: Record<string, HookHandler[]> | null;
  inlinePlugins: Plugin[];
  invokedSkills: Map<string, SkillInvocation>;

  // Observability (OpenTelemetry)
  meter: Meter | null;
  sessionCounter: Counter | null;
  costCounter: Counter | null;
  tokenCounter: Counter | null;
  // ... additional counters for LOC, PRs, commits, etc.

  // Remote/teleport
  isRemoteMode: boolean;
  teleportedSessionInfo: TeleportInfo | null;
  directConnectServerUrl?: string;
}
```

Key accessors are exposed as module-level functions: `d1()` (get session ID), `bg()` (get cwd), `ZZ()` (get total cost), `sW()` (get project root), etc.

### Model usage tracking

Per-model token usage is tracked with fine granularity:

```typescript
interface ModelUsageEntry {
  inputTokens: number;
  outputTokens: number;
  cacheReadInputTokens: number;
  cacheCreationInputTokens: number;
  webSearchRequests: number;
  costUSD: number;
  contextWindow: number;       // 200K default, 1M with beta
  maxOutputTokens: number;     // Varies by model (4K–64K)
}
```

---

## 5. Anthropic API Integration (The Agent SDK Layer)

### API client construction

The application creates an `Anthropic` client instance configured with:
- API key from environment, file descriptor, or OAuth flow
- Support for **Anthropic Direct**, **AWS Bedrock**, and **Google Vertex AI** providers
- Region-aware routing for cloud providers

### SDK Betas & Features

The following beta feature flags are used in API requests:

| Beta identifier | Purpose |
|---|---|
| `claude-code-20250219` | Core Claude Code functionality |
| `interleaved-thinking-2025-05-14` | Extended thinking with interleaved output |
| `context-1m-2025-08-07` | 1M context window (Sonnet 4) |
| `context-management-2025-06-27` | Server-side context management |
| `structured-outputs-2025-12-15` | JSON schema-constrained output |
| `web-search-2025-03-05` | Web search server tool |
| `tool-examples-2025-10-29` | Tool use examples in prompts |
| `advanced-tool-use-2025-11-20` | Advanced tool patterns |
| `tool-search-tool-2025-10-19` | Deferred tool loading |
| `effort-2025-11-24` | Thinking effort control |
| `prompt-caching-scope-2026-01-05` | Prompt caching scoping |

### Context window sizing

```
Model                    Context Window    Max Output Tokens
─────────────────────    ──────────────    ─────────────────
Claude 3 Opus             200,000           4,096
Claude 3.5 Sonnet         200,000           8,192
Claude 3.5 Haiku          200,000           4,096
Opus 4.5                  200,000          64,000
Opus 4                    200,000          32,000
Sonnet 4 / Haiku 4        200,000          64,000
Sonnet 4 [1M beta]      1,000,000          64,000
```

### Prompt caching

The system uses `cache_control` with `ephemeral` type for system prompts and tool definitions to reduce token costs on repeated API calls. Cache metrics are tracked per-model and surfaced in session statistics.

---

## 6. Model Resolution System

### Model registry

The CLI maintains a registry of model descriptor objects (`cli-beautify.js:120745–120784`), each mapping a model version to provider-specific IDs for Anthropic Direct (`firstParty`), AWS Bedrock, Google Vertex AI, and Foundry:

| Internal var | Model | firstParty | Bedrock | Vertex | Foundry |
|---|---|---|---|---|---|
| `BE1` | Sonnet 3.5 | `claude-3-5-sonnet-20241022` | `anthropic.claude-3-5-sonnet-20241022-v2:0` | `claude-3-5-sonnet-v2@20241022` | `claude-3-5-sonnet` |
| `mE1` | Haiku 3.5 | `claude-3-5-haiku-20241022` | `us.anthropic.claude-3-5-haiku-20241022-v1:0` | `claude-3-5-haiku@20241022` | `claude-3-5-haiku` |
| `FE1` | Haiku 4.5 | `claude-haiku-4-5-20251001` | `us.anthropic.claude-haiku-4-5-20251001-v1:0` | `claude-haiku-4-5@20251001` | `claude-haiku-4-5` |
| `a61` | Sonnet 4 | `claude-sonnet-4-20250514` | `us.anthropic.claude-sonnet-4-20250514-v1:0` | `claude-sonnet-4@20250514` | `claude-sonnet-4` |
| `ca6` | Sonnet 4.5 | `claude-sonnet-4-5-20250929` | `us.anthropic.claude-sonnet-4-5-20250929-v1:0` | `claude-sonnet-4-5@20250929` | `claude-sonnet-4-5` |
| `gE1` | Opus 4 | `claude-opus-4-20250514` | `us.anthropic.claude-opus-4-20250514-v1:0` | `claude-opus-4@20250514` | `claude-opus-4` |
| `QE1` | Opus 4.1 | `claude-opus-4-1-20250805` | `us.anthropic.claude-opus-4-1-20250805-v1:0` | `claude-opus-4-1@20250805` | `claude-opus-4-1` |
| `s61` | Opus 4.5 | `claude-opus-4-5-20251101` | `us.anthropic.claude-opus-4-5-20251101-v1:0` | `claude-opus-4-5@20251101` | `claude-opus-4-5` |

The default "sonnet" base variable (`sW5`) points to `a61` (Sonnet 4), and `h97` stores its firstParty model ID string.

### Provider detection — `F4()`

The function `F4()` (`cli-beautify.js:64528`) determines which provider is active:

```javascript
function F4() {
    return CLAUDE_CODE_USE_BEDROCK  ? "bedrock"
         : CLAUDE_CODE_USE_VERTEX   ? "vertex"
         : CLAUDE_CODE_USE_FOUNDRY  ? "foundry"
         : "firstParty"
}
```

The provider selects which column of the model registry is used when resolving IDs.

### Model map accessor — `sO()` and `UE1()`

`sO()` (`cli-beautify.js:120874`) returns the active model map — a dictionary mapping generation names to provider-appropriate model ID strings:

```javascript
// UE1() builds the map from the static registry for a given provider key:
function UE1(providerKey) {
    return {
        haiku35:  mE1[providerKey],   // e.g. "claude-3-5-haiku-20241022"
        haiku45:  FE1[providerKey],   // e.g. "claude-haiku-4-5-20251001"
        sonnet35: BE1[providerKey],   // e.g. "claude-3-5-sonnet-20241022"
        sonnet37: uE1[providerKey],   // e.g. "claude-3-7-sonnet-20250219"
        sonnet40: a61[providerKey],   // e.g. "claude-sonnet-4-20250514"
        sonnet45: ca6[providerKey],   // e.g. "claude-sonnet-4-5-20250929"
        opus40:   gE1[providerKey],   // e.g. "claude-opus-4-20250514"
        opus41:   QE1[providerKey],   // e.g. "claude-opus-4-1-20250805"
        opus45:   s61[providerKey],   // e.g. "claude-opus-4-5-20251101"
    }
}
```

For Bedrock, `sO()` dynamically discovers which models are available via API (`wK5()` at `cli-beautify.js:120835`) using the `Tx()` helper to match foundation model IDs against the registry. If discovery fails, it falls back to the static registry.

### Valid shorthand names

The recognized shorthand model names (`cli-beautify.js:128591`):

```javascript
YO1 = ["sonnet", "opus", "haiku", "sonnet[1m]", "opusplan"]
```

Plus `"inherit"` for sub-agents (stored in `zO1 = [...YO1, "inherit"]`).

### The core resolver — `b2()`

When you type `/model opus`, the core resolver `b2()` (`cli-beautify.js:128470`) translates shorthands to full model IDs:

```javascript
function b2(A) {
    let q = A.trim(),
        K = q.toLowerCase(),
        Y = K.endsWith("[1m]"),                // Check for 1M context suffix
        z = Y ? K.replace(/\[1m]$/i, "") : K;  // Strip suffix for lookup
    if (rt6(z)) switch (z) {                    // rt6() checks YO1.includes(z)
        case "opusplan": return eE() + (Y ? "[1m]" : "");  // → Sonnet 4.5
        case "sonnet":   return eE() + (Y ? "[1m]" : "");  // → Sonnet 4.5
        case "haiku":    return lt6() + (Y ? "[1m]" : "");  // → Haiku 4.5
        case "opus":     return wi();                        // → Opus 4.5 or 4.1
    }
    if (Y) return q.replace(/\[1m\]$/i, "").trim() + "[1m]";
    return q  // Not a shorthand → treat as literal model ID string
}
```

The `[1m]` suffix is preserved through resolution — it signals the server to use the 1M context window beta.

### Individual resolver functions

Each shorthand delegates to a function that checks environment variable overrides before falling back to the model registry:

**`"opus"` → `wi()`** (`cli-beautify.js:128236`):

```javascript
function wi() {
    if (process.env.ANTHROPIC_DEFAULT_OPUS_MODEL)
        return process.env.ANTHROPIC_DEFAULT_OPUS_MODEL;
    if (F4() === "firstParty") return sO().opus45;  // → "claude-opus-4-5-20251101"
    return sO().opus41                               // → Opus 4.1 ID for Bedrock/Vertex
}
```

**`"sonnet"` → `eE()`** (`cli-beautify.js:128219`):

```javascript
function eE() {
    if (process.env.ANTHROPIC_DEFAULT_SONNET_MODEL)
        return process.env.ANTHROPIC_DEFAULT_SONNET_MODEL;
    return sO().sonnet45  // → "claude-sonnet-4-5-20250929"
}
```

**`"haiku"` → `lt6()`** (`cli-beautify.js:128242`):

```javascript
function lt6() {
    if (process.env.ANTHROPIC_DEFAULT_HAIKU_MODEL)
        return process.env.ANTHROPIC_DEFAULT_HAIKU_MODEL;
    return sO().haiku45  // → "claude-haiku-4-5-20251001"
}
```

> **Provider asymmetry:** On non-firstParty providers (Bedrock, Vertex), `"opus"` resolves to **Opus 4.1** rather than 4.5 via `sO().opus41`. On firstParty (Anthropic Direct), it resolves to **Opus 4.5** via `sO().opus45`. This is because Opus 4.5 is not available on all cloud providers.

### Full resolution chain

```
User types: /model opus
    │
    ▼
C66() stores "opus" as the session model selection
    │   Sources (in priority order):
    │   1. CLI arg (Ce())
    │   2. process.env.ANTHROPIC_MODEL
    │   3. settings.model from config
    │
    ▼
fA1() returns current model shorthand → "opus"
    │
    ▼
q5() calls b2("opus") for full resolution
    │
    ▼
b2() recognizes "opus" as valid shorthand (in YO1 array)
    │
    ▼
wi() — the opus resolver
    │
    ├─ Check ANTHROPIC_DEFAULT_OPUS_MODEL env var → use if set
    │
    ├─ F4() === "firstParty"?
    │   ├─ Yes → sO().opus45 → "claude-opus-4-5-20251101"
    │   └─ No  → sO().opus41 → provider-specific Opus 4.1 ID
    │
    ▼
Final model ID passed to messages.stream()
```

### Default model selection

When no model is explicitly selected, `S66()` (`cli-beautify.js:128280`) determines the default:

```javascript
function S66() {
    // 1. Check for custom model from settings/plugins
    let customModel = B97();
    if (customModel !== undefined) return customModel;

    // 2. Max/Team/Pro plans default to Opus
    if (Dk1() || jk1() || HO1()) return wi();  // → Opus 4.5

    // 3. Everyone else defaults to Sonnet
    return eE();  // → Sonnet 4.5
}
```

The resolved default is then passed through `b2()` via `Ak()` to get the final model ID:

| Plan tier | Default model | Resolved ID |
|---|---|---|
| Max, Team, Pro | `wi()` → Opus | `claude-opus-4-5-20251101` |
| Free, other | `eE()` → Sonnet | `claude-sonnet-4-5-20250929` |

### The model picker UI

The `/model` command presents a selection menu with these options (`cli-beautify.js:128595–128632`):

```
┌──────────────────────────────────────────────────────────────────┐
│  Sonnet        Sonnet 4.5 · Best for everyday tasks · $X/mo     │
│  Sonnet (1M)   Sonnet 4.5 with 1M context · Uses rate limits    │
│  Opus          Opus 4.5 · Most capable for complex work         │
│  Haiku         Haiku 4.5 · Fastest for quick answers            │
│  Opus Plan     Use Opus 4.5 in plan mode, Sonnet 4.5 otherwise  │
└──────────────────────────────────────────────────────────────────┘
```

On non-firstParty providers, the Opus label shows "Opus 4.1" with a "Legacy" description instead.

The `"opusplan"` mode (`cli-beautify.js:128566`) is a cost-optimization strategy: it uses Opus 4.5 only when in plan mode (`permissionMode === "plan"`) via the `Hi()` function (`cli-beautify.js:128247`), and Sonnet 4.5 for normal execution.

### Sub-agent model resolution — `I66()`

Sub-agents (spawned via the `Task` tool) resolve their model through `I66()` (`cli-beautify.js:128500`):

```javascript
function I66(agentModelSetting, mainLoopModel, explicitModel, permissionMode) {
    // 1. Global sub-agent override takes highest priority
    if (process.env.CLAUDE_CODE_SUBAGENT_MODEL)
        return b2(process.env.CLAUDE_CODE_SUBAGENT_MODEL);

    // 2. Explicit model from Task tool call (e.g., model: "haiku")
    if (explicitModel) return b2(explicitModel);

    // 3. Agent type's configured model setting
    let agentModel = agentModelSetting ?? R66();
    if (agentModel === "inherit")
        return Hi({ permissionMode, mainLoopModel, exceeds200kTokens: false });

    // 4. Resolve the agent type's default via b2()
    return b2(agentModel)
}
```

For example, the `claude-code-guide` agent has `model: "haiku"`, so it resolves to `lt6()` → `"claude-haiku-4-5-20251001"`.

### Knowledge cutoff dates

The function `rsY()` (`cli-beautify.js:4769`) maps model families to knowledge cutoff dates for use in system prompts:

```javascript
function rsY(model) {
    if (model.includes("claude-opus-4-5"))                           return "May 2025";
    if (model.includes("claude-haiku-4"))                            return "February 2025";
    if (model.includes("claude-opus-4") ||
        model.includes("claude-sonnet-4-5") ||
        model.includes("claude-sonnet-4"))                           return "January 2025";
    return null;
}
```

### Display name resolution

`Iy()` (`cli-beautify.js:128490`) generates human-readable model descriptions:

```javascript
function Iy(model) {
    if (model === null) {
        if (isFirstPartyNonPaid()) return `Sonnet (${sonnetDescription})`;
        if (isFirstParty())        return `Default (${defaultModelName})`;
        return `Default (${resolvedDefaultModel})`
    }
    let resolved = b2(model);
    return model === resolved ? model : `${model} (${resolved})`
    // e.g., "opus (claude-opus-4-5-20251101)"
}
```

### Model family detection

Several utility functions extract model family information from the resolved ID string:

```javascript
// QW() — extract model family slug (cli-beautify.js:128293)
function QW(model) {
    if (model.includes("claude-opus-4-5"))  return "claude-opus-4-5";
    if (model.includes("claude-opus-4-1"))  return "claude-opus-4-1";
    if (model.includes("claude-opus-4"))    return "claude-opus-4";
    // ... regex fallback for other models
}

// y66() — check if model is opus-class (used for plan gating)
function y66(model) { return model.includes("opus") }
```

### Model-related environment variables

| Variable | Purpose |
|---|---|
| `ANTHROPIC_MODEL` | Set model directly, bypasses shorthand resolution entirely |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Override what `"opus"` shorthand resolves to |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Override what `"sonnet"` shorthand resolves to |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Override what `"haiku"` shorthand resolves to |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Force a specific model for all sub-agents |
| `CLAUDE_CODE_USE_BEDROCK` | Use AWS Bedrock provider (changes model ID format) |
| `CLAUDE_CODE_USE_VERTEX` | Use Google Vertex AI provider |
| `CLAUDE_CODE_USE_FOUNDRY` | Use Foundry provider |

### Hardcoded model constants

Two constants appear in the system prompt generation code (`cli-beautify.js:433687–433688`):

```javascript
SsY = "Claude Opus 4.5"             // Display name used in system prompt header
hsY = "claude-opus-4-5-20251101"     // Model ID used for system prompt generation
```

These are used when constructing the `Co-Authored-By` attribution and the model identity section of the system prompt.

---

## 7. The Agentic Loop

The core agentic loop follows the standard Messages API tool-use pattern:

```
┌──────────────────────────────────────────────────────┐
│                   Agentic Main Loop                   │
│                                                       │
│  1. Construct messages array (system + conversation)  │
│  2. Call messages.stream() with tools                 │
│  3. Stream response tokens to Ink UI                  │
│  4. If stop_reason == "tool_use":                     │
│     a. Extract tool_use blocks                        │
│     b. Check permissions for each tool                │
│     c. Execute tools (possibly in parallel)           │
│     d. Collect tool_result blocks                     │
│     e. Append to conversation → goto 1                │
│  5. If stop_reason == "end_turn":                     │
│     a. Render final response                          │
│     b. Wait for next user input                       │
│     c. Append user message → goto 1                   │
└──────────────────────────────────────────────────────┘
```

### Extended thinking

When enabled, the agent uses `interleaved-thinking-2025-05-14` beta to receive `thinking` content blocks between tool calls and text output. The user can trigger "ultrathink" or set `maxThinkingTokens` to control the thinking budget. Thinking tokens count toward `budget_tokens`.

### Context management

The system includes automatic context management via the `context-management-2025-06-27` beta, allowing the server to handle summarization and context pruning when the conversation approaches the context window limit.

---

## 8. Tool System Architecture

### Tool registry

Tools are defined as typed objects conforming to a standard interface:

```typescript
interface ToolDefinition {
  name: string;
  description(): Promise<string>;    // Dynamic prompt generation
  inputSchema: ZodSchema;            // Zod validation schema
  outputSchema?: ZodSchema;
  isEnabled(): boolean;
  isReadOnly(): boolean;
  isConcurrencySafe(): boolean;
  checkPermissions(input, context): PermissionResult;
  validateInput(input, context): ValidationResult;
  call(input, context, toolUse, meta): Promise<ToolResult>;

  // UI rendering hooks
  renderToolUseMessage(input, options): ReactNode;
  renderToolResultMessage(data, toolUse, options): ReactNode;
  renderToolUseRejectedMessage(input, options): ReactNode;
  renderToolUseErrorMessage(result, options): ReactNode;

  mapToolResultToToolResultBlockParam(data, toolUseId): ToolResultBlock;
}
```

### Built-in tools

| Tool name | Key | Read-only | Concurrent-safe | Purpose |
|---|---|---|---|---|
| `Bash` | `BashTool` | No | No | Execute shell commands |
| `Read` | `FileReadTool` | Yes | Yes | Read files from filesystem |
| `Edit` | `FileEditTool` | No | No | String-replacement file editing |
| `Write` | `FileWriteTool` | No | No | Write/create files |
| `Glob` | `GlobTool` | Yes | Yes | File pattern matching |
| `Grep` | `GrepTool` | Yes | Yes | Content search via ripgrep |
| `Task` | `AgentTool` | — | Yes | Spawn sub-agents |
| `TaskOutput` | — | Yes | Yes | Retrieve sub-agent output |
| `TaskStop` | — | No | No | Terminate background tasks |
| `WebFetch` | — | Yes | Yes | Fetch and analyze web content |
| `WebSearch` | — | Yes | Yes | Web search via server tool |
| `NotebookEdit` | — | No | No | Jupyter notebook cell editing |
| `TodoWrite` | — | No | No | Task list management |
| `AskUserQuestion` | — | — | — | Interactive user prompting |
| `ExitPlanMode` | — | — | — | Signal plan completion |
| `EnterPlanMode` | — | — | — | Transition to planning mode |
| `ToolSearch` | — | Yes | Yes | Deferred tool discovery/loading |
| `Config` | — | No | No | Read/write settings |
| `Skill` | — | — | — | Invoke registered skills |
| `ListMcpResources` | — | Yes | Yes | List MCP resources |
| `ReadMcpResource` | — | Yes | Yes | Read MCP resources |

### Deferred tool loading (ToolSearch)

MCP tools and optionally large tool sets use a **deferred loading** pattern:
1. The model sees a `ToolSearch` tool with a list of available deferred tool names
2. The model calls `ToolSearch` with a query or `select:tool_name` to load specific tools
3. Once loaded, the tools become available for the remainder of the conversation

### File state tracking

File tools maintain a `readFileState` map tracking:
- Last-read content and timestamp per file
- Offset/limit for partial reads
- Validation that files haven't been modified externally between read and edit

This enforces the invariant: **files must be read before they can be written/edited**.

---

## 9. Sub-Agent Architecture (Task Tool)

The `Task` tool (aliased as `AgentTool`) is the core mechanism for spawning sub-agents:

```typescript
interface AgentInput {
  prompt: string;                                    // Task description
  description: string;                               // Short label (3-5 words)
  subagent_type: string;                             // Agent type identifier
  model?: "sonnet" | "opus" | "haiku";               // Model override
  resume?: string;                                   // Resume from agent ID
  run_in_background?: boolean;                       // Async execution
  max_turns?: number;                                // Turn limit
  mode?: "acceptEdits" | "bypassPermissions" |       // Permission mode
         "default" | "delegate" | "dontAsk" | "plan";
}
```

### Built-in agent types

| Agent type | Model | Purpose | Available tools |
|---|---|---|---|
| `Explore` | Inherited | Fast codebase exploration | All except Task, Edit, Write |
| `Plan` | Inherited | Implementation planning | All except Task, Edit, Write |
| `claude-code-guide` | Haiku | Documentation/help | Read, Glob, Grep, WebFetch, WebSearch |
| `statusline-setup` | Sonnet | Configure status line | Read, Edit |
| Custom agents | Configurable | Project-specific | Configurable |

### Agent spawning modes

Agents can be spawned in three modes:
1. **In-process** — Runs as a nested agentic loop within the same Node.js process
2. **Tmux** — Spawns a new Claude Code process in a tmux pane
3. **Auto** — Chooses between in-process and tmux based on context

### Agent lifecycle

```
Parent Agent                    Sub-Agent
    │                               │
    ├─ Task(prompt, type) ─────────►│
    │                               ├─ Initialize with subset of tools
    │                               ├─ Run agentic loop with own model
    │                               ├─ Return result (or write to file)
    │◄──────────── result ──────────┤
    │                               │
    ├─ (optional) resume(agentId) ─►│  ← Resume with full context
    │                               │
```

Background agents write output to a file that can be monitored:
```
/private/tmp/claude-501/<project>/tasks/<agentId>.output
```

---

## 10. Permission System

### Permission modes

| Mode | Behavior |
|---|---|
| `default` | Ask user for permission on sensitive operations |
| `acceptEdits` | Auto-accept file edits, ask for bash commands |
| `bypassPermissions` | Skip all permission checks |
| `plan` | Require plan approval before implementation |
| `delegate` | Delegate mode for teammate agents |
| `dontAsk` | Never ask (for read-only sub-agents) |

### Permission checking flow

```
Tool invocation
    │
    ├─ Is mode "bypassPermissions"? ──► Execute immediately
    │
    ├─ Check tool-specific permissions
    │   ├─ Path-based allow/deny rules
    │   ├─ Command pattern matching (Bash)
    │   └─ Scope-based rules (user/project/local settings)
    │
    ├─ Result: allow / deny / ask
    │   ├─ allow → Execute
    │   ├─ deny  → Return error to model
    │   └─ ask   → Render permission prompt to user
    │             ├─ User approves → Execute
    │             └─ User denies  → Return rejection to model
    │
    └─ Post-execution: run hooks
```

### Sandbox enforcement

Bash commands can be sandboxed to prevent dangerous operations. The system tracks sandbox violations (e.g., macOS sandbox `deny` operations) and surfaces them appropriately.

---

## 11. Hooks System

Hooks allow shell commands to run at specific lifecycle points:

```typescript
type HookEvent =
  | "PreToolUse"              // Before a tool executes
  | "PostToolUse"             // After a tool executes
  | "Notification"            // On system notifications
  | "user-prompt-submit-hook" // When user submits input

interface HookHandler {
  command: string | "callback";
  pluginRoot?: string;        // If from a plugin
}
```

Hooks are configured per-project in settings and registered at session initialization. The hook output is fed back as context, treated as user-sourced.

---

## 12. MCP (Model Context Protocol) Integration

Claude Code acts as an **MCP client** that connects to external MCP servers:

### Connection flow

```
1. Load MCP config from settings (user/project/local scopes)
2. For each server config:
   a. Establish transport (stdio, SSE, WebSocket, SDK)
   b. Call listTools() to discover available tools
   c. Register tools as deferred (loaded via ToolSearch)
3. MCP tools become available through the ToolSearch mechanism
4. ListMcpResources / ReadMcpResource for resource access
```

### MCP server types

| Type | Transport | Description |
|---|---|---|
| `stdio` | Stdin/stdout | Local process communication |
| `sse` | HTTP SSE | Server-sent events |
| `sdk` | In-process | SDK-based integration |

### Chrome Extension MCP

A special MCP server mode (`--claude-in-chrome-mcp`) enables communication with the Claude browser extension via Unix domain sockets, allowing the browser to act as a tool provider.

---

## 13. UI Architecture (React/Ink)

The terminal UI is built with **Ink** (React for the terminal):

### Component hierarchy

```
<YfA>                          {/* FPS metrics wrapper */}
  <$Y initialState={...}>      {/* App state provider */}
    <bYA>                       {/* Main conversation view */}
      ├─ Messages list          {/* Scrolling message history */}
      │   ├─ User messages
      │   ├─ Assistant text
      │   ├─ Tool use renders   {/* Each tool has custom renderers */}
      │   └─ Tool results
      ├─ Input area             {/* User text input */}
      ├─ Status bar             {/* Model, cost, tokens */}
      └─ Permission prompts     {/* Inline permission UI */}
    </bYA>
  </$Y>
</YfA>
```

### Tool rendering

Each tool defines four render methods for the UI:
- `renderToolUseMessage` — Shows what the tool is about to do (e.g., file path being edited)
- `renderToolResultMessage` — Shows the outcome (e.g., diff view, file content)
- `renderToolUseRejectedMessage` — Shows what would have happened (preview)
- `renderToolUseErrorMessage` — Shows error information

### Syntax highlighting

Code blocks use **highlight.js** with the full language grammar set for rich syntax highlighting in the terminal, rendered with ANSI escape codes.

---

## 14. Skills System

Skills are extensible slash commands (`/commit`, `/review-pr`, etc.):

```typescript
interface Skill {
  name: string;
  description: string;
  type: "prompt" | "local";
  source: "built-in" | "plugin" | "project";
  disableNonInteractive?: boolean;
  supportsNonInteractive?: boolean;
}
```

When invoked (via the `Skill` tool), a skill expands into a detailed prompt that is injected into the conversation. Skills can come from:
1. **Built-in** — Shipped with Claude Code
2. **Project** — Defined in project `.claude/` configuration
3. **Plugin** — Installed from marketplaces

---

## 15. Plugin & Marketplace System

The plugin system (internally called "tengu") supports:

```
Marketplace → Plugin → { Skills, Agents, Hooks, MCP Servers }
```

### Plugin lifecycle

```
marketplace add <source>     → Register a marketplace
marketplace update           → Pull latest from sources
plugin install <name>        → Install to user/project/local scope
plugin enable/disable        → Toggle without uninstalling
plugin update               → Update to latest version
```

Plugins can provide:
- Custom agent types
- Custom skills (slash commands)
- Hook handlers
- MCP server configurations

---

## 16. Authentication Architecture

```
Authentication Resolution Order:
1. ANTHROPIC_API_KEY env var
2. API key from file descriptor (--api-key-fd)
3. OAuth token from file descriptor
4. OAuth token from stored session
5. Interactive OAuth flow (browser redirect)

Provider Selection:
├─ ANTHROPIC_API_KEY + ANTHROPIC_BASE_URL → Direct API
├─ AWS credentials → Bedrock
├─ GOOGLE credentials → Vertex AI
└─ OAuth token → claude.ai proxy
```

The OAuth flow uses `claude.ai` as the authorization server. Long-lived tokens (1-year) can be set up via `claude setup-token`.

---

## 17. Observability & Telemetry

### OpenTelemetry metrics

The system tracks metrics via OpenTelemetry counters:

| Counter | Description |
|---|---|
| `claude_code.session.count` | Sessions started |
| `claude_code.lines_of_code.count` | Lines added/removed |
| `claude_code.pull_request.count` | PRs created |
| `claude_code.commit.count` | Commits made |
| `claude_code.cost.usage` | USD cost |
| `claude_code.token.usage` | Tokens consumed |
| `claude_code.code_edit_tool.decision` | Edit accept/reject |
| `claude_code.active_time.total` | Active time (seconds) |

### Event logging

Events are logged via `n("tengu_*", {...})` calls throughout the codebase (Sentry/Statsig integration), covering:
- Feature flag evaluations (`G4("tengu_*", default)`)
- Tool usage patterns
- Error conditions
- Performance metrics

### Debug logging

Debug logs write to `~/.claude/debug/<sessionId>.txt` with configurable verbosity via `DEBUG` env var or `--debug` flag. A buffered writer (`lnA`) batches log writes for performance.

---

## 18. Git Integration

Deep git integration provides:

- **Status tracking** — `git status` for file state awareness
- **Diff generation** — Structured patch creation for file edits
- **Worktree support** — Multi-worktree repository handling
- **Auto-stash** — Automatic stashing before operations
- **Commit authorship** — `Co-Authored-By: Claude` attribution
- **PR creation** — Via `gh` CLI integration

---

## 19. Non-Interactive & Remote Modes

### Non-interactive mode (piped/SDK)

When invoked with `-p` (print mode), Claude Code operates without a terminal UI:

```bash
echo "fix the bug" | claude -p --output-format json
```

Output formats: `text`, `json`, `stream-json`

### Remote sessions

The `--remote` flag creates a session on `claude.ai` that can be:
- Monitored via web UI at `https://claude.ai/code/<sessionId>`
- Resumed locally with `--teleport <sessionId>`

### SDK integration

When used as an SDK (`--sdk-url`), Claude Code connects to a WebSocket endpoint for I/O streaming, enabling integration into larger agent orchestration systems.

---

## 20. Configuration Hierarchy

Settings are resolved from multiple scopes with precedence:

```
Policy settings    (highest priority — org-level enforcement)
  ↓
Flag settings      (feature flags / remote config)
  ↓
Local settings     (.claude/local-settings.json — gitignored)
  ↓
Project settings   (.claude/settings.json — committed)
  ↓
User settings      (~/.claude/settings.json)
```

### Key configuration areas

- **Model selection** — Default model, per-agent model overrides
- **Permission rules** — Allow/deny patterns for tools and paths
- **MCP servers** — Server configurations per scope
- **Hooks** — Lifecycle hook definitions
- **Plugins** — Marketplace and plugin configurations
- **Status line** — Custom terminal status line command

---

## 21. Environment Variables

Key environment variables (extracted from bundle analysis):

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | API authentication |
| `ANTHROPIC_MODEL` | Default model override (bypasses shorthand resolution) |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Override what `"opus"` shorthand resolves to |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Override what `"sonnet"` shorthand resolves to |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Override what `"haiku"` shorthand resolves to |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Force a specific model for all sub-agents |
| `ANTHROPIC_BASE_URL` | Custom API endpoint |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Output token limit (default 32K, max 64K) |
| `CLAUDE_CODE_SESSION_ID` | Session identifier |
| `CLAUDE_CONFIG_DIR` | Custom config directory (default `~/.claude`) |
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | Debug log directory |
| `CLAUDE_CODE_ENTRYPOINT` | Entry mode identifier |
| `CLAUDE_CODE_REMOTE` | Remote mode flag |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | Bash cwd behavior |
| `BASH_MAX_OUTPUT_LENGTH` | Bash output truncation (default 30K, max 150K) |
| `TASK_MAX_OUTPUT_LENGTH` | Task output truncation |
| `CLAUDE_CODE_USE_BEDROCK` | Use AWS Bedrock provider (changes model ID format) |
| `CLAUDE_CODE_USE_VERTEX` | Use Google Vertex AI provider |
| `CLAUDE_CODE_USE_FOUNDRY` | Use Foundry provider |
| `AWS_REGION` / `AWS_DEFAULT_REGION` | Bedrock region |
| `CLOUD_ML_REGION` | Vertex AI region |
| `VERTEX_REGION_CLAUDE_*` | Per-model Vertex regions |
| `MCP_CONNECTION_NONBLOCKING` | Non-blocking MCP startup |
| `DEBUG` | Debug logging filter |

---

## 22. How Claude Code Uses the Agent SDK Pattern

Claude Code itself **is** the reference implementation of the Claude Agent SDK pattern. Rather than importing a separate SDK library, it implements the agentic loop directly:

### Core agent loop pattern

```
                    ┌─────────────────────┐
                    │   System Prompt     │
                    │   + Tool Defs       │
                    │   + CLAUDE.md       │
                    │   + Context         │
                    └────────┬────────────┘
                             │
              ┌──────────────▼──────────────┐
              │   messages.stream({          │
              │     model, system, tools,   │
              │     messages, betas,        │
              │     max_tokens,             │
              │     thinking: { budget }    │
              │   })                        │
              └──────────────┬──────────────┘
                             │
              ┌──────────────▼──────────────┐
              │   Stream Processing:        │
              │   - text → render to UI     │
              │   - thinking → show/hide    │
              │   - tool_use → dispatch     │
              └──────────────┬──────────────┘
                             │
                    ┌────────▼────────┐
                    │  Tool Dispatch  │
                    │  (parallel OK   │
                    │   if safe)      │
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │   Permission Check          │
              │   → Execute Tool            │
              │   → Run Post-Use Hooks      │
              │   → Map Result to API       │
              └──────────────┬──────────────┘
                             │
                    ┌────────▼────────┐
                    │  Append results │
                    │  to messages    │
                    │  → loop back    │
                    └─────────────────┘
```

### SDK-facing type definitions

The `sdk-tools.d.ts` file provides TypeScript types for all tool inputs, enabling:
1. **Programmatic usage** — Build agents that call Claude Code tools programmatically
2. **SDK integration** — Use Claude Code as a tool provider in larger agent systems
3. **Type safety** — Validate tool inputs at compile time

### Sub-agent as recursive agent instantiation

The `Task` tool creates a **new agent loop** with:
- Its own subset of tools (per agent type definition)
- Its own model (can differ from parent)
- Its own permission mode
- Its own system prompt (from agent definition)
- A turn limit (`max_turns`) for cost control
- The ability to run in the background and be resumed later

This recursive agent spawning is the core of the "Agent SDK" pattern — agents can orchestrate other agents, each with appropriate tools and constraints.

---

## 23. Security Model

| Layer | Mechanism |
|---|---|
| **File access** | Path-based allow/deny rules per permission scope |
| **Bash commands** | Pattern matching, sandbox mode, description-based review |
| **API keys** | Stored in OS keychain or env vars, never in config files |
| **MCP servers** | Per-server trust, tool-level permissions |
| **Network** | WebFetch preflight checks, domain filtering |
| **File edits** | Mandatory read-before-write, external modification detection |
| **Git safety** | Never force push to main, never amend without permission |

---

## 24. CLI Flags & Options Reference

Complete reference of all `claude` CLI flags. Flags that enable **adding context**, **workflows/automation**, or **extensions/extensibility** are marked with badges.

### 24.1 Core Execution Modes

| Flag | Description |
|---|---|
| `(no flag)` | Start an interactive REPL session |
| `"query"` | Start REPL with an initial prompt (e.g., `claude "fix the bug"`) |
| `-p`, `--print` | **Print mode** — non-interactive/headless, runs query and exits. Required for CI/automation |
| `-c`, `--continue` | Continue the most recent conversation in the current directory |
| `-r`, `--resume <id\|name>` | Resume a specific session by ID or name (interactive picker if no arg) |
| `-v`, `--version` | Print version and exit |
| `update` | Update Claude Code to the latest version |

### 24.2 Context & Input Flags

> Flags that control what information Claude has access to during a session.

| Flag | Description | Badge |
|---|---|---|
| `--add-dir <directories...>` | Add additional working directories Claude can access beyond the current directory. Enables multi-repo or monorepo workflows | **CONTEXT** |
| `--file <specs...>` | File resources to download at startup. Format: `file_id:relative_path` (e.g., `--file file_abc:doc.txt file_def:img.png`) | **CONTEXT** |
| `--system-prompt <prompt>` | **Replace** the entire default system prompt with custom text. Removes all built-in instructions | **CONTEXT** |
| `--append-system-prompt <prompt>` | **Append** text to the default system prompt, preserving built-in behavior. **Recommended** for most use cases | **CONTEXT** |
| `--input-format <format>` | Input format: `text` (default), `stream-json` (print mode only) | |
| `--include-partial-messages` | Include partial streaming events in output (requires `-p` + `--output-format=stream-json`) | |

> **Tip:** The `CLAUDE.md` file in a project root and `~/.claude/CLAUDE.md` also inject context automatically. The `--append-system-prompt` flag is the CLI equivalent for one-off instructions.

### 24.3 Model Selection

| Flag | Description |
|---|---|
| `--model <name>` | Set the model. Accepts shorthands (`sonnet`, `opus`, `haiku`, `sonnet[1m]`, `opusplan`) or full model IDs |
| `--fallback-model <name>` | Automatic fallback model when primary is overloaded (print mode only) |

### 24.4 Output & Formatting

| Flag | Description |
|---|---|
| `--output-format <format>` | Output format (print mode only): `text` (default), `json` (single result), `stream-json` (realtime streaming) |
| `--json-schema <schema>` | JSON Schema for structured output validation (e.g., `'{"type":"object","properties":{"name":{"type":"string"}}}'`) |
| `--verbose` | Override verbose mode setting from config |
| `--replay-user-messages` | Re-emit user messages from stdin back on stdout for acknowledgment (requires `--input-format=stream-json` and `--output-format=stream-json`) |

### 24.5 Permission & Security

| Flag | Description |
|---|---|
| `--permission-mode <mode>` | Start in a specific mode: `default`, `plan`, `acceptEdits`, `bypassPermissions`, `delegate`, `dontAsk` |
| `--allowedTools`, `--allowed-tools <tools...>` | Comma or space-separated tools that execute without prompting (e.g., `"Bash(git:*)" "Read"`) |
| `--disallowedTools`, `--disallowed-tools <tools...>` | Comma or space-separated tools removed entirely — Claude cannot use them |
| `--tools <tools...>` | Restrict which built-in tools are available. Use `""` to disable all, `"default"` for all, or specific names (e.g., `"Bash,Edit,Read"`) |
| `--dangerously-skip-permissions` | Bypass **all** permission checks. Recommended only for sandboxes with no internet access |
| `--allow-dangerously-skip-permissions` | Enable permission bypassing as an available option without activating it by default |

### 24.6 Extensions, Agents & Plugins

> Flags that extend Claude Code's capabilities with custom agents, plugins, and MCP servers.

| Flag | Description | Badge |
|---|---|---|
| `--agent <name>` | Route the session to a specific subagent type | **EXTENSION** |
| `--agents <json>` | Define custom subagents inline via JSON (e.g., `'{"reviewer":{"description":"...","prompt":"..."}}'`) | **EXTENSION** |
| `--plugin-dir <path...>` | Load plugins from directories for this session only | **EXTENSION** |
| `--mcp-config <path\|json>` | Load MCP server configurations from a JSON file or inline JSON string | **EXTENSION** |
| `--strict-mcp-config` | **Only** use MCP servers from `--mcp-config`, ignoring settings files | **EXTENSION** |
| `--disable-slash-commands` | Disable all skills and slash commands | |
| `--chrome` / `--no-chrome` | Enable/disable Chrome browser integration for web automation | **EXTENSION** |
| `--ide` | Auto-connect to IDE integration on startup | **EXTENSION** |

### 24.7 Session Management

| Flag | Description |
|---|---|
| `--continue`, `-c` | Continue most recent conversation in current directory |
| `--resume`, `-r` | Resume specific session by ID or name |
| `--session-id <uuid>` | Use a specific UUID as the session ID |
| `--fork-session` | When resuming, create a new session ID instead of reusing the original |
| `--no-session-persistence` | Disable session persistence — sessions will not be saved to disk and cannot be resumed (print mode only) |
| `--from-pr [value]` | Resume a session linked to a PR by PR number/URL, or open interactive picker with optional search term |

### 24.8 Workflow & Cost Control

> Flags for automation pipelines and resource management.

| Flag | Description | Badge |
|---|---|---|
| `--max-budget-usd <amount>` | Maximum dollar amount to spend on API calls before stopping (print mode only) | **WORKFLOW** |

### 24.9 Configuration & Debugging

| Flag | Description |
|---|---|
| `--settings <file-or-json>` | Path to a settings JSON file or a JSON string to load additional settings from |
| `--setting-sources <sources>` | Comma-separated list of setting sources to load: `user`, `project`, `local` |
| `-d`, `--debug [filter]` | Enable debug mode with optional category filtering (e.g., `"api,hooks"` or `"!statsig,!file"`) |
| `--debug-file <path>` | Write debug logs to a specific file path (implicitly enables debug mode) |
| `--verbose` | Override verbose mode setting from config |
| `--betas <betas...>` | Beta headers to include in API requests (API key users only) |
| `--mcp-debug` | _(Deprecated — use `--debug` instead)_ Enable MCP debug mode |

### 24.10 Subcommands

#### `claude mcp` — MCP Server Management

```
claude mcp add --transport http <name> <url>        # Add remote HTTP server
claude mcp add --transport sse <name> <url>          # Add remote SSE server
claude mcp add [opts] <name> -- <cmd> [args...]      # Add local stdio server
claude mcp add-json <name> '<json>'                  # Add from raw JSON config
claude mcp add-from-claude-desktop                   # Import from Claude Desktop
claude mcp list                                      # List configured servers
claude mcp get <name>                                # Show server details
claude mcp remove <name>                             # Remove a server
claude mcp serve                                     # Expose Claude Code as MCP server
claude mcp reset-project-choices                     # Reset MCP approval state
```

**`claude mcp add` flags:**

| Flag | Description |
|---|---|
| `-t`, `--transport <transport>` | Transport type: `stdio` (default), `sse`, `http` |
| `-H`, `--header <header...>` | Set WebSocket/HTTP headers, repeatable (e.g., `-H "Authorization: Bearer token"`) |
| `-e`, `--env <env...>` | Set environment variables for server process, repeatable (e.g., `-e KEY=value`) |
| `-s`, `--scope <scope>` | Configuration scope: `local` (default), `project`, `user` |

**`claude mcp serve` flags:**

| Flag | Description |
|---|---|
| `-d`, `--debug` | Enable debug mode |
| `--verbose` | Override verbose mode setting from config |

#### `claude plugin` — Plugin Management

```
claude plugin install <plugin>             # Install a plugin (use plugin@marketplace for specific source)
claude plugin uninstall <plugin>           # Uninstall a plugin (alias: remove)
claude plugin list                         # List installed plugins
claude plugin enable <plugin>              # Enable a disabled plugin
claude plugin disable [plugin]             # Disable an enabled plugin
claude plugin update <plugin>              # Update a plugin to latest version
claude plugin validate <path>              # Validate a plugin or marketplace manifest
claude plugin marketplace add <source>     # Add a marketplace from URL, path, or GitHub repo
claude plugin marketplace list             # List configured marketplaces
claude plugin marketplace remove <name>    # Remove a marketplace
claude plugin marketplace update [name]    # Update marketplace(s) from source
```

#### `claude doctor`

Check the health of the Claude Code auto-updater.

#### `claude install [target]`

Install Claude Code native build. Use `[target]` to specify version (`stable`, `latest`, or a specific version).

| Flag | Description |
|---|---|
| `--force` | Force installation even if already installed |

#### `claude setup-token`

Set up a long-lived authentication token (requires Claude subscription).

#### `claude update`

Check for updates and install if available.

### 24.11 Summary: Flags by Purpose

The following table highlights all flags specifically relevant to **adding context**, **extending functionality**, or **building workflows**:

| Purpose | Flags |
|---|---|
| **Adding context** | `--add-dir`, `--file`, `--system-prompt`, `--append-system-prompt` |
| **Custom agents** | `--agent`, `--agents` |
| **Plugins** | `--plugin-dir`, `claude plugin install` |
| **MCP servers** | `--mcp-config`, `--strict-mcp-config`, `claude mcp add` |
| **IDE/Browser integration** | `--ide`, `--chrome` |
| **Workflow control** | `--max-budget-usd`, `--permission-mode`, `--allowedTools`, `--disallowedTools`, `--tools` |
| **CI/Pipeline** | `-p` (print mode), `--output-format`, `--json-schema`, `--no-session-persistence`, `--replay-user-messages` |
| **Session continuity** | `-c`, `-r`, `--session-id`, `--fork-session`, `--from-pr` |

### 24.12 Common Patterns

**Multi-directory development:**
```bash
claude --add-dir ../shared-lib --add-dir ../common-types
```

**CI pipeline with structured output:**
```bash
claude -p "Analyze this PR for issues" \
  --output-format json \
  --max-budget-usd 2.00 \
  --allowedTools "Read" "Grep" "Glob"
```

**Custom agent with MCP tools:**
```bash
claude --mcp-config ./mcp-servers.json \
  --agents '{"db-reviewer":{"description":"Database migration reviewer","prompt":"Review SQL migrations for safety"}}' \
  --agent db-reviewer
```

**Locked-down execution:**
```bash
claude -p "Review code" \
  --permission-mode plan \
  --tools "Read,Grep,Glob" \
  --disallowedTools "Bash(rm *)" "Bash(curl *)"
```

**Appending project-specific instructions:**
```bash
claude --append-system-prompt "Always prefer functional patterns. Use Rust idioms. Never use unwrap() in production code."
```

---

## Appendix A: Data Flow Diagram

```
User Input
    │
    ▼
┌──────────┐     ┌──────────────┐     ┌─────────────┐
│  Ink UI  │────►│  Agent Loop  │────►│  Anthropic   │
│  (React) │     │              │     │  Messages    │
│          │◄────│  Tool Exec   │◄────│  API         │
└──────────┘     └──────┬───────┘     └─────────────┘
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
     ┌──────────┐ ┌──────────┐ ┌──────────┐
     │  Bash    │ │  File    │ │  MCP     │
     │  (exec) │ │  System  │ │  Servers │
     └──────────┘ └──────────┘ └──────────┘
```

---

## Appendix B: Key Internal Identifiers (Minified → Purpose)

| Minified | Purpose |
|---|---|
| `f6` | Global session state singleton |
| `U` | Zod schema builder |
| `BA()` | Get filesystem abstraction |
| `x1()` | Get project root |
| `d1()` | Get session ID |
| `ZZ()` | Get total cost |
| `h()` | Debug logger |
| `n()` | Event/telemetry logger |
| `G4()` | Feature flag check |
| `b2()` | Model shorthand resolver (`"opus"` → full model ID) |
| `eE()` | Sonnet model resolver (returns Sonnet 4.5 ID) |
| `wi()` | Opus model resolver (returns Opus 4.5 or 4.1 ID) |
| `lt6()` | Haiku model resolver (returns Haiku 4.5 ID) |
| `sO()` | Model map accessor (returns generation→ID dictionary) |
| `F4()` | Provider detection (`"firstParty"`, `"bedrock"`, `"vertex"`, `"foundry"`) |
| `fA1()` | Get current model shorthand (e.g., `"opus"`) |
| `q5()` | Get fully resolved model ID |
| `S66()` | Get default model based on plan tier |
| `M9()` | Ink render function |
| `R6()` | Execute git command |
| `qq` | React module reference |
| `k()` | Lazy module initializer |
| `o()` | ESM import helper |
| `v()` | CommonJS module wrapper |

---

*Document generated by analysis of the minified cli.js bundle v2.1.25. Actual source code structure may differ from the reconstructed architecture.*
