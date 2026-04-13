# Claw Code — Rust Implementation: Architecture Blueprint

> **Generated:** 2026-04-13T01:25:20.545Z  
> **Mode:** Rust  
> **Scope:** `rust/` workspace inside `claw-code` repository  
> **Detail Level:** Comprehensive / Implementation-Ready

---

## Table of Contents

1. [Architectural Overview](#1-architectural-overview)
2. [Architecture Visualization (C4)](#2-architecture-visualization-c4)
3. [Workspace Crate Map](#3-workspace-crate-map)
4. [Core Architectural Components](#4-core-architectural-components)
5. [Architectural Layers and Dependencies](#5-architectural-layers-and-dependencies)
6. [Data Architecture](#6-data-architecture)
7. [Cross-Cutting Concerns](#7-cross-cutting-concerns)
8. [Service Communication Patterns](#8-service-communication-patterns)
9. [Rust-Specific Architectural Patterns](#9-rust-specific-architectural-patterns)
10. [Implementation Patterns](#10-implementation-patterns)
11. [Testing Architecture](#11-testing-architecture)
12. [Deployment Architecture](#12-deployment-architecture)
13. [Extension and Evolution Patterns](#13-extension-and-evolution-patterns)
14. [Architectural Pattern Examples](#14-architectural-pattern-examples)
15. [Architectural Decision Records](#15-architectural-decision-records)
16. [Architecture Governance](#16-architecture-governance)
17. [Blueprint for New Development](#17-blueprint-for-new-development)

---

## 1. Architectural Overview

### What Claw Code Is

Claw Code is a high-performance, safety-first **AI coding agent CLI** — a Rust rewrite of an original Python/TypeScript implementation. The binary is named `claw` and wraps the Anthropic Claude API (and OpenAI-compatible endpoints) in an interactive terminal agent capable of reading, writing, and executing files via a structured tool-use loop.

The system supports two primary modes of operation:
- **Interactive REPL** — continuous multi-turn conversation with the model in a terminal.
- **One-shot prompt** — non-interactive `--print` / `--output-format json` mode for scripting and CI.

### Guiding Architectural Principles

| Principle | Evidence in code |
|---|---|
| **Safety over convenience** | `unsafe_code = "forbid"` in workspace lints; permission gate on every tool invocation |
| **Trait-driven abstraction** | `ApiClient`, `ToolExecutor`, `PermissionPrompter` are traits, not concrete types |
| **Layered responsibility** | Strict crate dependency graph; each crate owns one domain |
| **Fail-fast with structured errors** | `Result`-returning public APIs; `RuntimeError`, `ApiError`, `ToolError` per domain |
| **Observable by default** | `telemetry` crate is a first-class dependency of both `api` and `runtime` |
| **Pluggable extension points** | Plugin system (builtin / bundled / external), MCP protocol, hook callbacks |

### High-Level Execution Flow

```
User input (stdin / CLI args)
        │
        ▼
  [rusty-claude-cli]  ← argument parsing, output formatting, REPL loop
        │
        ├─ [commands]     ← slash-command dispatch, skill invocation
        │
        ├─ [runtime]      ← session, conversation loop, tools, permissions, MCP
        │       │
        │       ├─ [plugins]   ← plugin lifecycle, hook runner
        │       └─ [telemetry] ← structured event emission
        │
        └─ [api]          ← HTTP client, SSE streaming, provider abstraction
                │
                └─ [telemetry]  (shared)
```

---

## 2. Architecture Visualization (C4)

### Level 1 — System Context

```
┌─────────────────────────────────────────────────────────────────────┐
│                        External World                               │
│                                                                     │
│  ┌──────────┐      ┌───────────────────────────┐    ┌───────────┐  │
│  │  Human   │─────▶│      claw  CLI binary      │───▶│ Anthropic │  │
│  │  / CI    │      │  (interactive or scripted) │    │  API /    │  │
│  └──────────┘      └───────────────────────────┘    │ OpenAI    │  │
│                              │                       │ compat    │  │
│                    ┌─────────┴──────────┐            └───────────┘  │
│                    │  Filesystem / Git  │                           │
│                    │  MCP Servers       │                           │
│                    │  LSP Servers       │                           │
│                    └────────────────────┘                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Level 2 — Container Diagram (Cargo workspace)

```
┌──────────────────────────── rust/ workspace ─────────────────────────────┐
│                                                                           │
│  ┌────────────────────┐                                                   │
│  │  rusty-claude-cli  │  (binary: claw)                                   │
│  │  - REPL / one-shot │                                                   │
│  │  - arg parsing     │                                                   │
│  │  - terminal render │                                                   │
│  └──────────┬─────────┘                                                   │
│             │ depends on                                                  │
│   ┌─────────┼──────────────────────────────────────────┐                 │
│   │         │                                           │                 │
│   ▼         ▼                     ▼                     ▼                 │
│ ┌────────┐ ┌──────────────────┐ ┌─────────┐ ┌──────────────────────┐    │
│ │  api   │ │     runtime      │ │commands │ │  compat-harness      │    │
│ │        │ │ (largest crate)  │ │         │ │  (manifest extractor)│    │
│ │  HTTP  │ │ conversation     │ │ slash   │ └──────────────────────┘    │
│ │  SSE   │ │ session          │ │ commands│                              │
│ │ provid.│ │ permissions      │ │ skills  │                              │
│ │  auth  │ │ tools/file ops   │ │         │                              │
│ │        │ │ MCP plumbing     │ └─────────┘                              │
│ └────┬───┘ │ plugin lifecycle │                                          │
│      │     │ policy engine    │         ┌──────────────────────┐         │
│      │     │ worker boot      │◀────────│        tools         │         │
│      │     └────────┬─────────┘         │  execute_tool()      │         │
│      │              │                   │  GlobalToolRegistry  │         │
│      │              │                   └──────────────────────┘         │
│      │              ▼                                                     │
│      │     ┌──────────────────┐         ┌──────────────────────┐         │
│      └────▶│    telemetry     │◀────────│       plugins        │         │
│            │  SessionTracer   │         │  PluginManager       │         │
│            │  JsonlSink       │         │  PluginRegistry      │         │
│            └──────────────────┘         │  hook runner         │         │
│                                         └──────────────────────┘         │
│                                                                           │
│  ┌──────────────────────────────────────────────────┐                    │
│  │ mock-anthropic-service  (test/dev only)          │                    │
│  │  in-process TCP server; SSE streaming mock       │                    │
│  └──────────────────────────────────────────────────┘                    │
└───────────────────────────────────────────────────────────────────────────┘
```

### Level 3 — Key Component Interactions (runtime)

```
ConversationRuntime
     │
     ├── ApiClient trait ──────────────────▶ AnthropicClient (api crate)
     │        │                                    │
     │        │                             SSE stream (reqwest)
     │        ▼
     ├── ToolExecutor trait ─────────────▶ GlobalToolRegistry (tools crate)
     │        │                                    │
     │        │                        ┌───────────┴──────────────┐
     │        │                        │  file_ops  bash  lsp  mcp │
     │        ▼
     ├── PermissionPrompter trait ─────▶ CliPermissionPrompter
     │        │                          (in rusty-claude-cli)
     │        ▼
     ├── HookRunner ────────────────▶ pre/post tool-use shell hooks
     │
     └── Session ──────────────────▶ JSONL file persistence
```

---

## 3. Workspace Crate Map

| Crate | Kind | Primary Responsibility | Key Exports |
|---|---|---|---|
| `rusty-claude-cli` | Binary | Entry point, CLI arg parsing, REPL, terminal rendering | `main`, `LiveCli`, `CliAction` |
| `runtime` | Library | Session, conversation loop, permissions, MCP, tools file ops, config | `ConversationRuntime`, `Session`, `PermissionMode`, `McpServer`, `WorkerRegistry` |
| `api` | Library | HTTP client, SSE parser, Anthropic/OpenAI providers, prompt cache | `AnthropicClient`, `OpenAiCompatClient`, `PromptCache`, `SessionTracer` (re-export) |
| `commands` | Library | Slash-command specs and dispatch, skill system | `SlashCommand`, `slash_command_specs`, `resolve_skill_invocation` |
| `tools` | Library | Tool registration and execution dispatch, LSP/MCP bridge | `GlobalToolRegistry`, `execute_tool`, `mvp_tool_specs` |
| `plugins` | Library | Plugin discovery, lifecycle, hook running (builtin/bundled/external) | `PluginManager`, `PluginRegistry`, `HookRunner` |
| `telemetry` | Library | Structured event emission, session tracing, analytics | `SessionTracer`, `TelemetrySink`, `JsonlTelemetrySink` |
| `compat-harness` | Library | Upstream manifest extraction for parity validation | `UpstreamPaths`, `extract_manifest` |
| `mock-anthropic-service` | Library (test) | Deterministic TCP mock of Anthropic API for integration tests | `MockAnthropicService`, `CapturedRequest` |

---

## 4. Core Architectural Components

### 4.1 `rusty-claude-cli` — Entry Point and Terminal Layer

**Purpose**: Owns the binary entry point (`src/main.rs`), CLI argument parsing, output format selection (text / JSON), interactive REPL, terminal rendering, and permission prompting.

**Internal structure**:
- `main.rs` — `main()` → `run()` → `parse_args()` → `CliAction` dispatch
- `render.rs` — `TerminalRenderer`, `Spinner`, `MarkdownStreamState`, `ColorTheme` using `crossterm` + `pulldown-cmark` + `syntect`
- `init.rs` — `initialize_repo()`, repo scaffold (`CLAW_JSON`, `.gitignore` entries)
- `input.rs` — line-reading / rustyline REPL input handling

**Key types**:
```
CliAction       — exhaustive enum of every CLI command
CliOutputFormat — Text | Json
LiveCli         — stateful orchestrator for interactive or one-shot turns
```

**Interaction pattern**: `LiveCli` composes `ConversationRuntime` from `runtime`, passes itself as `ToolExecutor` + `PermissionPrompter`, streams `AssistantEvent`s back and renders them incrementally.

---

### 4.2 `runtime` — Core Runtime Engine

**Purpose**: The largest and most important crate. Owns everything that happens _after_ a prompt arrives and _before_ the response is displayed.

**Sub-modules and their responsibilities**:

| Module | Responsibility |
|---|---|
| `conversation` | `ConversationRuntime` — the turn loop; `ApiClient` + `ToolExecutor` traits |
| `session` | `Session` — JSONL-backed conversation persistence; message rotation |
| `config` / `config_validate` | Multi-source layered config (user → project → local); JSON schema validation |
| `permissions` / `permission_enforcer` | `PermissionMode`, `PermissionPolicy`, `PermissionPrompter` trait; enforcement result |
| `policy_engine` | `PolicyEngine`, `PolicyRule`, `LaneContext` — lane / branch merge governance |
| `file_ops` | `read_file`, `write_file`, `edit_file`, `glob_search`, `grep_search` |
| `bash` / `bash_validation` | Sandboxed bash command execution; command safety validation |
| `mcp*` | Full MCP (Model Context Protocol) plumbing: stdio/SSE/managed proxy/WebSocket transports |
| `hooks` | `HookRunner` — pre/post tool-use lifecycle hooks via shell subprocess |
| `plugin_lifecycle` | `PluginLifecycle` — server health, discovery, degraded-mode handling |
| `worker_boot` | `WorkerRegistry` — in-memory state machine for sub-agent worker lifecycle |
| `task_registry` | Sub-agent task lifecycle management (`Created → Running → Completed/Failed`) |
| `team_cron_registry` | Team and cron job scheduling registries |
| `lsp_client` | `LspRegistry` — Language Server Protocol action dispatch (diagnostics, hover, etc.) |
| `bootstrap` | `BootstrapPhase` / `BootstrapPlan` — ordered startup phase sequencing |
| `sandbox` | `SandboxConfig`, `SandboxStatus`, container environment detection |
| `compact` / `summary_compression` | Session compaction when token budget is exhausted |
| `prompt` | `SystemPromptBuilder`, context file loading, `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` |
| `recovery_recipes` | Failure scenario detection and structured recovery step generation |
| `stale_base` / `stale_branch` | Git base commit and branch freshness checks |
| `branch_lock` | Branch lock collision detection for parallel workers |
| `oauth` | PKCE OAuth2 flow, token storage/refresh |
| `remote` | Upstream proxy bootstrap, remote session context |
| `usage` | Token usage tracking, cost estimation, USD formatting |
| `lane_events` | Lane commit provenance, event deduplication |
| `task_packet` | Validated task packet structure for inter-agent handoff |
| `green_contract` | Green-level contract for policy-gated lane progression |
| `sse` | Incremental SSE frame parser (used internally) |

---

### 4.3 `api` — Provider Abstraction Layer

**Purpose**: Wraps external AI provider HTTP APIs behind a streaming trait. Handles authentication, retries, prompt caching, and SSE parsing.

**Providers**:
- `providers/anthropic.rs` — `AnthropicClient`: full Anthropic Messages API with streaming, PKCE OAuth refresh, exponential backoff (up to 8 retries, 128 s max)
- `providers/openai_compat.rs` — `OpenAiCompatClient`: drop-in OpenAI-compatible endpoint

**Authentication sources** (`AuthSource` enum):
- `ApiKey` — `ANTHROPIC_API_KEY` env var
- `BearerToken` — `ANTHROPIC_AUTH_TOKEN` env var
- `ApiKeyAndBearer` — both present simultaneously

**Key patterns**:
- `ProviderClient` trait — `stream()` returning `MessageStream`
- `PromptCache` — records cache hits/misses, surfaced as `PromptCacheEvent` in `AssistantEvent`
- `SseParser` — incremental SSE frame parsing; `parse_frame()` for single-frame testing
- `ProxyConfig` — upstream proxy support via `build_http_client_with()`

---

### 4.4 `tools` — Tool Execution Hub

**Purpose**: Central dispatch for every tool the model can call. Maintains global static registries for LSP, MCP, team, cron, task, and worker state.

**Tool categories**:
- `Base` — always available (file ops, bash, grep, glob, git context)
- `Conditional` — activated by configuration or model request
- Plugin-contributed tools via `PluginTool`

**Global registries** (lazily initialized via `OnceLock`):
```rust
global_lsp_registry()    → &'static LspRegistry
global_mcp_registry()    → &'static McpToolRegistry
global_team_registry()   → &'static TeamRegistry
global_cron_registry()   → &'static CronRegistry
global_task_registry()   → &'static TaskRegistry
global_worker_registry() → &'static WorkerRegistry
```

**`execute_tool(name, input)`** — the single dispatch entrypoint called by `ToolExecutor::execute()`.

---

### 4.5 `plugins` — Plugin System

**Purpose**: Discovers, loads, and manages the lifecycle of plugins. Provides a hook runner for pre/post tool-use callbacks.

**Plugin kinds**:
- `Builtin` — shipped with the binary (`builtin/` marketplace)
- `Bundled` — shipped in `crates/plugins/bundled/` (e.g. `example-bundled`, `sample-hooks`)
- `External` — user-installed from external directories

**Plugin metadata**: defined by `plugin.json` (manifest at `.claude-plugin/plugin.json`)

**`PluginManager`**: orchestrates discovery (`DiscoveryResult`), registration, health checks (`PluginHealthcheck`), and degraded mode (`DegradedMode`).

**`HookRunner`** (in `plugins/src/hooks.rs`): executes `PreToolUse` / `PostToolUse` / `PostToolUseFailure` hook commands as subprocesses; supports abort signals and progress reporting.

---

### 4.6 `commands` — Slash Command System

**Purpose**: Defines the full set of REPL slash commands (`/help`, `/status`, `/mcp`, `/agents`, `/skills`, `/plugins`, etc.) and skill invocation.

**`SlashCommandSpec`** — static spec with `name`, `aliases`, `summary`, `argument_hint`, `resume_supported`.

**Skill dispatch**: `resolve_skill_invocation()` → `SkillSlashDispatch::Local | Invoke(path)` — skills are markdown documents in `.github/skills/` that are injected as context.

---

### 4.7 `telemetry` — Observability Layer

**Purpose**: Provides structured analytics event emission, session tracing, and JSONL sink writing. First-class dependency of both `api` and `runtime`.

**Key types**:
- `SessionTracer` — wraps a `TelemetrySink`; records `SessionTraceRecord` entries
- `JsonlTelemetrySink` — appends JSONL records to a file
- `MemoryTelemetrySink` — in-memory accumulator for testing
- `AnthropicRequestProfile` — version + identity + beta flags carried in every API request
- `ClientIdentity` — `app_name`, `app_version`, `runtime` ("rust")

---

### 4.8 `compat-harness` — Parity Validation Bridge

**Purpose**: Extracts command and tool manifests from the upstream Python/TypeScript sources to drive parity validation.

**`UpstreamPaths`**: locates `src/commands.ts`, `src/tools.ts`, `src/entrypoints/cli.tsx` relative to the repo root.

**`extract_manifest()`**: returns `ExtractedManifest { commands: CommandRegistry, tools: ToolRegistry, bootstrap: BootstrapPlan }`.

---

### 4.9 `mock-anthropic-service` — Test Infrastructure

**Purpose**: Provides a deterministic, async TCP server that speaks the Anthropic Messages API protocol (JSON + SSE streaming) for integration tests without network calls.

**`MockAnthropicService::spawn()`**: binds to an ephemeral port; captures `CapturedRequest` records; responds with scripted fixtures keyed by `PARITY_SCENARIO:` request prefix.

---

## 5. Architectural Layers and Dependencies

### Dependency Graph (acyclic, enforced by cargo)

```
rusty-claude-cli
  ├── api
  │     └── telemetry
  ├── runtime
  │     ├── plugins
  │     │     └── (no internal deps)
  │     └── telemetry
  ├── commands
  │     ├── plugins
  │     └── runtime
  ├── tools
  │     ├── api
  │     ├── plugins
  │     └── runtime
  └── compat-harness
        ├── commands
        ├── runtime
        └── tools
```

**Dependency rules**:
1. `telemetry` and `plugins` are foundation crates — they have no internal workspace dependencies.
2. `runtime` may depend on `plugins` and `telemetry` but NOT on `api`, `tools`, `commands`, or the binary.
3. `api` may depend on `runtime` (for OAuth/config helpers) and `telemetry`.
4. `tools` bridges `api`, `runtime`, and `plugins` — it is the composition layer for tool execution.
5. `commands` composes `runtime` and `plugins`.
6. `rusty-claude-cli` is the only crate allowed to depend on all libraries.

**No circular dependencies** are present or permissible.

---

## 6. Data Architecture

### 6.1 Session Persistence

Sessions are stored as **JSONL files** (`.jsonl`) in `.claw/sessions/` with automatic rotation at 256 KB, keeping up to 3 rotated files.

```
Session
 ├── id: String (monotonic counter + timestamp)
 ├── messages: Vec<ConversationMessage>
 │    ├── role: MessageRole (System | User | Assistant | Tool)
 │    ├── blocks: Vec<ContentBlock>
 │    │    ├── Text { text }
 │    │    ├── ToolUse { id, name, input }
 │    │    └── ToolResult { tool_use_id, tool_name, output, is_error }
 │    └── usage: Option<TokenUsage>
 ├── compaction: Option<SessionCompaction>
 └── fork: Option<SessionFork>
```

### 6.2 Configuration Model

Config is layered (User < Project < Local) and merged at startup:

```
RuntimeConfig
 └── RuntimeFeatureConfig
      ├── RuntimeHookConfig     (pre/post tool hook commands)
      ├── RuntimePluginConfig   (enabled plugins, install root, registry path)
      ├── McpConfigCollection   (MCP server configs keyed by name)
      ├── OAuthConfig           (OAuth endpoint configuration)
      ├── ResolvedPermissionMode
      ├── RuntimePermissionRuleConfig (per-tool permission overrides)
      ├── SandboxConfig
      ├── ProviderFallbackConfig (primary + fallback model chain)
      └── trusted_roots: Vec<String>
```

Config files: `~/.claude.json` (user), `.claw/settings.json` (project), `.claw/settings.local.json` (local, gitignored).

### 6.3 Token Budget

`UsageTracker` accumulates `TokenUsage { input_tokens, output_tokens, cache_read_tokens, cache_write_tokens }` per turn. `CompactionConfig` triggers `compact_session()` when input tokens exceed `CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS` (default: 100,000).

### 6.4 Task Packet

`TaskPacket` is a validated inter-agent handoff structure containing a prompt, optional metadata, and team routing information. Validated by `validate_packet()` → `ValidatedPacket`.

---

## 7. Cross-Cutting Concerns

### 7.1 Authentication & Authorization

**API authentication** (`api/src/providers/anthropic.rs`):
- `AuthSource::from_env()` reads `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN`
- OAuth2 PKCE flow via `runtime::oauth` — `generate_pkce_pair()`, `code_challenge_s256()`, token exchange and refresh with `save_oauth_credentials()` / `load_oauth_credentials()`
- `resolve_startup_auth_source()` selects the right source at startup

**Tool authorization** (`runtime/src/permissions.rs`, `permission_enforcer.rs`):
- Every tool call passes through `PermissionEnforcer::enforce()`
- `PermissionMode` hierarchy: `ReadOnly < WorkspaceWrite < DangerFullAccess | Prompt | Allow`
- `PermissionPolicy` maps tool names to required modes
- `PermissionPrompter` trait — interactive or automatic decision (hook override → policy → user prompt)
- Hook overrides (`PermissionOverride::Allow | Deny | Ask`) can short-circuit the standard flow

### 7.2 Error Handling & Resilience

- Public APIs return `Result<T, E>` with typed error enums per domain
- `RuntimeError`, `ApiError`, `ToolError`, `SessionError`, `ConfigError`, `PromptBuildError` — no `anyhow` or untyped `Box<dyn Error>` in library crates (only the binary uses `Box<dyn Error>`)
- `AnthropicClient` implements **exponential backoff** (1 s → 128 s, max 8 retries) on 429/500/503
- `ProviderFallbackConfig` — ordered fallback model chain tried on retryable failures
- `recovery_recipes` — `FailureScenario` detection + `RecoveryRecipe` with ordered `RecoveryStep`s + `EscalationPolicy`
- MCP servers: `McpLifecycleValidator` tracks phase-by-phase health; `McpDegradedReport` surfaces per-server failures without aborting the session

### 7.3 Logging & Monitoring

- `telemetry::SessionTracer` records structured `TelemetryEvent`s to a JSONL sink during every turn
- `AnthropicRequestProfile` carries `ClientIdentity { app_name: "claude-code", runtime: "rust" }` and beta flags in every API request header
- `PromptCacheEvent` surfaces cache hit/miss details as part of `AssistantEvent::PromptCache`
- `UsageTracker` records cumulative token costs; `format_usd()` formats cost display

### 7.4 Validation

- Config validation: `validate_config_file()` → `ValidationResult { diagnostics: Vec<ConfigDiagnostic> }` with `DiagnosticKind::Error | Warning | Info`
- Bash command safety: `bash_validation` module checks commands before execution in sandboxed mode
- Task packets: `validate_packet()` enforces structural integrity before registration
- Permission mode: explicitly enforced before every tool invocation; violations produce structured `PermissionOutcome`

### 7.5 Sandbox & Isolation

`SandboxConfig` / `SandboxStatus` track:
- `FilesystemIsolationMode` — `Off | WorkspaceOnly | AllowList`
- `namespace_restrictions`, `network_isolation` — Linux namespaces (where supported)
- `ContainerEnvironment` detection — checks for container markers in the environment
- `build_linux_sandbox_command()` wraps tool invocations with sandbox constraints

### 7.6 Configuration Management

- Three-level layered config: User (`~/.claude.json`) → Project (`.claw/settings.json`) → Local (`.claw/settings.local.json`)
- `ConfigLoader` merges sources in precedence order
- `CLAW_SETTINGS_SCHEMA_NAME = "SettingsSchema"` — JSON schema advertised for editor tooling
- Feature flags embedded in `RuntimeFeatureConfig`; no separate feature-flag service

---

## 8. Service Communication Patterns

### 8.1 Anthropic / OpenAI API (synchronous HTTP + SSE streaming)

```
AnthropicClient::stream(request)
  │
  └─ reqwest blocking Client
       │
       ├─ POST /v1/messages  (JSON body, stream: true)
       │
       └─ SSE response → SseParser → Vec<StreamEvent> → Vec<AssistantEvent>
```

The `api` crate uses **blocking** `reqwest` (not async) to avoid requiring the calling context to be inside an async runtime. `tokio` is used only in `runtime` and the binary for process spawning and time.

### 8.2 MCP (Model Context Protocol) — Multiple Transports

MCP servers can be reached via four transport types:

| Transport | Config type | Notes |
|---|---|---|
| `McpStdioTransport` | `McpStdioServerConfig` | Subprocess stdin/stdout JSON-RPC |
| `McpRemoteTransport` | `McpRemoteServerConfig` | HTTP SSE remote server |
| `McpSdkTransport` | `McpSdkServerConfig` | SDK-managed server |
| `McpManagedProxyTransport` | `McpManagedProxyServerConfig` | Proxy with OAuth |

**Lifecycle phases** (tracked by `McpLifecycleValidator`):
```
ConfigLoad → ServerRegistration → SpawnConnect → InitializeHandshake
  → ToolDiscovery → ResourceDiscovery → Ready → Invocation
  → ErrorSurfacing → Shutdown → Cleanup
```

JSON-RPC 2.0 over stdio: `JsonRpcRequest` / `JsonRpcResponse` / `JsonRpcError` with typed `McpToolCallParams`, `McpListToolsResult`, etc.

`McpServerManager` manages multiple concurrent MCP servers; each gets a name-scoped tool prefix via `mcp_tool_prefix()`.

### 8.3 LSP (Language Server Protocol)

`LspRegistry` maintains a map of language-server connections. Actions: `Diagnostics`, `Hover`, `Definition`, `References`, `Completion`, `Symbols`, `Format`. Results are returned as typed structs (`LspDiagnostic`, `LspLocation`, `LspHoverResult`) surfaced through `tools::execute_tool`.

### 8.4 Hook Subprocesses

Hooks are shell commands executed as subprocesses (`std::process::Command`) at `PreToolUse` / `PostToolUse` / `PostToolUseFailure` events. They receive JSON on stdin and can return a `PermissionOverride` decision.

### 8.5 Worker / Sub-Agent Communication

`WorkerRegistry` tracks child worker processes by ID. Workers transition through `WorkerStatus` states (`Spawning → TrustRequired → ReadyForPrompt → Running → Finished | Failed`). Communication uses `WorkerEvent` records; trust resolution uses `WorkerTrustResolution`.

---

## 9. Rust-Specific Architectural Patterns

### 9.1 Crate Organization

- **Workspace resolver = "2"** — independent feature resolution per crate
- All crates share `version`, `edition = "2021"`, `license`, `publish = false` via `[workspace.package]`
- `serde_json` is a workspace dependency to enforce a single version

### 9.2 Memory and Safety

- `#![forbid(unsafe_code)]` enforced workspace-wide via `[workspace.lints.rust]`
- `Arc<Mutex<T>>` used for shared mutable registries (e.g. `WorkerRegistry`, `TaskRegistry`, `MockAnthropicService`)
- `OnceLock<T>` for lazily-initialized global singletons (all global registries in `tools`)
- `AtomicU64` for lock-free counters (session ID, telemetry sequence)
- `BTreeMap` preferred over `HashMap` in public data structures for deterministic ordering

### 9.3 Trait-Based Abstraction

Three critical traits enable dependency injection and testability:

```rust
// Conversation driver
pub trait ApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>;
}

// Tool dispatcher
pub trait ToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError>;
}

// Permission decision
pub trait PermissionPrompter {
    fn decide(&mut self, request: &PermissionRequest) -> PermissionPromptDecision;
}
```

`ConversationRuntime` is generic over these traits — making it fully testable with mock implementations.

### 9.4 Async Strategy

- `tokio` is used for MCP stdio subprocess management, the mock service, and terminal signal handling
- The API client uses **blocking** `reqwest` (not tokio) to remain usable from synchronous callers
- Hook subprocess execution uses `std::process::Command` (sync), with a background thread for abort signaling

### 9.5 Error Propagation

```rust
// Library crates: typed errors, no Box<dyn Error>
pub enum RuntimeError { ... }
pub struct ApiError { ... }
pub struct ToolError { message: String }

// Binary crate: ergonomic Box<dyn Error>
fn run() -> Result<(), Box<dyn std::error::Error>> { ... }
```

### 9.6 Clippy Governance

Workspace enables `clippy::all` and `clippy::pedantic` at warn level with `priority = -1`, with selective allows:
- `module_name_repetitions = "allow"` — permits `McpServer`, `McpServerManager`, etc.
- `missing_panics_doc = "allow"` / `missing_errors_doc = "allow"` — reduces doc burden on internal APIs

Per-file `#![allow(...)]` overrides exist in `main.rs` (dead code during porting) and `worker_boot.rs`.

---

## 10. Implementation Patterns

### 10.1 Interface Design

**Trait segregation**: `ApiClient`, `ToolExecutor`, `PermissionPrompter`, `HookProgressReporter`, `TelemetrySink` are all narrow single-responsibility traits. Callers depend on trait objects or generics, never concrete types.

**`#[must_use]`** applied to all factory and query methods returning meaningful values — enforced by clippy::pedantic.

### 10.2 Service Implementation

**Lifetime management**:
- `GlobalToolRegistry` and friends are `'static` singletons via `OnceLock`
- `Session` encapsulates file handles and manages rotation internally
- `ConversationRuntime` is short-lived — one per interactive session or one-shot prompt

**Provider fallback chain** in `api`:
```rust
ProviderFallbackConfig { primary: Option<String>, fallbacks: Vec<String> }
```
On retryable failure, the runtime tries each fallback in order before surfacing the error.

### 10.3 Tool Implementation Pattern

Every tool callable by the model follows this contract:
1. Has a `ToolDefinition` (name, description, JSON schema for parameters)
2. Is registered in `mvp_tool_specs()` or via a plugin's `PluginTool`
3. Dispatched by `execute_tool(name, input_json)` → `String` result or `ToolError`
4. Goes through `PermissionEnforcer` before execution
5. Hook callbacks fire pre and post execution

### 10.4 Plugin Implementation Pattern

A plugin is a directory containing:
- `plugin.json` manifest (`PluginMetadata`: id, name, version, description, kind)
- Optional hook command scripts
- Optional tool definitions

Registration flow:
```
PluginManager::discover() 
  → PluginHealthcheck 
  → PluginLifecycleEvent::Registered 
  → PluginRegistry
```

### 10.5 Command Implementation Pattern

Slash commands are defined as static `SlashCommandSpec` entries. Handlers return `Value` (JSON) or formatted `String` depending on `output_format`. The `resume_supported` flag controls availability after session resume.

---

## 11. Testing Architecture

### 11.1 Test Strategy

| Layer | Test type | Location | Notes |
|---|---|---|---|
| Unit | `#[test]` in-module | `src/*.rs` | Pure logic; no I/O |
| Integration | `tests/*.rs` in crate | `crates/*/tests/` | Uses `mock-anthropic-service` |
| Parity | Mock harness | `crates/rusty-claude-cli/tests/mock_parity_harness.rs` | Full CLI end-to-end via scripted scenarios |
| CLI contract | Integration | `cli_flags_and_config_defaults.rs`, `output_format_contract.rs` | Validates arg parsing, JSON output shape |

### 11.2 Test Infrastructure

**`MockAnthropicService`**: spawns an async TCP server on an ephemeral port; test code sets `ANTHROPIC_BASE_URL` to point at it. Captures all requests for assertion.

**Scenario-driven**: scenarios are keyed by `PARITY_SCENARIO:` prefix in the request body; `mock_parity_scenarios.json` defines scripted responses.

**`test_env_lock()`** in `runtime`: a global `Mutex` ensuring env-var–mutating tests don't race.

**`MemoryTelemetrySink`** in `telemetry`: accumulates events in-memory for assertion without file I/O.

**`plugins::test_isolation`**: provides per-test plugin isolation utilities.

### 11.3 Test Tooling

- `cargo test --workspace` — runs all tests
- `cargo fmt` — formatting check (enforced in CI)
- `cargo clippy --workspace --all-targets -- -D warnings` — lint gate

---

## 12. Deployment Architecture

### 12.1 Binary Distribution

The `claw` binary is compiled from `rusty-claude-cli` as a single statically-linked executable (rustls TLS, no openssl dependency via `reqwest = { default-features = false, features = ["rustls-tls"] }`).

Build-time metadata injected by `build.rs`:
- `BUILD_DATE` — compilation timestamp
- `GIT_SHA` — source commit
- `TARGET` — Rust target triple

### 12.2 Runtime Environment

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Primary API authentication |
| `ANTHROPIC_AUTH_TOKEN` | OAuth bearer token |
| `ANTHROPIC_BASE_URL` | Override API base URL (proxy / mock) |
| `CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS` | Compaction threshold |
| `XAI_BASE_URL` | xAI provider base URL |

### 12.3 Filesystem Artifacts

```
~/.claude.json                    — user-level config
.claw/
  settings.json                   — project config (committed)
  settings.local.json             — local overrides (gitignored)
  sessions/
    <session-id>.jsonl            — JSONL session transcript
    <session-id>-1.jsonl          — rotated segment
  credentials/                    — OAuth token files
```

### 12.4 Container / Sandbox Support

- `detect_container_environment()` probes for `/.dockerenv`, cgroup markers, etc.
- `build_linux_sandbox_command()` wraps tool invocations with Linux namespace restrictions when `SandboxStatus.active = true`
- `Containerfile` at repo root provides a reference container image

### 12.5 CI / Parity Harness

`rust/scripts/run_mock_parity_harness.sh` — runs the binary against `MockAnthropicService` in a clean environment with scripted scenarios from `mock_parity_scenarios.json`.

---

## 13. Extension and Evolution Patterns

### 13.1 Adding a New Tool

1. Define `ToolDefinition` (name, description, JSON input schema)
2. Add an arm in `execute_tool()` in `tools/src/lib.rs`
3. Register in `mvp_tool_specs()` or conditionally in `GlobalToolRegistry`
4. Add permission mode requirement in `PermissionPolicy`
5. Write unit tests for the execution logic
6. Add integration test scenarios to `mock_parity_scenarios.json`

### 13.2 Adding a New Provider

1. Create `crates/api/src/providers/my_provider.rs` implementing the `Provider` trait
2. Add variant to `ProviderKind` enum
3. Wire into `detect_provider_kind()` / `resolve_startup_auth_source()`
4. Export from `api/src/lib.rs`

### 13.3 Adding a New Slash Command

1. Add `SlashCommandSpec` entry to `SLASH_COMMAND_SPECS` in `commands/src/lib.rs`
2. Implement handler function returning `Result<Value, String>`
3. Add dispatch arm in `rusty-claude-cli/src/main.rs`
4. Set `resume_supported` appropriately

### 13.4 Adding a New Plugin

1. Create plugin directory with `plugin.json`
2. Define hook scripts for `PreToolUse` / `PostToolUse`
3. Place in `bundled/` (shipped) or an external directory (user-installed)
4. Enable via config: `plugins.enabled.<plugin-id> = true`

### 13.5 Adding a New MCP Server

1. Add server config to `.claw/settings.json` under `mcpServers`
2. Choose transport: `stdio` | `remote` | `sdk` | `managed-proxy`
3. MCP lifecycle validator handles connection, tool discovery, and degraded mode automatically

### 13.6 Integration Anti-Corruption Patterns

- The `compat-harness` crate acts as an **anti-corruption layer** between the Rust implementation and the upstream Python/TypeScript sources — manifest extraction is isolated here, not scattered through business logic
- `UpstreamPaths` encapsulates path resolution, shielding callers from upstream directory structure changes

---

## 14. Architectural Pattern Examples

### 14.1 Trait-Based Dependency Injection (Conversation Loop)

```rust
// runtime/src/conversation.rs

pub trait ApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>;
}

pub trait ToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError>;
}

pub struct ConversationRuntime {
    session: Session,
    feature_config: RuntimeFeatureConfig,
    // ApiClient and ToolExecutor injected at construction time
}
```

The binary (`LiveCli`) implements both `ApiClient` (wrapping `AnthropicClient`) and `ToolExecutor` (dispatching to `tools::execute_tool`), allowing `ConversationRuntime` to remain independent of concrete provider and tool implementations.

### 14.2 OnceLock Global Registry Pattern

```rust
// tools/src/lib.rs

fn global_lsp_registry() -> &'static LspRegistry {
    use std::sync::OnceLock;
    static REGISTRY: OnceLock<LspRegistry> = OnceLock::new();
    REGISTRY.get_or_init(LspRegistry::new)
}
```

This pattern eliminates initialization overhead on unused registries while guaranteeing `'static` lifetime for use across async task boundaries.

### 14.3 Layered Config Merge

```rust
// runtime/src/config.rs

pub enum ConfigSource { User, Project, Local }  // ordered by precedence

pub struct ConfigLoader {
    entries: Vec<ConfigEntry>,  // User < Project < Local
}

impl ConfigLoader {
    pub fn load() -> Result<RuntimeConfig, ConfigError> {
        // merge entries in ascending precedence
        // later entries override earlier keys
    }
}
```

### 14.4 Permission Enforcement Gate

```rust
// Every tool call path:
match permission_enforcer.enforce(tool_name, input, &context) {
    EnforcementResult::Allowed => execute_tool(tool_name, input),
    EnforcementResult::Denied(reason) => Err(ToolError::new(reason)),
    EnforcementResult::NeedsPrompt(request) => {
        match prompter.decide(&request) {
            PermissionPromptDecision::Allow => execute_tool(tool_name, input),
            PermissionPromptDecision::Deny => Err(ToolError::new("denied by user")),
        }
    }
}
```

### 14.5 SSE Stream → AssistantEvent Conversion

```rust
// api → runtime boundary

// api emits StreamEvent (raw Anthropic SSE)
// runtime wraps in AssistantEvent (domain-level event)
AssistantEvent::TextDelta(String)
AssistantEvent::ToolUse { id, name, input }
AssistantEvent::Usage(TokenUsage)
AssistantEvent::PromptCache(PromptCacheEvent)
AssistantEvent::MessageStop
```

---

## 15. Architectural Decision Records

### ADR-001: Rust Rewrite of Python/TypeScript CLI

**Context**: The original Claude Code CLI was implemented in Python and TypeScript. Performance, distribution, and safety guarantees were constraints.

**Decision**: Full Rust rewrite with a Cargo workspace, binary `claw`, and library crates per domain.

**Consequences (+)**: Single statically-linked binary, no Python/Node runtime dependency, `unsafe_code = "forbid"` enforced, `clippy::pedantic` linting, direct system call access for sandbox.

**Consequences (−)**: Parity maintenance overhead; `compat-harness` crate exists specifically to track drift against upstream.

---

### ADR-002: Blocking reqwest in API Crate (Not Async)

**Context**: `ConversationRuntime` is called from both synchronous contexts (CLI main thread) and tests.

**Decision**: Use `reqwest::blocking::Client` in the `api` crate rather than `reqwest::Client` (async).

**Consequences (+)**: No `async`/`await` surface in `api` or `runtime` — simpler call stacks, easier testing.

**Consequences (−)**: Cannot use `tokio::spawn` inside `api`; MCP and subprocess management that needs async lives in `runtime` with its own tokio runtime.

---

### ADR-003: Trait Objects for ApiClient, ToolExecutor, PermissionPrompter

**Context**: Tests must substitute real API/tool/permission implementations without network or filesystem.

**Decision**: Define narrow traits; `ConversationRuntime` is parameterized over them.

**Consequences (+)**: `MockAnthropicService` + `MemoryTelemetrySink` enable fully deterministic integration tests. `StaticToolExecutor` (a stub) allows unit-testing the conversation loop.

**Consequences (−)**: Slightly more boilerplate in wiring; `dyn Trait` dispatch overhead (negligible vs. I/O).

---

### ADR-004: Plugin System with Three Marketplaces

**Context**: Different deployment scenarios require different plugin sets: bundled (shipped), builtin (compiled-in), external (user-provided).

**Decision**: `PluginKind { Builtin, Bundled, External }` with separate marketplace roots.

**Consequences (+)**: Clear separation; external plugins don't require recompilation.

**Consequences (−)**: Discovery logic must handle three directory trees.

---

### ADR-005: JSONL Session Storage with File Rotation

**Context**: Conversation sessions can be long; large files hurt startup and export performance.

**Decision**: JSONL format, one record per line, rotate at 256 KB, keep 3 segments.

**Consequences (+)**: Append-only writes, streaming read, easy partial export; human-readable.

**Consequences (−)**: Reading the full session requires concatenating all segments; no random access.

---

### ADR-006: MCP as the Plugin-to-Tool Bridge

**Context**: Third-party tools need a standard integration protocol.

**Decision**: Implement full MCP (Model Context Protocol) with four transport options.

**Consequences (+)**: Ecosystem compatibility with any MCP-speaking server; tool discovery is protocol-level.

**Consequences (−)**: MCP lifecycle management (11 phases, degraded mode, multiple transports) is the most complex subsystem in the codebase.

---

## 16. Architecture Governance

### Enforcement Mechanisms

| Mechanism | Enforced by |
|---|---|
| No unsafe code | `unsafe_code = "forbid"` in workspace Cargo.toml |
| Clippy pedantic | `clippy::pedantic = "warn"` → `-D warnings` in CI |
| Formatting | `cargo fmt --check` in CI |
| Dependency acyclicity | Cargo's workspace resolver; circular deps = compile error |
| Test coverage | `cargo test --workspace` must pass; parity harness validates CLI contracts |
| Parity with upstream | `compat-harness` manifest extraction + `mock_parity_harness.rs` tests |

### Architectural Review Signals

- `PARITY.md` — documents known gaps between Rust and upstream implementations
- `ROADMAP.md` — tracks planned architectural evolution
- `TUI-ENHANCEMENT-PLAN.md` (in `rust/`) — documents planned terminal UI improvements

---

## 17. Blueprint for New Development

### 17.1 Development Workflow

**Starting a new feature**:
1. Identify which crate owns the domain (see §3 Workspace Crate Map)
2. Add types/traits in the owning library crate
3. Wire dispatch in `tools/src/lib.rs` (if tool) or `commands/src/lib.rs` (if slash command)
4. Expose in `rusty-claude-cli/src/main.rs` (CLI surface)
5. Write unit tests in the owning crate's `src/` module
6. Write integration tests in `tests/` (with `mock-anthropic-service` if API is involved)
7. Add parity scenario to `mock_parity_scenarios.json` if CLI output shape changes

**Component creation sequence**:
```
1. Define types/enums (domain model)
2. Define traits (abstraction boundary)
3. Implement types (concrete behavior)
4. Export from crate lib.rs
5. Integrate at call site
6. Test
```

### 17.2 File Organization for New Components

```
crates/<crate-name>/
  src/
    my_feature.rs         ← implementation
    lib.rs                ← pub use my_feature::MyType;
  tests/
    my_feature_tests.rs   ← integration tests
```

### 17.3 Implementation Templates

**New tool**:
```rust
// In tools/src/lib.rs, inside execute_tool()
"my_tool" => {
    let input: MyToolInput = serde_json::from_str(input_json)?;
    // validate permissions
    // execute
    // return String result
}
```

**New slash command**:
```rust
// In commands/src/lib.rs
SlashCommandSpec {
    name: "mycommand",
    aliases: &["mc"],
    summary: "Does the thing",
    argument_hint: Some("<target>"),
    resume_supported: false,
},
```

**New config field**:
```rust
// In runtime/src/config.rs, RuntimeFeatureConfig
pub struct RuntimeFeatureConfig {
    // ...existing fields...
    my_feature: MyFeatureConfig,
}
```

### 17.4 Common Pitfalls

| Pitfall | Avoidance |
|---|---|
| Adding `async` to `api` crate | Keep `api` synchronous; async goes in `runtime` or the binary |
| Adding a workspace dependency cycle | Check dependency graph before adding `path = "../<crate>"` |
| Skipping `PermissionEnforcer` | Every tool execution must pass through the enforcer |
| Using `HashMap` in public structs | Prefer `BTreeMap` for deterministic ordering / test stability |
| `unwrap()` in library code | Use `?` and typed `Result`; `unwrap()` only in tests |
| Ignoring `#[must_use]` | Clippy will catch it — but don't suppress, fix the call site |
| Writing to session without going through `Session` | Use `Session::append_message()`, never raw file writes |
| Hardcoding model names | Use `DEFAULT_MODEL` constant or config; models change |

### 17.5 Testing Requirements by Layer

| Layer | Requirement |
|---|---|
| New tool execution | Unit test for success + error cases; permission test |
| New API interaction | Integration test with `MockAnthropicService` |
| New CLI flag | Test in `cli_flags_and_config_defaults.rs` |
| New JSON output field | Test in `output_format_contract.rs` |
| New slash command | Test in `resume_slash_commands.rs` if `resume_supported` |
| New config field | Unit test in `config.rs` module test; validate with `config_validate` |

---

## Appendix: Key Constants Reference

| Constant | Value | Location |
|---|---|---|
| `DEFAULT_MODEL` | `"claude-opus-4-6"` | `rusty-claude-cli/src/main.rs` |
| `DEFAULT_BASE_URL` | `"https://api.anthropic.com"` | `api/src/providers/anthropic.rs` |
| `DEFAULT_MAX_RETRIES` | `8` | `api/src/providers/anthropic.rs` |
| `DEFAULT_MAX_BACKOFF` | `128 s` | `api/src/providers/anthropic.rs` |
| `ROTATE_AFTER_BYTES` | `256 KiB` | `runtime/src/session.rs` |
| `MAX_ROTATED_FILES` | `3` | `runtime/src/session.rs` |
| `DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD` | `100,000` | `runtime/src/conversation.rs` |
| `DEFAULT_OAUTH_CALLBACK_PORT` | `4545` | `rusty-claude-cli/src/main.rs` |
| `DEFAULT_ANTHROPIC_VERSION` | `"2023-06-01"` | `telemetry/src/lib.rs` |
| `DEFAULT_AGENTIC_BETA` | `"claude-code-20250219"` | `telemetry/src/lib.rs` |
| `CLAW_SETTINGS_SCHEMA_NAME` | `"SettingsSchema"` | `runtime/src/config.rs` |
| `PRIMARY_SESSION_EXTENSION` | `"jsonl"` | `rusty-claude-cli/src/main.rs` |

---

*This blueprint was generated on 2026-04-13. It reflects the codebase as of that date. Update this document when: new crates are added to the workspace, new cross-cutting architectural patterns are introduced, or the dependency graph changes materially.*
