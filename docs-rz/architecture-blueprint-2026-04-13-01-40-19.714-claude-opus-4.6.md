# Claw Code — Project Architecture Blueprint

> **Generated:** 2026-04-13T01:40:19.714Z
> **Mode:** Rust
> **Repository:** ultraworkers/claw-code
> **Commit:** `0c51f83` (HEAD)
> **Codebase stats:** 787 commits · 5 contributors · 9 Rust crates · ~69,853 Rust LOC · 79 `.rs` files

---

## Table of Contents

1. [Architectural Overview](#1-architectural-overview)
2. [Architecture Visualization](#2-architecture-visualization)
3. [Core Architectural Components](#3-core-architectural-components)
4. [Architectural Layers and Dependencies](#4-architectural-layers-and-dependencies)
5. [Data Architecture](#5-data-architecture)
6. [Cross-Cutting Concerns](#6-cross-cutting-concerns)
7. [Service Communication Patterns](#7-service-communication-patterns)
8. [Rust Architectural Patterns](#8-rust-architectural-patterns)
9. [Implementation Patterns](#9-implementation-patterns)
10. [Testing Architecture](#10-testing-architecture)
11. [Deployment Architecture](#11-deployment-architecture)
12. [Extension and Evolution Patterns](#12-extension-and-evolution-patterns)
13. [Architectural Pattern Examples](#13-architectural-pattern-examples)
14. [Architectural Decision Records](#14-architectural-decision-records)
15. [Architecture Governance](#15-architecture-governance)
16. [Blueprint for New Development](#16-blueprint-for-new-development)

---

## 1. Architectural Overview

### What Claw Code Is

Claw Code is a **Rust CLI agent harness** — a terminal-based conversational AI tool that connects to LLM providers (Anthropic Claude, OpenAI-compatible, xAI Grok, Alibaba DashScope) and executes local tool operations (file I/O, bash, grep, git, MCP, LSP) on behalf of the model. It is the open-source Rust re-implementation of the `claw` CLI, purpose-built for autonomous software development workflows coordinated by multiple AI agents.

### Guiding Principles

The project philosophy (documented in `PHILOSOPHY.md`) drives these architectural choices:

1. **Clawable-first** — The harness is designed for machine operation, not human babysitting. State is machine-readable, recovery is automated, events are typed.
2. **State-machine lifecycle** — Every worker, session, MCP server, and plugin has explicit lifecycle states with typed events.
3. **Events over scraped prose** — Structured typed events (`LaneEvent`, `WorkerEvent`, `McpLifecyclePhase`) replace log scraping.
4. **Recovery before escalation** — Known failure modes auto-heal via `RecoveryRecipes` before asking for human help.
5. **Permission-gated execution** — All tool operations pass through a layered permission system (mode + enforcer + policy + prompt).
6. **Provider-agnostic API** — A unified `ProviderClient` enum abstracts Anthropic, OpenAI-compatible, and xAI wire formats.

### Primary Architectural Pattern

**Layered Architecture with Plugin/Registry Extensions** — The codebase is organized into clearly separated crates following a dependency-flow from leaf crates (telemetry, plugins) through mid-tier infrastructure (runtime, api, tools, commands) to the binary entry point (rusty-claude-cli). Global singleton registries (via `OnceLock`) bridge tool dispatch to stateful runtime subsystems.

---

## 2. Architecture Visualization

### High-Level System Context (C4 Level 1)

```
┌──────────────────────────────────────────────────┐
│                  Human / Orchestrator             │
│         (Terminal REPL · Discord · clawhip)       │
└──────────────────────┬───────────────────────────┘
                       │ CLI invocation / REPL input
                       ▼
┌──────────────────────────────────────────────────┐
│                  claw CLI Binary                  │
│              (rusty-claude-cli crate)             │
│  ┌──────────┐ ┌─────────┐ ┌──────────────────┐  │
│  │   REPL   │ │ One-shot│ │ Subcommands      │  │
│  │  Loop    │ │ Prompt  │ │ (doctor,status,..)│  │
│  └────┬─────┘ └────┬────┘ └────────┬─────────┘  │
│       └─────────────┼──────────────┘             │
│                     ▼                            │
│  ┌──────────────────────────────────────────┐    │
│  │        ConversationRuntime<C, T>         │    │
│  │   (model loop · hooks · compaction)      │    │
│  └──────┬──────────────┬────────────────────┘    │
│         │              │                         │
│    ┌────▼────┐   ┌─────▼─────┐                   │
│    │   API   │   │   Tools   │                   │
│    │ Client  │   │  Dispatch │                   │
│    └────┬────┘   └─────┬─────┘                   │
└─────────┼──────────────┼─────────────────────────┘
          │              │
    ┌─────▼─────┐  ┌─────▼──────────────────────┐
    │ LLM APIs  │  │ Local Filesystem / Bash /   │
    │ Anthropic │  │ Git / MCP Servers / LSP     │
    │ OpenAI    │  └────────────────────────────┘
    │ xAI/Grok  │
    │ DashScope │
    └───────────┘
```

### Crate Dependency Graph (C4 Level 2)

```
                    rusty-claude-cli (14,800 LOC)
                    ┌───────────────────────────┐
                    │  main.rs · init.rs         │
                    │  input.rs · render.rs      │
                    └─┬──┬──┬──┬──┬──┬──┬──┬────┘
                      │  │  │  │  │  │  │  │
          ┌───────────┘  │  │  │  │  │  │  └──────────────┐
          │     ┌────────┘  │  │  │  │  └──────┐          │
          │     │    ┌──────┘  │  │  └───┐     │          │
          ▼     ▼    ▼         ▼  ▼      ▼     ▼          ▼
        api  commands tools  runtime plugins compat   mock-anthropic
       7979   5279   9525   26550  3825   319      1100  (LOC)
        │       │      │       │     ▲      │          │
        │       │      │       │     │      │          │
        │       └──────┼───────┼─────┤      │          │
        │              │       │     │      └──────────┘
        │              ├───┐   │     │
        │              │   │   │     │
        ▼              ▼   ▼   ▼     │
     runtime ◄─────  api  plugins   │
        │              │            │
        ├──────────────┘            │
        ▼                           │
     telemetry (476 LOC)  plugins ◄─┘
```

**Explicit workspace dependency edges:**

| Crate | Depends On (workspace) |
|---|---|
| `api` | `runtime`, `telemetry` |
| `commands` | `plugins`, `runtime` |
| `tools` | `api`, `commands`, `plugins`, `runtime` |
| `runtime` | `plugins`, `telemetry` |
| `compat-harness` | `commands`, `tools`, `runtime` |
| `mock-anthropic-service` | `api` |
| `rusty-claude-cli` | `api`, `commands`, `compat-harness`, `runtime`, `plugins`, `tools`, `mock-anthropic-service` |
| `telemetry` | *(leaf — no workspace deps)* |
| `plugins` | *(leaf — no workspace deps)* |

### Data Flow Diagram

```
User Input (REPL / one-shot / pipe)
    │
    ▼
┌──────────────────────────────────────────┐
│ CLI Argument Parsing (clap-style manual) │
│ Config Resolution (user→project→local)   │
│ Auth Resolution (env vars → OAuth)       │
└──────────────┬───────────────────────────┘
               │
    ┌──────────▼───────────────────────────────────┐
    │         ConversationRuntime Turn Loop         │
    │                                               │
    │  1. Assemble system prompt + session history  │
    │  2. Send request to ProviderClient            │
    │  3. Stream response (SSE / chunked)           │
    │  4. Parse assistant events                    │
    │  5. For each tool_use:                        │
    │     a. Permission check (enforcer + policy)   │
    │     b. Pre-tool-use hook                      │
    │     c. Execute tool (dispatch by name)        │
    │     d. Post-tool-use hook                     │
    │     e. Append result to session               │
    │  6. Check auto-compaction threshold           │
    │  7. If more tool calls → iterate (go to 2)   │
    │  8. Return TurnSummary                        │
    └──────────────────────────────────────────────┘
               │
    ┌──────────▼──────────┐
    │ Session Persistence │
    │ (.claw/sessions/*.jsonl)  │
    └──────────────────────┘
```

---

## 3. Core Architectural Components

### 3.1 `rusty-claude-cli` — CLI Binary (14,800 LOC)

**Purpose:** The main entry point (`claw` binary). Owns the REPL, one-shot prompt mode, all direct CLI subcommands, streaming display rendering, and user interaction.

**Internal Structure:**

| Module | Responsibility |
|---|---|
| `main.rs` | Argument parsing, subcommand dispatch, REPL loop, session management, streaming display, provider/tool/permission wiring |
| `init.rs` | Repository initialization (`claw init` / `/init`) |
| `input.rs` | Readline/rustyline input handling, tab completion, history |
| `render.rs` | `TerminalRenderer` — ANSI markdown rendering, `Spinner`, `MarkdownStreamState` |

**Key constants:**

- `DEFAULT_MODEL` = `claude-opus-4-6`
- Default permission mode = `danger-full-access`
- Session files = `.claw/sessions/*.jsonl`
- `VERSION` from `CARGO_PKG_VERSION`

**Interaction patterns:**
- Creates `ProviderClient` based on model + auth source
- Creates `GlobalToolRegistry` with built-in + plugin + runtime tools
- Creates `ConversationRuntime<C, T>` parameterized over `ApiClient` + `ToolExecutor`
- Runs the turn loop, rendering streaming events to terminal with ANSI markdown
- Manages session resume, fork, compaction via `Session` and `SessionStore`

### 3.2 `runtime` — Core Runtime (26,550 LOC)

**Purpose:** The largest and most critical crate. Owns session persistence, conversation loop, config loading, permission evaluation, prompt assembly, MCP plumbing, file operations, bash execution, git context, hooks, and all stateful runtime subsystems.

**Module Map (43 modules):**

| Module | Responsibility |
|---|---|
| `conversation.rs` | `ConversationRuntime<C, T>` — the central model loop coordinating API calls, tool execution, hooks, compaction, and session updates |
| `session.rs` | `Session` — in-memory + persisted conversation state with JSONL append-only storage |
| `session_control.rs` | `SessionStore` — session listing, resume, fork, compaction orchestration |
| `config.rs` | `ConfigLoader`, `RuntimeConfig`, `RuntimeFeatureConfig` — multi-source config merge (user→project→local) |
| `config_validate.rs` | Config file validation and diagnostics |
| `permissions.rs` | `PermissionPolicy`, `PermissionMode`, `PermissionPrompter` trait — modal permission evaluation |
| `permission_enforcer.rs` | `PermissionEnforcer` — tool-level gating, workspace boundary checks, bash read-only heuristics |
| `prompt.rs` | `SystemPromptBuilder`, `ProjectContext` — system prompt assembly from instruction files + git context |
| `bash.rs` | `execute_bash()` — subprocess execution with timeout, background, and sandbox support |
| `bash_validation.rs` | Bash command validation submodules (read-only, destructive, sed, path, mode, command semantics) |
| `file_ops.rs` | `read_file`, `write_file`, `edit_file`, `glob_search`, `grep_search` — with binary detection, size limits, workspace boundary, symlink escape guards |
| `sandbox.rs` | Container detection (Docker/Podman), Linux namespace sandboxing, `unshare` capability probing |
| `hooks.rs` | `HookRunner` — lifecycle hooks (pre/post tool-use) with abort signals |
| `mcp.rs` | MCP naming, hashing, proxy URL helpers |
| `mcp_client.rs` | MCP client transport abstractions (stdio, SDK, remote, managed-proxy) |
| `mcp_stdio.rs` | `McpServerManager`, `McpStdioProcess` — stdio-based MCP server orchestration and JSON-RPC |
| `mcp_tool_bridge.rs` | `McpToolRegistry` — registry bridge for MCP tool dispatch |
| `mcp_server.rs` | `McpServer` — claw acting as an MCP server |
| `mcp_lifecycle_hardened.rs` | `McpLifecycleValidator` — degraded-mode detection and structured failure reporting |
| `lsp_client.rs` | `LspRegistry` — LSP protocol dispatch (diagnostics, hover, definition, references, completion, symbols, formatting) |
| `task_registry.rs` | `TaskRegistry` — in-memory task lifecycle management (create, get, list, stop, update, output) |
| `team_cron_registry.rs` | `TeamRegistry`, `CronRegistry` — team and scheduled job management |
| `worker_boot.rs` | `WorkerRegistry`, `Worker`, `WorkerStatus` — worker lifecycle state machine |
| `lane_events.rs` | `LaneEvent`, `LaneEventName` — typed events for parallel development lanes |
| `policy_engine.rs` | `PolicyEngine` — rule evaluation for merge/retry/rebase/escalation policies |
| `recovery_recipes.rs` | `RecoveryRecipe`, `attempt_recovery()` — automated failure recovery |
| `plugin_lifecycle.rs` | `PluginLifecycle` — plugin health, discovery, degraded-mode reporting |
| `stale_branch.rs` | `check_freshness()`, `StaleBranchPolicy` — branch freshness enforcement |
| `stale_base.rs` | `check_base_commit()` — base commit drift detection |
| `branch_lock.rs` | `detect_branch_lock_collisions()` — branch lock collision detection |
| `task_packet.rs` | `TaskPacket`, `validate_packet()` — task payload validation |
| `summary_compression.rs` | `compress_summary_text()` — output/summary compression |
| `green_contract.rs` | Green-level contract verification |
| `trust_resolver.rs` | Trust prompt auto-resolution (test-only) |
| `git_context.rs` | `GitContext` — git status, diff, commit log context extraction |
| `compact.rs` | `compact_session()`, `should_compact()` — session compaction logic |
| `usage.rs` | `UsageTracker`, `ModelPricing` — token counting and cost estimation |
| `oauth.rs` | OAuth PKCE flow, credential storage/loading |
| `remote.rs` | Remote session context, upstream proxy bootstrap |
| `bootstrap.rs` | `BootstrapPlan`, `BootstrapPhase` — project onboarding |
| `sse.rs` | `IncrementalSseParser` — incremental Server-Sent Events parsing |
| `json.rs` | JSON value wrapper utilities |

### 3.3 `api` — Provider Clients (7,979 LOC)

**Purpose:** Multi-provider API client layer. Handles authentication, SSE streaming, request construction, model aliasing, and prompt caching.

**Key Types:**

| Type | Purpose |
|---|---|
| `ProviderClient` | Enum wrapping `AnthropicClient`, `OpenAiCompatClient` (for xAI and OpenAI) |
| `AnthropicClient` | Anthropic Messages API client with SSE streaming + prompt caching |
| `OpenAiCompatClient` | OpenAI-compatible chat completions client (also used for xAI, DashScope, Ollama, OpenRouter) |
| `AuthSource` | Enum: `ApiKey(String)` or `BearerToken(String)` |
| `ProviderKind` | Enum: `Anthropic`, `Xai`, `OpenAi` |
| `MessageRequest` / `MessageResponse` | Wire-format request/response types |
| `StreamEvent` | Parsed SSE stream events (message_start, content_block_delta, etc.) |
| `PromptCache` | Anthropic-specific prompt caching with automatic cache-control breakpoints |
| `ProxyConfig` | HTTP/HTTPS proxy configuration |
| `SseParser` | Frame-level SSE parser |

**Module map:**

| Module | Purpose |
|---|---|
| `client.rs` | `ProviderClient` enum, model-to-provider routing, auth resolution |
| `providers/anthropic.rs` | Anthropic Messages API — SSE streaming, `x-api-key` / `Authorization: Bearer` auth |
| `providers/openai_compat.rs` | OpenAI-compatible `/v1/chat/completions` — tool-call normalization, reasoning model param stripping |
| `providers/mod.rs` | Model alias table, `detect_provider_kind()`, `max_tokens_for_model()` |
| `http_client.rs` | `build_http_client()` with proxy support |
| `prompt_cache.rs` | Cache-control breakpoint injection and hit/miss tracking |
| `sse.rs` | SSE frame parser |
| `types.rs` | Wire-format types for Messages API and stream events |
| `error.rs` | `ApiError` type |

### 3.4 `tools` — Tool Specs & Execution (9,525 LOC)

**Purpose:** Defines all 40 built-in tool specifications and the `execute_tool()` dispatcher. Bridges static tool definitions to runtime execution.

**Key Types:**

| Type | Purpose |
|---|---|
| `ToolSpec` | Static tool definition: name, description, JSON schema, required permission |
| `GlobalToolRegistry` | Merges built-in, plugin, and runtime tools; handles permission enforcement |
| `RuntimeToolDefinition` | Dynamic runtime-contributed tool definition |
| `ToolManifestEntry` / `ToolSource` | Tool provenance tracking (Base vs Conditional) |
| `ToolSearchOutput` | Tool search/discovery result |

**Tool dispatch flow:**

```
execute_tool(name, input, cwd, permission_mode, ..)
    │
    ├── "bash" → execute_bash(BashCommandInput)
    ├── "read_file" → read_file(path, range)
    ├── "write_file" → write_file(path, content, cwd)
    ├── "edit_file" → edit_file(path, old, new, cwd)
    ├── "glob_search" → glob_search(pattern, cwd)
    ├── "grep_search" → grep_search(GrepSearchInput)
    ├── "TaskCreate/Get/List/Stop/Update/Output"
    │       → global_task_registry()
    ├── "TeamCreate/TeamDelete"
    │       → global_team_registry()
    ├── "CronCreate/CronDelete/CronList"
    │       → global_cron_registry()
    ├── "LSP" → global_lsp_registry().dispatch()
    ├── "MCP/ListMcpResources/ReadMcpResource/McpAuth"
    │       → global_mcp_registry()
    ├── "WebFetch" / "WebSearch" → HTTP-based
    ├── "Agent" → sub-agent spawning
    ├── "TodoWrite" → todo file management
    ├── "Skill" → skill resolution + execution
    └── ... (40 total tool specs)
```

**Global singleton registries** (via `OnceLock`):
- `global_task_registry()` → `TaskRegistry`
- `global_team_registry()` → `TeamRegistry`
- `global_cron_registry()` → `CronRegistry`
- `global_lsp_registry()` → `LspRegistry`
- `global_mcp_registry()` → `McpToolRegistry`
- `global_worker_registry()` → `WorkerRegistry`

### 3.5 `commands` — Slash Commands (5,279 LOC)

**Purpose:** Defines the slash command registry for the REPL, including parsing, dispatch, help rendering, and JSON/text output formatting.

**Key types:**

| Type | Purpose |
|---|---|
| `SlashCommand` | Enum of all slash commands |
| `SkillSlashDispatch` | Skill invocation routing |

**Slash command surface:** `/help`, `/status`, `/cost`, `/resume`, `/session`, `/config`, `/memory`, `/diff`, `/commit`, `/pr`, `/issue`, `/export`, `/hooks`, `/files`, `/doctor`, `/mcp`, `/agents`, `/skills`, `/tasks`, `/plugin`, `/subagent`, `/team`, `/telemetry`, `/providers`, `/cron`, `/review`, `/advisor`, `/insights`, `/security-review`, `/desktop`, `/release-notes`, and more.

### 3.6 `plugins` — Plugin System (3,825 LOC)

**Purpose:** Plugin metadata, install/enable/disable/uninstall lifecycle, plugin tool definitions, hook integration.

**Module map:**

| Module | Purpose |
|---|---|
| `lib.rs` | `PluginManager`, `PluginRegistry`, `PluginTool`, `PluginHooks`, `PluginManagerConfig` |
| `hooks.rs` | Plugin hook integration (pre/post tool-use hooks from plugins) |
| `test_isolation.rs` | Test isolation utilities for plugin tests |

**Plugin lifecycle:** `install(path)` → `enable(name)` → `disable(name)` → `uninstall(id)` / `update(id)`

### 3.7 `telemetry` — Session Tracing (476 LOC)

**Purpose:** Lightweight telemetry types — session trace events, analytics events, and sink abstractions.

**Key types:**

| Type | Purpose |
|---|---|
| `SessionTracer` | Collects session trace records |
| `TelemetrySink` | Trait for telemetry output destinations |
| `JsonlTelemetrySink` | JSONL file sink |
| `MemoryTelemetrySink` | In-memory sink (for testing) |
| `TelemetryEvent` / `AnalyticsEvent` | Event payloads |
| `AnthropicRequestProfile` | Request-level profiling data |
| `DEFAULT_ANTHROPIC_VERSION` | API version constant |

### 3.8 `mock-anthropic-service` — Mock Server (1,100 LOC)

**Purpose:** Deterministic Anthropic-compatible HTTP mock service for end-to-end parity testing. Serves scripted `/v1/messages` responses with SSE streaming.

### 3.9 `compat-harness` — Manifest Extraction (319 LOC)

**Purpose:** Extracts tool and prompt manifests from the upstream TypeScript source for parity comparison.

---

## 4. Architectural Layers and Dependencies

### Layer Diagram

```
Layer 4: Presentation / CLI
    ┌─────────────────────────────────┐
    │       rusty-claude-cli          │  REPL, rendering, CLI args
    └─────────┬───────────────────────┘
              │ depends on ▼
Layer 3: Orchestration / Dispatch
    ┌─────────┴───────────────────────┐
    │  tools  │  commands  │ compat   │  Tool dispatch, slash cmds
    └─────────┬───────────────────────┘
              │ depends on ▼
Layer 2: Core Runtime / API
    ┌─────────┴───────────────────────┐
    │  runtime  │  api                │  Conversation loop, providers,
    │           │                     │  permissions, sessions, config
    └─────────┬───────────────────────┘
              │ depends on ▼
Layer 1: Foundation / Leaf
    ┌─────────┴───────────────────────┐
    │  plugins  │  telemetry          │  Plugin metadata, telemetry types
    └─────────────────────────────────┘
```

### Dependency Rules

1. **Layer 1 (Foundation)** — `plugins` and `telemetry` are leaf crates with zero workspace dependencies.
2. **Layer 2 (Core)** — `runtime` depends on `plugins` + `telemetry`; `api` depends on `runtime` + `telemetry`.
3. **Layer 3 (Orchestration)** — `tools` depends on `api` + `commands` + `plugins` + `runtime`; `commands` depends on `plugins` + `runtime`.
4. **Layer 4 (Presentation)** — `rusty-claude-cli` depends on all other crates.
5. **Test/dev crates** — `mock-anthropic-service` depends only on `api`; `compat-harness` depends on `commands` + `tools` + `runtime`.

### Layer Separation Mechanisms

- **Trait boundaries:** `ApiClient` trait in runtime, implemented by the cli; `ToolExecutor` trait in runtime, implemented by tools. `PermissionPrompter` trait decouples permission UI from policy logic.
- **No upward dependencies:** Runtime never imports from tools, commands, or the CLI binary.
- **Workspace `[workspace.lints]`:** `unsafe_code = "forbid"` globally; `clippy::all` + `clippy::pedantic` at warn level.

### Known Architectural Notes

- `api` depends on `runtime` — this creates a layer-2 circular potential. In practice, `api` uses `runtime` for telemetry types and config, not for the conversation loop.
- Global registries (`OnceLock` singletons in `tools/src/lib.rs`) are a pragmatic shortcut for session-scoped state; they prevent passing registries through deeply nested call chains.

---

## 5. Data Architecture

### Domain Model

```
Session
├── session_id: String
├── messages: Vec<ConversationMessage>
│   ├── role: MessageRole (System | User | Assistant | Tool)
│   └── blocks: Vec<ContentBlock>
│       ├── Text { text }
│       ├── ToolUse { id, name, input }
│       └── ToolResult { tool_use_id, tool_name, output, is_error }
├── usage: Option<TokenUsage>
├── compaction: Option<SessionCompaction>
├── fork: Option<SessionFork>
└── prompt_entries: Vec<SessionPromptEntry>

ConversationRuntime<C: ApiClient, T: ToolExecutor>
├── session: Session
├── api_client: C
├── tool_executor: T
├── permission_policy: PermissionPolicy
├── system_prompt: Vec<String>
├── usage_tracker: UsageTracker
├── hook_runner: HookRunner
└── session_tracer: Option<SessionTracer>

RuntimeConfig
├── merged: BTreeMap<String, JsonValue>
├── loaded_entries: Vec<ConfigEntry>
└── feature_config: RuntimeFeatureConfig
    ├── hooks: RuntimeHookConfig
    ├── plugins: RuntimePluginConfig
    ├── mcp: McpConfigCollection
    ├── oauth: Option<OAuthConfig>
    ├── model: Option<String>
    ├── aliases: BTreeMap<String, String>
    ├── permission_mode: Option<ResolvedPermissionMode>
    ├── permission_rules: RuntimePermissionRuleConfig
    ├── sandbox: SandboxConfig
    └── provider_fallbacks: ProviderFallbackConfig
```

### Data Access Patterns

| Pattern | Implementation |
|---|---|
| **Session persistence** | Append-only JSONL files under `.claw/sessions/`. `Session` writes each message as a JSON line. Rotation at 256 KB with up to 3 rotated files. |
| **Config resolution** | `ConfigLoader::discover()` loads user (`~/.claw.json`, `~/.config/claw/settings.json`) → project (`.claw.json`, `.claw/settings.json`) → local (`.claw/settings.local.json`), merging with later entries overriding earlier ones. |
| **Global registries** | `OnceLock`-based singletons for `TaskRegistry`, `TeamRegistry`, `CronRegistry`, `LspRegistry`, `McpToolRegistry`, `WorkerRegistry`. Thread-safe via `Mutex`/`RwLock` internals. |
| **Instruction files** | `prompt.rs` discovers `CLAUDE.md`, `.claw/instructions.md` and similar files, truncated to `MAX_INSTRUCTION_FILE_CHARS` (4,000) and total `MAX_TOTAL_INSTRUCTION_CHARS` (12,000). |
| **OAuth credentials** | Stored at `credentials_path()`, loaded/saved as JSON with PKCE support. |

### Data Validation

- **File operations:** NUL-byte binary detection, `MAX_READ_SIZE`, `MAX_WRITE_SIZE`, canonical workspace-boundary validation, symlink escape prevention.
- **Task packets:** `validate_packet()` in `task_packet.rs` validates task payloads before dispatch.
- **Config files:** `config_validate.rs` provides `validate_config_file()` with structured `ConfigDiagnostic` results.

---

## 6. Cross-Cutting Concerns

### 6.1 Permission System

The permission system is layered with three tiers:

```
Tier 1: PermissionMode (session-level)
    ReadOnly | WorkspaceWrite | DangerFullAccess | Prompt | Allow

Tier 2: PermissionPolicy (rule-based)
    ├── active_mode: PermissionMode
    ├── allow_rules: Vec<String>
    ├── deny_rules: Vec<String>
    └── ask_rules: Vec<String>
    → authorize() → PermissionOutcome (Allow | Deny)

Tier 3: PermissionEnforcer (tool-specific)
    ├── check() → EnforcementResult
    ├── check_file_write() → workspace boundary + read-only denial
    └── check_bash() → mutating command detection in read-only mode

Context: PermissionContext (hook overrides)
    └── override_decision: Allow | Deny | Ask
```

**Flow:** Each tool spec declares a `required_permission: PermissionMode`. Before execution, the enforcer checks the tool against the active session mode. If the requirement exceeds the mode, the `PermissionPrompter` trait is invoked for interactive approval.

### 6.2 Error Handling & Resilience

| Concern | Implementation |
|---|---|
| **API errors** | `ApiError` in the `api` crate — wraps HTTP errors, auth failures, rate limits, deserialization failures |
| **Runtime errors** | `RuntimeError` — conversation turn failures |
| **Tool errors** | `ToolError` — local tool execution failures, returned as `is_error: true` tool results to the model |
| **Recovery** | `RecoveryRecipes` — automated recipe-based recovery for known failure patterns (`FailureScenario` → `RecoveryRecipe` → `RecoveryStep` chain) |
| **MCP resilience** | `McpLifecycleValidator` detects degraded mode, partial startup, handshake failures with structured `McpDegradedReport` |
| **Provider fallbacks** | `ProviderFallbackConfig` — ordered fallback chain for retryable provider errors (429/500/503) |
| **Session robustness** | `PoisonError::into_inner` on test env lock to recover from panicked tests; session counter uses `AtomicU64` |

### 6.3 Logging & Telemetry

| Component | Purpose |
|---|---|
| `SessionTracer` | Records per-session trace events |
| `TelemetrySink` trait | Pluggable output: `JsonlTelemetrySink` (file), `MemoryTelemetrySink` (test) |
| `TelemetryEvent` / `AnalyticsEvent` | Typed event payloads |
| `AnthropicRequestProfile` | Per-request profiling (latency, tokens, cache hits) |
| `UsageTracker` | Cumulative token usage + USD cost estimation per model |

### 6.4 Hooks System

| Hook | Trigger |
|---|---|
| `pre_tool_use` | Before every tool execution |
| `post_tool_use` | After every successful tool execution |
| `post_tool_use_failure` | After tool execution failure |

Hooks are configured via `RuntimeHookConfig` (from settings files). `HookRunner` executes them as external commands with `HookAbortSignal` for cancellation and `HookProgressReporter` for status feedback. Hooks can provide `PermissionOverride` (Allow/Deny/Ask) via `PermissionContext`.

### 6.5 Configuration Management

**Resolution order (later overrides earlier):**

1. `~/.claw.json`
2. `~/.config/claw/settings.json`
3. `<repo>/.claw.json`
4. `<repo>/.claw/settings.json`
5. `<repo>/.claw/settings.local.json`

**Legacy fallback paths:** `.codex/` directories and `CODEX_HOME` env var are scanned alongside `.claw/` paths.

**Feature toggles:** `RuntimeFeatureConfig` consolidates all parsed config into typed fields: hooks, plugins, MCP servers, OAuth, model, aliases, permission mode, permission rules, sandbox config, provider fallbacks, trusted roots.

### 6.6 Sandbox / Container Detection

`sandbox.rs` detects container environments via:
- `/.dockerenv`, `/run/.containerenv` file markers
- `CONTAINER`, `container`, `KUBERNETES_SERVICE_HOST` env vars
- `/proc/1/cgroup` hints
- `unshare` capability probing (not binary existence)

`SandboxStatus` reports isolation mode: `None`, `Container`, `LinuxNamespace`. `FilesystemIsolationMode` is derived from sandbox status.

---

## 7. Service Communication Patterns

### LLM Provider Communication

| Provider | Protocol | Endpoint | Auth Header |
|---|---|---|---|
| Anthropic | Messages API + SSE streaming | `https://api.anthropic.com/v1/messages` | `x-api-key` or `Authorization: Bearer` |
| xAI | OpenAI-compatible Chat Completions | `https://api.x.ai/v1/chat/completions` | `Authorization: Bearer` |
| OpenAI / OpenRouter / Ollama | OpenAI-compatible Chat Completions | configurable base URL | `Authorization: Bearer` |
| DashScope (Alibaba) | OpenAI-compatible | `https://dashscope.aliyuncs.com/compatible-mode/v1` | `Authorization: Bearer` |

**Streaming:** All providers use SSE (Server-Sent Events). `SseParser` handles frame parsing; `IncrementalSseParser` handles incremental buffering. The Anthropic client emits typed `StreamEvent` variants; the OpenAI-compat client normalizes to the same event model.

**Model routing:** `detect_provider_kind(model_name)` uses prefix matching (`claude` → Anthropic, `grok` → xAI, `qwen-`/`qwen/` → DashScope, else → OpenAI-compat fallback). Model aliases (`opus` → `claude-opus-4-6`) resolve through `resolve_model_alias()`.

### MCP Communication

| Transport | Implementation |
|---|---|
| stdio | `McpStdioProcess` — spawns child process, JSON-RPC over stdin/stdout |
| SDK | `McpSdkTransport` — SDK-based connection |
| Remote | `McpRemoteTransport` — remote server connection |
| Managed Proxy | `McpManagedProxyTransport` — proxy-mediated connection |

`McpServerManager` orchestrates server lifecycle. `McpToolRegistry` bridges tool dispatch. JSON-RPC protocol with `initialize`, `tools/list`, `tools/call`, `resources/list`, `resources/read`.

### HTTP Proxy Support

Standard `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` env vars. `ProxyConfig` also supports a unified `proxy_url` field that overrides per-scheme settings. `build_http_client_with()` configures `reqwest` with proxy settings.

---

## 8. Rust Architectural Patterns

### 8.1 Crate Organization

The workspace uses a **flat crate layout** under `rust/crates/`. Each crate has a focused responsibility. The workspace `Cargo.toml` configures:
- `resolver = "2"` (Rust 2021 resolver)
- `edition = "2021"`
- `unsafe_code = "forbid"` globally
- `clippy::all` + `clippy::pedantic` at warn level

### 8.2 Generic Trait-Based Runtime

The central `ConversationRuntime<C, T>` is parameterized over two traits:

```rust
pub trait ApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>;
}

pub trait ToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError>;
}
```

This enables:
- **Production:** `ProviderClient` as `ApiClient`, `GlobalToolRegistry` + `execute_tool()` as `ToolExecutor`
- **Testing:** Mock API clients and tool executors for deterministic tests

### 8.3 Global Singleton Registries

Stateful subsystems (tasks, teams, cron, LSP, MCP, workers) use `OnceLock`-based singletons:

```rust
fn global_task_registry() -> &'static TaskRegistry {
    use std::sync::OnceLock;
    static REGISTRY: OnceLock<TaskRegistry> = OnceLock::new();
    REGISTRY.get_or_init(TaskRegistry::new)
}
```

Internal thread safety via `Mutex` or `RwLock` within each registry. This pattern avoids threading registries through deep call stacks while maintaining safety.

### 8.4 Enum-Based Provider Dispatch

```rust
pub enum ProviderClient {
    Anthropic(AnthropicClient),
    Xai(OpenAiCompatClient),
    OpenAi(OpenAiCompatClient),
}
```

Each variant wraps a provider-specific client. Shared operations (`stream`, `prompt_cache_stats`) dispatch through `match self`. The `OpenAiCompatClient` is reused for both xAI and OpenAI-compatible providers with different `OpenAiCompatConfig`.

### 8.5 Builder Pattern

- `ConversationRuntime::new().with_max_iterations().with_hook_abort_signal().with_session_tracer()`
- `GlobalToolRegistry::builtin().with_plugin_tools().with_runtime_tools().with_enforcer()`
- `SystemPromptBuilder` for prompt assembly

### 8.6 Memory and Safety Patterns

| Pattern | Usage |
|---|---|
| `unsafe_code = "forbid"` | No unsafe code anywhere in the workspace |
| `Arc<Mutex<T>>` / `RwLock` | Thread-safe shared state in registries |
| `OnceLock` | Lazy singleton initialization |
| `AtomicU64` | Lock-free session ID counter and timestamp tracking |
| `PoisonError::into_inner` | Resilient mutex recovery after test panics |
| `BTreeMap` / `BTreeSet` | Deterministic ordering for reproducible output |

### 8.7 Asynchronous Patterns

The codebase is **synchronous** (blocking I/O with `reqwest::blocking`). There is no `async`/`await` or tokio runtime. This is a deliberate choice:
- Simpler mental model for a CLI tool
- No async runtime overhead
- `mpsc` channels for background heartbeats/progress
- Thread spawning (`std::thread`) for parallel operations

---

## 9. Implementation Patterns

### 9.1 Tool Implementation Pattern

Every built-in tool follows this pattern:

1. **Spec definition** in `mvp_tool_specs()` — name, description, JSON schema, required permission
2. **Dispatch** in `execute_tool()` — match on tool name string
3. **Execution** — call into `runtime` crate functions (`execute_bash`, `read_file`, etc.)
4. **Result** — return JSON string for success or `ToolError` for failure

```rust
// Pattern: Tool spec
ToolSpec {
    name: "read_file",
    description: "Read a file from the filesystem",
    input_schema: json!({ "type": "object", "properties": { "path": { "type": "string" } }, "required": ["path"] }),
    required_permission: PermissionMode::ReadOnly,
}

// Pattern: Tool dispatch
"read_file" => {
    let path = extract_string(&input_val, "path")?;
    let output = read_file(path, range)?;
    Ok(serde_json::to_string(&output)?)
}
```

### 9.2 Slash Command Pattern

```rust
// In commands/src/lib.rs
pub enum SlashCommand {
    Help,
    Status,
    Doctor,
    Plugin(PluginSubcommand),
    // ...
}
```

Commands support both text and JSON output modes. Plugin-related commands delegate to `PluginManager`.

### 9.3 Config Merge Pattern

```rust
// ConfigLoader::discover() merges in order:
// 1. User: ~/.claw.json, ~/.config/claw/settings.json
// 2. Project: <repo>/.claw.json, <repo>/.claw/settings.json
// 3. Local: <repo>/.claw/settings.local.json
// Later entries override earlier entries (BTreeMap merge)
```

### 9.4 Session Persistence Pattern

```rust
// Append-only JSONL:
// - Each message serialized as one JSON line
// - File rotation at 256 KB (ROTATE_AFTER_BYTES)
// - Up to 3 rotated files (MAX_ROTATED_FILES)
// - Monotonic timestamps via AtomicU64
// - Session ID counter via AtomicU64
```

### 9.5 Permission Enforcement Pattern

```rust
// Three-layer check:
// 1. PermissionEnforcer::check(tool_name, input)
//    → workspace boundary, bash read-only heuristics
// 2. PermissionPolicy::authorize(request)
//    → mode comparison + allow/deny/ask rules
// 3. PermissionPrompter::decide(request)
//    → interactive user approval (if needed)
```

### 9.6 Recovery Recipe Pattern

```rust
pub struct RecoveryRecipe {
    scenario: FailureScenario,
    steps: Vec<RecoveryStep>,
    escalation: EscalationPolicy,
}

// attempt_recovery() matches the current failure scenario
// to a recipe, executes steps in order, and escalates if
// all steps fail.
```

---

## 10. Testing Architecture

### Test Strategy

| Level | Implementation |
|---|---|
| **Unit tests** | `#[cfg(test)]` modules within source files; extensive use of mock implementations |
| **Integration tests** | `rust/crates/rusty-claude-cli/tests/mock_parity_harness.rs` — end-to-end CLI tests against the mock Anthropic service |
| **Parity tests** | `mock_parity_scenarios.json` + `run_mock_parity_diff.py` — scenario-to-PARITY.md mapping and verification |
| **Test infrastructure** | `mock-anthropic-service` crate provides a deterministic `/v1/messages` server |

### Mock Parity Harness

10 scripted scenarios covering core behavioral paths:

| Scenario | Coverage |
|---|---|
| `streaming_text` | Basic SSE streaming response |
| `read_file_roundtrip` | File read tool execution |
| `grep_chunk_assembly` | Chunked grep output handling |
| `write_file_allowed` | Write tool with permission granted |
| `write_file_denied` | Write tool with permission denied |
| `multi_tool_turn_roundtrip` | Multiple tool calls in one turn |
| `bash_stdout_roundtrip` | Bash execution and output capture |
| `bash_permission_prompt_approved` | Bash with permission prompt → approve |
| `bash_permission_prompt_denied` | Bash with permission prompt → deny |
| `plugin_tool_roundtrip` | Plugin tool execution path |

### Test Isolation

- `test_env_lock()` in `runtime/src/lib.rs` — global `Mutex` preventing concurrent env-var mutation
- `PoisonError::into_inner` — prevents cascading test failures from poisoned locks
- `plugins/src/test_isolation.rs` — plugin test isolation utilities

### CI Pipeline

**`.github/workflows/rust-ci.yml`** — Runs on push/PR to `main`:

| Job | Command |
|---|---|
| `doc-source-of-truth` | Python script checking docs for stale branding |
| `cargo fmt` | `cargo fmt --all --check` |
| `cargo test --workspace` | Full workspace test suite |
| `cargo clippy --workspace` | Clippy lint check |

**Concurrency:** Groups cancel in-progress runs for the same branch/PR.

---

## 11. Deployment Architecture

### Binary Distribution

**Binary name:** `claw` (`claw.exe` on Windows)

**Build modes:**
- `cargo build --workspace` — debug build
- `cargo build --release -p rusty-claude-cli` — release build

**Release workflow (`.github/workflows/release.yml`):**

| Target | Runner | Artifact |
|---|---|---|
| `linux-x64` | `ubuntu-latest` | `claw-linux-x64` |
| `macos-arm64` | `macos-14` | `claw-macos-arm64` |

Triggered by `v*` tags. Artifacts uploaded to GitHub Releases via `softprops/action-gh-release@v2`.

### Container Support

**`Containerfile`** — `rust:bookworm` base with `git`, `libssl-dev`, `pkg-config`, `ca-certificates`. Bind-mounts the repo at `/workspace` (no COPY). Supports both Docker and Podman.

**`.devcontainer/devcontainer.json`** — VS Code dev container with `mcr.microsoft.com/devcontainers/rust:1`, Python 3.13, Node 24, GitHub CLI, Copilot CLI, rust-analyzer, and debugging extensions.

### Install Script

**`install.sh`** — Cross-platform installer (Linux, macOS, WSL). Detects OS, verifies Rust toolchain, builds workspace, runs verification. Supports `--release`, `--no-verify`, `--debug` flags.

### Runtime Dependencies

| Dependency | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` or `ANTHROPIC_AUTH_TOKEN` | Anthropic API authentication |
| `OPENAI_API_KEY` + `OPENAI_BASE_URL` | OpenAI-compatible providers |
| `XAI_API_KEY` | xAI Grok |
| `DASHSCOPE_API_KEY` | Alibaba DashScope |
| `git` | Git context, diff, commit operations |
| `unshare` (Linux) | Optional namespace sandboxing |

---

## 12. Extension and Evolution Patterns

### Adding a New Tool

1. **Define the tool spec** in `tools/src/lib.rs` → `mvp_tool_specs()`:
   ```rust
   ToolSpec {
       name: "my_tool",
       description: "What it does",
       input_schema: json!({ ... }),
       required_permission: PermissionMode::WorkspaceWrite,
   }
   ```
2. **Add dispatch** in `execute_tool()`:
   ```rust
   "my_tool" => { /* implementation */ }
   ```
3. **Implement execution** — either inline or in a new `runtime` module.
4. **Add tests** — unit test in the module, integration test in mock parity harness if applicable.

### Adding a New Provider

1. **Create a new client** in `api/src/providers/` (implement streaming, auth, request construction).
2. **Add a variant** to `ProviderKind` and `ProviderClient`.
3. **Add detection logic** in `detect_provider_kind()`.
4. **Add model aliases** if needed.
5. **Wire auth** — add env var handling in `ProviderClient::from_model()`.

### Adding a New Slash Command

1. **Add the variant** to `SlashCommand` enum in `commands/src/lib.rs`.
2. **Add parsing** in the command classifier.
3. **Add execution** in the REPL dispatch (`main.rs`).
4. **Add help text** and JSON/text rendering.

### Adding a New Plugin

1. Plugin metadata and tool definitions go through `PluginManager`.
2. Plugin tools must not conflict with built-in tool names (enforced by `GlobalToolRegistry::with_plugin_tools()`).
3. Plugins can contribute hooks (pre/post tool-use).

### Adding a New Runtime Subsystem

1. **Create a new module** in `runtime/src/`.
2. **Export types** from `runtime/src/lib.rs`.
3. **If stateful:** Create a registry with `Mutex`/`RwLock` internals.
4. **If tool-facing:** Add a `global_*_registry()` singleton in `tools/src/lib.rs`.
5. **Wire into tool dispatch** in `execute_tool()`.

---

## 13. Architectural Pattern Examples

### Layer Separation: Trait-Based API Client

```rust
// runtime/src/conversation.rs — defines the contract
pub trait ApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>;
}

// api/src/client.rs — implements it (via adapter in CLI)
pub enum ProviderClient {
    Anthropic(AnthropicClient),
    Xai(OpenAiCompatClient),
    OpenAi(OpenAiCompatClient),
}

// rusty-claude-cli/src/main.rs — wires them together
let runtime = ConversationRuntime::new(
    session, api_client, tool_executor, permission_policy, system_prompt
);
```

### Tool Dispatch: Registry + Match

```rust
// tools/src/lib.rs — global singleton
fn global_task_registry() -> &'static TaskRegistry {
    static REGISTRY: OnceLock<TaskRegistry> = OnceLock::new();
    REGISTRY.get_or_init(TaskRegistry::new)
}

// Tool dispatch
pub fn execute_tool(name: &str, input: &str, ...) -> Result<String, ToolError> {
    match name {
        "TaskCreate" => {
            let reg = global_task_registry();
            let task = reg.create(parsed_input.name, parsed_input.description)?;
            Ok(serde_json::to_string(&task)?)
        }
        // ...
    }
}
```

### Permission Enforcement: Layered Check

```rust
// Before tool execution:
// 1. Enforcer checks tool-specific rules
let enforcement = enforcer.check(tool_name, &input_value);
match enforcement {
    EnforcementResult::Denied(reason) => return Err(ToolError::new(reason)),
    EnforcementResult::Allowed => { /* proceed */ }
}

// 2. Policy checks session-level mode vs required permission
let outcome = policy.authorize(&PermissionRequest {
    tool_name, input, current_mode, required_mode, reason
});
match outcome {
    PermissionOutcome::Allow => { /* execute */ }
    PermissionOutcome::Deny { reason } => return Err(ToolError::new(reason)),
}
```

### Config Merge: Precedence Chain

```rust
// runtime/src/config.rs
impl ConfigLoader {
    pub fn discover(cwd: &Path) -> Result<RuntimeConfig, ConfigError> {
        let entries = vec![
            // User scope
            resolve_user_config("~/.claw.json"),
            resolve_user_config("~/.config/claw/settings.json"),
            // Project scope
            resolve_project_config(cwd, ".claw.json"),
            resolve_project_config(cwd, ".claw/settings.json"),
            // Local scope (gitignored)
            resolve_project_config(cwd, ".claw/settings.local.json"),
        ];
        // Merge: later entries override earlier
        merge_configs(entries)
    }
}
```

---

## 14. Architectural Decision Records

### ADR-1: Rust Over Python/TypeScript

**Context:** The original implementation was Python/TypeScript. Performance, safety, and binary distribution motivated a rewrite.

**Decision:** Full Rust rewrite with `unsafe_code = "forbid"`.

**Consequences:**
- (+) Single static binary with no runtime dependencies
- (+) Memory safety guaranteed at compile time
- (+) Excellent cross-platform support
- (-) Steeper learning curve for contributors
- (-) Longer compile times

### ADR-2: Synchronous I/O (No Async Runtime)

**Context:** CLI tools can use async for concurrent I/O, but add complexity.

**Decision:** Use `reqwest::blocking` and `std::thread` instead of `tokio`/`async-std`.

**Consequences:**
- (+) Simpler code, easier debugging
- (+) No colored function problem
- (+) No runtime overhead
- (-) Cannot easily parallelize multiple provider calls
- (-) Thread-per-connection for MCP stdio processes

### ADR-3: Global Singleton Registries

**Context:** Tool dispatch needs access to stateful registries (tasks, teams, MCP, LSP). Threading them through all call layers is verbose.

**Decision:** Use `OnceLock`-based global singletons in `tools/src/lib.rs`.

**Consequences:**
- (+) Simple access pattern from any call depth
- (+) Thread-safe via internal `Mutex`/`RwLock`
- (-) Global state makes unit testing harder (must use shared test locks)
- (-) Cannot have multiple independent registry instances

### ADR-4: Multi-Provider Architecture

**Context:** Users need flexibility beyond just Anthropic.

**Decision:** `ProviderClient` enum with `AnthropicClient` and `OpenAiCompatClient` (reused for xAI, OpenAI, DashScope, Ollama, OpenRouter).

**Consequences:**
- (+) Single codebase supports 5+ providers
- (+) Model-name prefix routing auto-selects provider
- (-) Different providers have different feature sets (prompt caching is Anthropic-only)
- (-) OpenAI-compat normalization adds edge cases

### ADR-5: Append-Only JSONL Session Storage

**Context:** Sessions must be durable, resumable, and inspectable.

**Decision:** Each message appended as a JSON line to `.claw/sessions/*.jsonl`. Rotation at 256 KB.

**Consequences:**
- (+) Simple, crash-resistant (no partial writes corrupt the file)
- (+) Human-readable and greppable
- (+) Easy session export
- (-) No random access (must replay to load)
- (-) Rotation adds complexity for long sessions

### ADR-6: Permission System with Three Tiers

**Context:** Tools have varying risk levels (read vs write vs bash execution).

**Decision:** Three-tier system: `PermissionMode` (session), `PermissionPolicy` (rules), `PermissionEnforcer` (tool-specific).

**Consequences:**
- (+) Fine-grained control at every level
- (+) Hooks can override permissions via `PermissionContext`
- (+) Default `danger-full-access` for frictionless autonomous operation
- (-) Complexity — three layers to reason about

---

## 15. Architecture Governance

### Automated Checks

| Check | Enforcement |
|---|---|
| `cargo fmt --all --check` | CI — formatting consistency |
| `cargo clippy --workspace` | CI — lint quality (pedantic level) |
| `cargo test --workspace` | CI — all tests pass |
| `unsafe_code = "forbid"` | Workspace lint — no unsafe anywhere |
| `clippy::pedantic` | Workspace lint — high code quality bar |
| Doc source-of-truth check | CI — Python script verifying docs aren't stale |
| Mock parity harness | `scripts/run_mock_parity_harness.sh` — behavioral regression |
| PARITY.md mapping | `scripts/run_mock_parity_diff.py` — scenario-to-parity verification |

### Architectural Documentation

| Document | Purpose |
|---|---|
| `PARITY.md` | Rust port parity status (lanes, tool surface, open items) |
| `ROADMAP.md` | Active roadmap with phases and product principles |
| `PHILOSOPHY.md` | Project intent and coordination system design |
| `USAGE.md` | Task-oriented user guide |
| `rust/README.md` | Crate map, features, workspace layout |
| `docs/container.md` | Container-first workflow |
| `CLAUDE.md` | AI assistant guidance for working with this codebase |

### Review Practices

- Parity tracking via `PARITY.md` with commit hashes and evidence
- Lane-based development (9 feature lanes documented with merge commits)
- `#[ignore]` test prohibition (verified via grep)

---

## 16. Blueprint for New Development

### Development Workflow

1. **Understand the crate map** — Start from the layer diagram. Identify which crate(s) your change touches.
2. **Read the relevant module** — Each `.rs` file has a focused responsibility. Check `lib.rs` exports.
3. **Check PARITY.md** — If your change relates to upstream feature parity, update the status.
4. **Implement** — Follow existing patterns (tool spec → dispatch → execution, slash command → enum → handler).
5. **Test** — Unit tests in `#[cfg(test)]` modules. Integration tests if behavioral.
6. **Verify** — `cargo fmt`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`.

### Component Creation Sequence

**New tool:**
1. `tools/src/lib.rs` — add `ToolSpec` to `mvp_tool_specs()`
2. `tools/src/lib.rs` — add match arm in `execute_tool()`
3. `runtime/src/*.rs` — implement execution logic (if complex, new module)
4. `runtime/src/lib.rs` — export new public types
5. Test in `#[cfg(test)]` module

**New runtime subsystem:**
1. `runtime/src/new_module.rs` — implement types + logic
2. `runtime/src/lib.rs` — add `pub mod` + `pub use` exports
3. If tool-facing: `tools/src/lib.rs` — add `global_*_registry()` singleton
4. Wire into tool dispatch or slash commands as needed

**New provider:**
1. `api/src/providers/new_provider.rs` — implement client
2. `api/src/providers/mod.rs` — add to provider detection
3. `api/src/client.rs` — add `ProviderClient` variant
4. `api/src/lib.rs` — export

### Verification Checklist

```bash
cd rust
cargo fmt --all --check           # formatting
cargo clippy --workspace --all-targets -- -D warnings  # lints
cargo test --workspace            # all tests
./scripts/run_mock_parity_harness.sh  # parity (Linux/macOS)
```

### Common Pitfalls

| Pitfall | Guidance |
|---|---|
| **Circular crate deps** | Layer 1 crates (plugins, telemetry) must never depend on higher layers |
| **Unsafe code** | Workspace-level `forbid` — don't even try |
| **Tool name conflicts** | `GlobalToolRegistry::with_plugin_tools()` enforces uniqueness |
| **Permission bypass** | Every tool spec must declare `required_permission` |
| **Global registry state** | Tests sharing global singletons must use `test_env_lock()` |
| **Session timestamp monotonicity** | Use `LAST_TIMESTAMP_MS` atomic to ensure ordering |
| **Config precedence** | Local overrides project overrides user — don't reverse |
| **Provider routing** | Model prefix determines provider; don't rely solely on env vars |

### LOC Distribution by Crate

| Crate | LOC | % | Role |
|---|---|---|---|
| `runtime` | 26,550 | 38.0% | Core engine |
| `rusty-claude-cli` | 14,800 | 21.2% | CLI binary |
| `tools` | 9,525 | 13.6% | Tool dispatch |
| `api` | 7,979 | 11.4% | Provider clients |
| `commands` | 5,279 | 7.6% | Slash commands |
| `plugins` | 3,825 | 5.5% | Plugin system |
| `mock-anthropic-service` | 1,100 | 1.6% | Test mock |
| `telemetry` | 476 | 0.7% | Tracing types |
| `compat-harness` | 319 | 0.5% | Parity harness |
| **Total** | **69,853** | **100%** | |

---

*This blueprint was generated on 2026-04-13 from commit `0c51f83` of `ultraworkers/claw-code`. As the architecture evolves, regenerate this blueprint after significant structural changes (new crates, new layers, major refactors). Keep `PARITY.md` and `ROADMAP.md` as the living documents for feature status and planned work.*
