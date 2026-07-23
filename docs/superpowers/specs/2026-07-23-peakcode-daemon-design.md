# peakcode-daemon architecture design

**Date:** 2026-07-23
**Status:** Approved (design)
**Scope:** The daemon that hosts peakcode-core agent sessions independently of any
frontend. This spec covers the local daemon; the networked `peakcode-server` is a future
spec that builds on it.

## Goals

- Agent sessions run in the daemon and **survive the CLI closing, the SSH session dropping,
  or the laptop sleeping.** This is the killer feature.
- The CLI (`peakcode-cli`) is a thin access point: it attaches to a session, renders live
  output, approves tools, and detaches without stopping the agent.
- Multiple clients can attach to one session; detaching one never stops the agent.
- A crashed agent session never takes down other sessions or the daemon.
- Strong isolation and enforceable resource limits per session.
- The daemon's protocol generalizes to the future networked server.

## Decisions (confirmed in brainstorm)

1. **Sequencing:** daemon first, then CLI (validate the daemon boundary before the TUI).
2. **Protocol:** gRPC (protobuf) over a Unix domain socket locally, TLS over the network
   for the future server. Chosen over HTTP/JSON-RPC because bidirectional streaming,
   multiplexing, backpressure, and compile-time typing are first-class - exactly this
   workload's sweet spot.
3. **Execution:** supervised subprocess per session (actor/supervisor model). The daemon is
   a supervisor + IPC gateway; each session is an isolated worker process. Chosen over
   in-process tokio tasks for real isolation, crash containment, enforceable resource limits
   (cgroups), and clean per-session lifecycle.

## Non-goals

- `peakcode-server` (remote/networked agents, E2E session sync) - separate spec; this daemon
  is its foundation.
- `peakcode-cli` TUI - separate plan; consumes this daemon.
- `peakcode-desktop`, classic-IDE editing - far future.
- Public REST/HTTP API for third parties - `grpc-gateway` can add it later if needed.
- Multimodal model OUTPUT (images emitted by the model) - rare for coding agents.

## Process architecture

Two roles, one binary:

- **Daemon process** (`peakcode-daemon`, default mode): supervisor + gRPC server + session
  registry + session store (SQLCipher). It does **not** run agent loops. It owns:
  - the listening UDS gRPC socket,
  - the session registry (`session_id -> WorkerHandle`),
  - the persistent session store (messages, metadata),
  - supervision of worker processes,
  - fan-out of worker events to attached clients.
- **Worker process** (`peakcode-daemon worker --session <id>`): the daemon fork+execs itself
  in worker mode. A worker:
  - runs exactly one peakcode-core `Agent` session,
  - owns the `ToolRegistry`, `ToolContext`, and a `RemoteApprover`,
  - streams `AgentEvent`s live to the daemon over a dedicated channel,
  - receives forwarded client commands (input, approvals, cancel) from the daemon,
  - exits when the session ends or it is stopped/crashes.

Rationale: one artifact, two modes (like systemd/postgres). The worker is the unit of
isolation, lifecycle, and resource limits.

## Lifecycle

- **Primary:** a systemd user service (`~/.config/systemd/user/peakcode-daemon.service`)
  with **socket activation** on the UDS. The daemon starts on first connect and survives
  user logout (lingering). This is the production-grade, zero-debt lifecycle.
- **Fallback:** if systemd is unavailable, the CLI auto-spawns the daemon on first connect
  (`peakcode-daemon daemonize` forks and detaches) when the UDS is not present.
- **Explicit control:** `peakcode-daemon start | stop | status` and the equivalent
  `peakcode daemon ...` subcommands exposed through the CLI.

## Session registry

In-memory map `session_id (UUID) -> WorkerHandle { child: Child, events_rx, metadata,
attached_clients }`, plus the persistent store. Operations: create, list, get, stop. The
registry is rebuilt from the store on daemon restart (sessions persist; workers are
re-spawned lazily on attach).

## gRPC API surface (sketch)

A single protobuf `peakcode.proto`. Unary RPCs for control, one bidirectional stream for
attachment.

```
service SessionService {
  rpc CreateSession(CreateSessionRequest) returns (Session);
  rpc ListSessions(Empty) returns (SessionList);
  rpc GetSession(SessionId) returns (Session);
  rpc StopSession(SessionId) returns (Empty);
}

service AttachService {
  // Bidirectional: client sends commands, daemon streams session events.
  rpc Attach(stream ClientCommand) returns (stream DaemonEvent);
}
```

- `ClientCommand { session_id, oneof body { SendInput{text}, ApproveTool{call_id, decision}, Cancel, Detach } }`
- `DaemonEvent { session_id, seq, oneof body { TextDelta, AssistantMessage, ToolStart, ToolResult, NeedsApproval{call_id, tool, arguments}, TurnFinished, SessionEnded, Error } }`
- Worker-to-daemon is a separate bidirectional channel (worker emits events, daemon sends
  forwarded commands) over the worker's pipe/socket.

The `seq` field on `DaemonEvent` supports replay on re-attach (the daemon buffers a recent
window per session).

## Attach / detach

- Multiple clients may attach to one session; the daemon **fans out** events to all attached
  streams.
- Detach drops the client stream only; the worker keeps running.
- Re-attach opens a new `Attach` stream and replays from the client's last-seen `seq` (the
  daemon keeps a bounded event buffer per session).
- `NeedsApproval` is delivered to all attached clients; the first `ApproveTool` response
  resolves it (the worker's `RemoteApprover` resolves the pending promise).

## Approval flow (cross-process)

The worker implements the peakcode-core `Approver` trait as a `RemoteApprover`:
- on `Approval::Required`, the worker emits a `NeedsApproval` event and awaits a one-shot
  `oneshot::Receiver<ApprovalDecision>`,
- the daemon forwards the event to attached clients,
- a client sends `ApproveTool { call_id, decision }`,
- the daemon forwards the decision to the worker, which fulfills the `oneshot`,
- the worker's agent loop resumes.

`AllowAll` is remembered per session in the worker (in-memory), as today.

## Event streaming (resolves the core inline gap)

peakcode-core's `Agent::run` currently runs the loop inline and returns the event receiver
only after completion. In the worker, the loop is spawned on a tokio task and events are
forwarded to the daemon **as they happen**; the daemon fans them out to clients live. This
gives incremental token rendering and tool-start-before-execute with no buffering-to-end-of-turn.

Prerequisite: `Agent::run` must be spawnable (detachable from the calling task). This is a
hard prerequisite addressed in the daemon plan (the worker wraps `Agent::run` so it produces
events on a channel the daemon drains concurrently).

## Crash recovery / supervision

- The daemon supervises each worker process. On unexpected exit:
  - the crash is logged (with the worker's last span context via `tracing`),
  - a `SessionEnded { reason: crashed }` event is fanned out to attached clients,
  - the worker is restarted per a restart policy (max N restarts per rolling window),
  - on restart the worker reloads the session message history from SQLCipher and the agent
    continues from the last persisted assistant turn.
- A worker crash never affects other workers or the daemon (process isolation).
- Daemon crash: systemd restarts it; the registry is rebuilt from the store; workers are
  re-spawned lazily on attach.

## Config & secrets

- Config: `~/.config/peakcode/config.toml` (read by the daemon via peakcode-core `Config`).
- OpenAI API key: fetched from the **OS keyring** by the daemon at worker spawn and passed
  to the worker via its environment (never written to disk in plaintext).
- Session DB key (SQLCipher): from the OS keyring (see structural refactor #3).

## Observability

- `tracing` spans across session, turn, provider call, tool execution, in both daemon and
  worker.
- Structured logs to journald (when under systemd) or a rotating file.
- Foundation for Prometheus metrics (active sessions, token usage, latency, restarts) -
  added later, not blocking.

## Prerequisites (must land before/with the daemon)

In peakcode-core (the parked `feat/core-structural-upgrades` branch):
1. **Multimodal content blocks** (`Message.content: Vec<ContentBlock>`) - the worker carries
   user input including images.
2. **Typed errors** (`ProviderError`, `ConfigError`, `SessionError`) - the daemon retries on
   `RateLimit`, re-prompts on `Auth`, surfaces `BadResponse`.
3. **SQLCipher** - persistent, encrypted session store.

In peakcode-core (daemon work):
4. **Detachable `Agent::run`** - the loop spawns on a task and emits events on a channel the
   caller drains concurrently (fixes the known inline-run gap; required for live streaming
   and for the worker model).

## Repo / crate

New repo `peakcode-daemon` (sibling to `peakcode-core` and `peakcode-cli`), created on
`git.peakssh.dev/peakcode/peakcode-daemon.git` + GitHub mirror `peakcode-dev/peakcode-daemon`,
with the same scaffold (AGENTS.md naming rule + doc conventions, SECURITY, CONTRIBUTING,
LICENSE, pgp_public.asc). Single binary crate depending on `peakcode-core` via sibling path
dep. Uses `tonic` (gRPC), `tokio`, `prost` (protobuf), `tracing`, `keyring`.

## Cross-cutting

- All public contracts (the gRPC `.proto`, the daemon/worker protocol, the `RemoteApprover`)
  require `/docs` updates on change (AGENTS.md rule).
- Naming: `peakcode` lowercase everywhere; repo slug `peakcode-daemon`.
- Commits: `git commit -s`, Conventional Commits. No em dashes.

## Status

- Design approved.
- Next: implementation plan (writing-plans) covering the prerequisites refactor + the daemon.
- The structural refactors (#1-3) execute first, then the daemon crate, then the detachable
  `Agent::run`, then supervision/streaming/attach/approval.
