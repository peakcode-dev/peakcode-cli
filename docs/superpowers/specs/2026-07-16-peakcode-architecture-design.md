# peakcode architecture design

**Date:** 2026-07-16
**Status:** Approved (pre-alpha)
**Scope:** Pre-alpha milestone. Establishes the two-repo split, core abstractions, CLI
TUI, and the minimal working agent loop. Explicitly defers multi-provider, MCP, plugins,
encryption, and theming.

## Goals

- A terminal AI coding agent written in Rust, modeled on [opencode](https://opencode.ai).
- A shared engine (`peakcode-core`) usable by both this CLI and a future desktop app.
- OpenAI as the first and only provider for pre-alpha.
- Default terminal background is `none`: peakcode never forces a background color and
  adapts to whatever the terminal reports.
- Ship a usable pre-alpha: chat, streaming, tool calls with approval, basic file/bash
  tools, config, session persistence.

## Non-goals (pre-alpha)

- Anthropic / local / other providers (the `Provider` trait is built for them, but only
  OpenAI is implemented).
- MCP server integration.
- Plugin system.
- Session encryption, keyring-managed keys (env var only for pre-alpha).
- Rich theming / color schemes (background stays `none`; opt-in themes come later).
- Non-interactive / headless mode.

## Repository layout

Two repos. Both follow the peakssh conventions: lowercase `peakcode` naming, `git commit -s`,
Conventional Commits, em-dash ban, `/docs` authoring rules, same scaffold (AGENTS.md,
SECURITY.md, CONTRIBUTING.md, LICENSE, pgp_public.asc).

### `peakcode-core` (separate repo, library crate)

UI-agnostic engine. Depends only on tokio, reqwest, serde, sqlite, and std. No TUI deps.

Modules:
- `provider/` - `Provider` trait + `openai` implementation.
- `tool/` - `Tool` trait + built-in tools (`read`, `write`, `edit`, `bash`, `grep`, `glob`).
- `agent/` - the agent loop: stream -> render -> approve -> execute -> re-stream.
- `session/` - conversation storage (SQLite) + resume.
- `config/` - config model + discovery.
- `message/` - message / content-block / tool-call types shared across providers.

### `peakcode` (this repo, binary crate)

The CLI. Depends on `peakcode-core` via a sibling path dependency (`../peakcode-core`).

Modules:
- `cli/` - arg parsing (clap), config discovery, entry point.
- `ui/` - ratatui + crossterm TUI (background `none`), `Approver` implementation.

### Future: `peakcode-desktop`

Will depend on `peakcode-core` and provide its own native UI (not part of this spec).

## Dependency mechanism

Sibling path dependency for local development, consistent with the peakssh ecosystem
(peakssh-plugin-runtime, peakssh-wg-stack are all consumed as sibling path deps). In
`peakcode/Cargo.toml`:

```toml
[dependencies]
peakcode-core = { path = "../peakcode-core", version = "0.1.0" }
```

Git dependency / crates.io publishing is a post-pre-alpha concern.

## Core abstractions (peakcode-core)

### Provider trait

```rust
#[async_trait]
pub trait Provider: Send + Sync {
    fn name(&self) -> &str;
    async fn stream(
        &self,
        messages: &[Message],
        tools: &[ToolSchema],
        config: &ProviderConfig,
    ) -> Result<Pin<Box<dyn Stream<Item = Result<ProviderEvent>> + Send>>>;
}
```

`ProviderEvent` is a unified stream of: text delta, tool-call start, tool-call args delta,
tool-call end, finish. The OpenAI impl maps SSE chunks to this. Adding Anthropic later is a
new impl behind the same trait - no changes to the agent loop.

### Tool trait

```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn schema(&self) -> serde_json::Value;   // JSON schema sent to the model
    fn requires_approval(&self, params: &Value) -> Approval; // read-only? mutating? bash?
    async fn execute(&self, params: Value, ctx: &ToolContext) -> Result<ToolResult>;
}
```

Built-in tools:
- `read` - read a file (no approval).
- `glob` - file pattern match (no approval).
- `grep` - content search (no approval).
- `write` - create/overwrite a file (approval).
- `edit` - string replacement edit (approval).
- `bash` - run a shell command (approval, with a confirmation of the command).

`ToolContext` carries the project root, config, and a capability to request approval from
the host. Tools are UI-agnostic; they never print to a terminal directly.

### Approver (host hook)

```rust
#[async_trait]
pub trait Approver: Send + Sync {
    async fn approve(&self, request: ApprovalRequest) -> ApprovalDecision;
}
```

Core calls `Approver` when a tool needs approval. The CLI implements an inline TUI prompt;
the future desktop implements its own dialog. Core never blocks on a specific UI.

### Agent loop

1. Build messages + tool schemas, call `provider.stream(...)`.
2. Forward text deltas to a render channel (consumed by the TUI).
3. On tool-call end, ask `Approver` (if required). On approval, execute the tool.
4. Append the `tool_result` message.
5. Re-stream from step 1 until the model returns no further tool calls.
6. Persist the session.

The loop is fully async (tokio). It does not own a terminal; it emits events on a channel.

## CLI design (peakcode)

### TUI (ratatui + crossterm)

- Single chat view: scrolling message log (user / assistant / tool), streaming render, input
  box pinned to the bottom, status footer (model, token usage, session id).
- **Background `none`:** never paint `Color::Reset`/`Color::Black`/etc. as a background.
  Use `Style::default()` for backgrounds so the terminal default shows through. No hard-coded
  theme colors in pre-alpha.
- Main thread runs the ratatui event loop; core runs on tokio tasks; communication via
  `tokio::sync::mpsc` (UI events in, render deltas out).

### CLI surface (clap)

- `peakcode` - start an interactive session in the current project.
- `peakcode resume [id]` - resume a stored session.
- `peakcode --model <id>` - override the configured model.
- `PEAKCODE_OPENAI_API_KEY` env var for the API key (pre-alpha).

### Approval UX

Inline prompt rendered in the TUI when a tool needs approval: shows the tool name, params
(for `bash`, the exact command), and `[y] allow / [n] deny / [a] allow-all` keys. Maps to the
core `Approver` trait.

## Session persistence

SQLite via `rusqlite`, stored at `~/.local/share/peakcode/sessions.db`. Schema: sessions,
messages, tool_calls. `peakcode resume` lists/restores past sessions. Encryption is deferred.

## Config

- `~/.config/peakcode/config.toml` (user) + `peakcode.toml` (project, optional).
- Keys: `provider`, `model`, `max_tokens`, system prompt overrides, tool toggles.
- API key via env var only for pre-alpha; keyring integration is post-pre-alpha.

## Dependencies

peakcode-core: `tokio`, `async-trait`, `reqwest` (streaming), `serde`/`serde_json`,
`rusqlite`, `toml`, `anyhow`, `thiserror`, `futures`.

peakcode: `peakcode-core`, `tokio`, `ratatui`, `crossterm`, `clap`, `anyhow`.

## Cross-cutting

- All public/host-facing contracts (Provider trait, Tool trait, Approver, config schema)
  require `/docs` updates on change (see AGENTS.md "Documentation Conventions").
- Naming: `peakcode` lowercase everywhere.
- Commits: `git commit -s`, Conventional Commits.
- No em dashes in any text.
