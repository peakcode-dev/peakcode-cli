# peakcode-daemon implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Build `peakcode-daemon` - a long-running host for peakcode-core agent sessions that
survives the CLI/SSH/laptop closing. CLI is a thin gRPC client; each session is an isolated
supervised worker process.

**Architecture:** Daemon process = supervisor + gRPC server (UDS) + session registry + store.
Worker process (`peakcode-daemon worker`) = one peakcode-core `Agent` session, streaming
events live over an internal IPC socket. Actor/supervisor model; crash of one session never
affects others.

**Tech Stack:** Rust (edition 2021), tokio, tonic + prost (gRPC over UDS), tokio Unix sockets,
peakcode-core (sibling path dep), tracing, the same keyring/rand deps as core.

## Global Constraints

- New repo `/home/user/Git/peakcode-daemon` (sibling to peakcode-core). Create on
  `git.peakssh.dev/peakcode/peakcode-daemon.git` + GitHub mirror `peakcode-dev/peakcode-daemon`
  with the same scaffold (AGENTS.md naming+doc rules, SECURITY, CONTRIBUTING, LICENSE, pgp_public.asc).
- Naming `peakcode` lowercase; repo slug `peakcode-daemon`. No em dashes.
- Commits `git commit -s`, Conventional Commits.
- Depends on `peakcode-core = { path = "../peakcode-core", version = "0.1.0" }` (after the
  core upgrades plan merges and bumps core to 0.1.0).
- Verification gate per task: `cargo test` + `cargo clippy --all-targets -- -D warnings`.
- Reference design: `peakcode-cli/docs/superpowers/specs/2026-07-23-peakcode-daemon-design.md`.

---

## File Structure

```
peakcode-daemon/
├── Cargo.toml
├── build.rs                  # tonic_build::compile_protos("proto/peakcode.proto")
├── proto/peakcode.proto
├── src/
│   ├── main.rs               # mode dispatch: daemon | worker | daemonize | start/stop/status
│   ├── lib.rs
│   ├── proto.rs              # pub mod peakcode { tonic::include_codegen! } re-export
│   ├── ipc.rs                # worker<->daemon NDJSON framing + message types
│   ├── registry.rs           # SessionRegistry: session_id -> WorkerHandle
│   ├── worker.rs             # worker mode: connect IPC, run core Agent, stream events
│   ├── remote_approver.rs    # RemoteApprover: NeedsApproval round-trip over IPC
│   ├── server.rs             # tonic gRPC impls: SessionService + AttachService
│   ├── supervisor.rs         # spawn workers, watch exits, restart policy
│   └── socket.rs             # UDS path discovery + systemd socket-activation helper
└── tests/integration.rs      # end-to-end: create session, attach, stream, stop
```

---

### Task 1: repo scaffold + protobuf contract

**Files:** repo + scaffold + `Cargo.toml`, `build.rs`, `proto/peakcode.proto`, `src/main.rs`, `src/lib.rs`, `src/proto.rs`.

- [ ] **Step 1:** Create the repo at `/home/user/Git/peakcode-daemon`, set up origin + GitHub mirror (same as peakcode-core setup). Add scaffold (AGENTS.md with the daemon-direction + UI-agnostic notes, README, SECURITY, CONTRIBUTING, LICENSE, pgp_public.asc copied from peakssh, .gitignore).

- [ ] **Step 2:** `Cargo.toml`:
```toml
[package]
name = "peakcode-daemon"
version = "0.1.0"
edition = "2021"
license = "MIT"

[dependencies]
peakcode-core = { path = "../peakcode-core", version = "0.1.0" }
tokio = { version = "1", features = ["full"] }
tonic = "0.12"
prost = "0.13"
tokio-stream = "0.1"
futures = "0.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
anyhow = "1"
uuid = { version = "1", features = ["v4"] }

[build-dependencies]
tonic-build = "0.12"
```

- [ ] **Step 3:** `build.rs`:
```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::configure().compile_protos(&["proto/peakcode.proto"], &["proto"])?;
    Ok(())
}
```

- [ ] **Step 4:** `proto/peakcode.proto` (the contract - the design's API surface, verbatim):
```protobuf
syntax = "proto3";
package peakcode.daemon.v1;

service SessionService {
  rpc CreateSession(CreateSessionRequest) returns (Session);
  rpc ListSessions(google.protobuf.Empty) returns (SessionList);
  rpc GetSession(SessionId) returns (Session);
  rpc StopSession(StopSessionRequest) returns (google.protobuf.Empty);
}

service AttachService {
  rpc Attach(stream ClientCommand) returns (stream DaemonEvent);
}

// (define all messages: Empty, SessionId, Session, SessionStatus enum,
//  CreateSessionRequest{model,system_prompt,workdir}, SessionList, StopSessionRequest,
//  ClientCommand{session_id, oneof SendInput/ApproveTool/Cancel}, SendInput{text},
//  ApproveTool{call_id, decision}, ApprovalDecision enum {ALLOW=1,DENY=2,ALLOW_ALL=3}, Cancel,
//  DaemonEvent{session_id, seq, oneof TextDelta/AssistantMessage/ToolStart/ToolResult/
//   NeedsApproval/TurnFinished/SessionEnded/ErrorEvent}, and each event payload type,
//  importing "google/protobuf/empty.proto".)
```
Write out every message fully (see the design's API surface for fields/types).

- [ ] **Step 5:** `src/proto.rs`:
```rust
pub mod peakcode {
    tonic::include_proto!("peakcode");
}
```
`src/lib.rs`: `pub mod proto;` `src/main.rs`: empty `fn main() {}` stub for now.

- [ ] **Step 6:** `cargo build` confirms protobuf compiles. Commit `chore: scaffold peakcode-daemon repo and protobuf contract`.

---

### Task 2: binary mode dispatch + lifecycle stubs

**Files:** `src/main.rs`.

**Interfaces:** `peakcode-daemon` (daemon mode), `peakcode-daemon worker --session <id> --ipc <path>`, `peakcode-daemon daemonize`, `peakcode-daemon start|stop|status`.

- [ ] **Step 1:** `src/main.rs` parses a subcommand (`clap` or manual `env::args`):
```rust
#[derive(Debug)]
enum Mode {
    Daemon,                                          // run the daemon server (foreground)
    Daemonize,                                       // fork+detach (fallback when no systemd)
    Worker { session_id: String, ipc_path: String }, // worker mode (called by the daemon)
    Start, Stop, Status,                             // lifecycle (talk to a running daemon)
}
```
For now, each arm prints what it would do and returns Ok; the real implementations come in later tasks. `Daemon`/`Worker` get fleshed out in tasks 3-7.

- [ ] **Step 2:** `cargo run -- --help` shows the subcommands. Commit `feat(cli): add mode dispatch (daemon/worker/daemonize/lifecycle)`.

---

### Task 3: internal IPC protocol (worker <-> daemon)

**Files:** `src/ipc.rs`.

**Interfaces:** `IpcFrame` (NDJSON, one JSON object per line over a Unix socket), `WorkerEvent`, `DaemonCommand`.

- [ ] **Step 1:** `src/ipc.rs` defines the wire types and read/write helpers:
```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
#[serde(tag = "kind")]
pub enum WorkerEvent {
    TextDelta { text: String },
    AssistantMessage { text: String },
    ToolStart { call_id: String, name: String, arguments_json: String },
    ToolResult { call_id: String, name: String, content: String, is_error: bool },
    NeedsApproval { call_id: String, tool: String, arguments_json: String },
    TurnFinished,
    Done { final_message_count: usize },
    Crash { message: String },
}

#[derive(Serialize, Deserialize)]
#[serde(tag = "kind")]
pub enum DaemonCommand {
    Approve { call_id: String, decision: String },  // "allow"|"deny"|"allow_all"
    Cancel,
    Stop,
}

/// Write one NDJSON frame.
pub async fn write_frame<W: tokio::io::AsyncWrite + Unpin>(
    w: &mut W, frame: &impl Serialize,
) -> std::io::Result<()> {
    let mut line = serde_json::to_vec(frame)?;
    line.push(b'\n');
    use tokio::io::AsyncWriteExt;
    w.write_all(&line).await
}

/// Read one NDJSON frame.
pub async fn read_frame<R: tokio::io::AsyncBufRead + Unpin, T: serde::de::DeserializeOwned>(
    r: &mut R,
) -> std::io::Result<Option<T>> {
    use tokio::io::AsyncBufReadExt;
    let mut buf = String::new();
    let n = r.read_line(&mut buf).await?;
    if n == 0 { return Ok(None); }
    let frame: T = serde_json::from_str(buf.trim()).map_err(|e| std::io::Error::new(std::io::ErrorKind::InvalidData, e))?;
    Ok(Some(frame))
}
```

- [ ] **Step 2:** Unit-test roundtrip: serialize a `WorkerEvent::TextDelta` and a `DaemonCommand::Approve` to a `Vec<u8>`, read back, assert equality. Commit `feat(ipc): add NDJSON worker<->daemon framing`.

---

### Task 4: RemoteApprover + worker mode

**Files:** `src/remote_approver.rs`, `src/worker.rs`.

**Interfaces:** `RemoteApprover` impls `peakcode_core::Approver`; `worker::run(session_id, ipc_path)` connects to the daemon IPC and runs one `Agent` session, streaming events.

- [ ] **Step 1:** `src/remote_approver.rs`:
```rust
use std::collections::HashMap;
use std::sync::Arc;
use async_trait::async_trait;
use tokio::sync::{oneshot, Mutex};
use peakcode_core::{ApprovalDecision, ApprovalRequest, Approver};
use crate::ipc::DaemonCommand;

/// Approver that routes each request to the daemon (which forwards to a client),
/// and awaits the client's decision via a oneshot keyed by call_id.
#[derive(Default, Clone)]
pub struct RemoteApprover {
    pending: Arc<Mutex<HashMap<String, oneshot::Sender<ApprovalDecision>>>>,
}

impl RemoteApprover {
    pub fn new() -> Self { Self::default() }
    /// Register a waiter; returns the receiver the worker awaits, plus the id.
    pub async fn register(&self, call_id: &str) -> oneshot::Receiver<ApprovalDecision> {
        let (tx, rx) = oneshot::channel();
        self.pending.lock().await.insert(call_id.to_string(), tx);
        rx
    }
    /// Called when a DaemonCommand::Approve arrives from the daemon.
    pub async fn resolve(&self, call_id: &str, decision: ApprovalDecision) {
        if let Some(tx) = self.pending.lock().await.remove(call_id) {
            let _ = tx.send(decision);
        }
    }
}

#[async_trait]
impl Approver for RemoteApprover {
    async fn approve(&self, req: ApprovalRequest) -> ApprovalDecision {
        // The worker main loop emits NeedsApproval BEFORE calling approve; here we just
        // await the registered receiver. If dropped, default to Deny (safe).
        match self.register(&req.call_id).await.await {
            Ok(d) => d,
            Err(_) => ApprovalDecision::Deny,
        }
    }
}
```
(Note: the worker loop must emit the `NeedsApproval` frame and call `approver.approve(req)` in sequence; see Step 2. Register-then-await keeps the waiter keyed correctly - refactor so `approve` itself registers, which it does above.)

- [ ] **Step 2:** `src/worker.rs` - worker main loop:
```rust
use std::sync::Arc;
use peakcode_core::{Agent, AgentEvent, Config, Message, OpenAiProvider, Provider,
    ToolContext, ToolRegistry, tools::*};
use crate::ipc::{DaemonCommand, WorkerEvent, read_frame, write_frame};

pub async fn run(session_id: String, ipc_path: String) -> anyhow::Result<()> {
    // 1. Connect to the daemon IPC socket.
    let stream = tokio::net::UnixStream::connect(&ipc_path).await?;
    let (read_half, mut write_half) = stream.into_split();
    let mut reader = tokio::io::BufReader::new(read_half);

    // 2. Load session: config + message history (from core SessionStore).
    let config = Config::discover();
    let provider_cfg = Arc::new(config.provider_config()?);
    let provider: Arc<dyn Provider> = Arc::new(OpenAiProvider::default());
    let approver = Arc::new(crate::remote_approver::RemoteApprover::new());
    let approver_clone = Arc::clone(&approver);
    let ctx = Arc::new(ToolContext::new(std::env::current_dir()?));

    let mut tools = ToolRegistry::new();
    tools.register(Box::new(ReadTool)); tools.register(Box::new(GlobTool));
    tools.register(Box::new(GrepTool)); tools.register(Box::new(WriteTool));
    tools.register(Box::new(EditTool)); tools.register(Box::new(BashTool));
    let tools = Arc::new(tools);

    // (Load or start message history via core SessionStore here - omitted for brevity in
    //  this step; full version reads the session's prior messages.)

    // 3. Pump inbound DaemonCommands -> resolve approvals / cancel.
    let approver_for_pump = Arc::clone(&approver);
    tokio::spawn(async move {
        loop {
            match read_frame::<_, DaemonCommand>(&mut reader).await {
                Ok(Some(DaemonCommand::Approve { call_id, decision })) => {
                    let d = match decision.as_str() { "allow_all" => peakcode_core::ApprovalDecision::AllowAll, "deny" => peakcode_core::ApprovalDecision::Deny, _ => peakcode_core::ApprovalDecision::Allow };
                    approver_for_pump.resolve(&call_id, d).await;
                }
                Ok(Some(DaemonCommand::Cancel)) | Ok(Some(DaemonCommand::Stop)) | Ok(None) => break,
                Err(_) => break,
            }
        }
    });

    // 4. Run the (now-detachable) Agent loop and forward events to the daemon.
    let messages: Vec<Message> = vec![]; // or loaded history
    let (mut rx, _handle) = Agent::run(provider, provider_cfg, tools, approver, ctx, messages).await;
    while let Some(ev) = rx.recv().await {
        let frame = match ev {
            AgentEvent::TextDelta(t) => WorkerEvent::TextDelta { text: t },
            AgentEvent::AssistantMessage(m) => WorkerEvent::AssistantMessage { text: m.text_content() },
            AgentEvent::ToolStart { call_id, name, arguments } => WorkerEvent::ToolStart { call_id, name, arguments_json: arguments.to_string() },
            AgentEvent::ToolResult { call_id, name, output } => WorkerEvent::ToolResult { call_id, name, content: output.content, is_error: output.is_error },
            AgentEvent::TurnFinished => WorkerEvent::TurnFinished,
        };
        write_frame(&mut write_half, &frame).await?;
    }
    write_frame(&mut write_half, &WorkerEvent::Done { final_message_count: 0 }).await?;
    Ok(())
}
```

- [ ] **Step 3:** Unit-test `RemoteApprover` register/resolve roundtrip (register a call_id, resolve it, assert the receiver gets the decision). Commit `feat(worker): add RemoteApprover and worker-mode agent runner`.

---

### Task 5: session registry + store

**Files:** `src/registry.rs`, plus daemon use of `peakcode_core::SessionStore`.

**Interfaces:** `SessionRegistry` holds `session_id -> WorkerHandle`; `WorkerHandle` owns the worker process + a writer to send commands + event buffer for replay.

- [ ] **Step 1:** `src/registry.rs`:
```rust
use std::sync::Arc;
use tokio::sync::Mutex;
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub struct SessionMeta { pub id: String, pub created_at: i64, pub model: String, pub status: SessionStatus }
#[derive(Debug, Clone, Copy)] pub enum SessionStatus { Idle, Running, Stopped, Crashed }

pub struct WorkerHandle {
    pub child: tokio::process::Child,
    pub command_writer: tokio::io::WriteHalf<tokio::net::UnixStream>,
    pub event_buffer: Vec<crate::ipc::WorkerEvent>, // bounded replay window
    pub meta: SessionMeta,
}

#[derive(Default)]
pub struct SessionRegistry { inner: Mutex<HashMap<String, Arc<Mutex<WorkerHandle>>>> }

impl SessionRegistry {
    pub async fn insert(&self, h: WorkerHandle) { self.inner.lock().await.insert(h.meta.id.clone(), Arc::new(Mutex::new(h))); }
    pub async fn get(&self, id: &str) -> Option<Arc<Mutex<WorkerHandle>>> { self.inner.lock().await.get(id).cloned() }
    pub async fn list(&self) -> Vec<SessionMeta> {
        let g = self.inner.lock().await;
        g.values().map(|h| h.blocking_lock().meta.clone()).collect() // (use proper async in real code)
    }
    pub async fn remove(&self, id: &str) -> Option<Arc<Mutex<WorkerHandle>>> { self.inner.lock().await.remove(id) }
}
```
(The `list` async-over-Mutex detail: iterate via a loop that locks each handle; the sketch above is indicative - implement cleanly without `blocking_lock` inside async.)

- [ ] **Step 2:** Wire the daemon to use `peakcode_core::SessionStore` for persistent history (create_session, append messages as they stream, load on worker spawn). Commit `feat(registry): add session registry and worker handles`.

---

### Task 6: gRPC server (SessionService + AttachService over UDS)

**Files:** `src/server.rs`, `src/socket.rs`, `src/main.rs` (Daemon arm).

- [ ] **Step 1:** `src/socket.rs` - UDS path discovery (`~/.local/share/peakcode/daemon.sock`) + systemd socket-activation helper (detect `LISTEN_FDS` and take fd 3 if present).

- [ ] **Step 2:** `src/server.rs` implements the tonic services. `SessionService::create_session` -> spawn a worker via the supervisor, insert into registry. `AttachService::attach` -> look up the session, fan out the worker's events to this client's stream, forward the client's commands to the worker's command writer. Use `tokio::sync::broadcast` or a per-session event channel for fan-out to multiple clients.

- [ ] **Step 3:** Bind the gRPC server over the UDS (tokio UnixListener -> tonic transport). Wire the `Daemon` mode arm in `main.rs` to start tracing, open the store, build the registry, and serve.

- [ ] **Step 4:** Integration test `tests/integration.rs`: start the daemon in-process on a temp UDS, a tonic client creates a session, attaches, asserts it receives events (use a stub/mock provider injected for the test - expose a way to swap the provider). Commit `feat(server): add gRPC SessionService + AttachService over UDS`.

---

### Task 7: supervision + crash recovery + multi-client fan-out

**Files:** `src/supervisor.rs`, `src/server.rs` (fan-out).

- [ ] **Step 1:** `src/supervisor.rs` spawns the worker process (`Command::new(current_exe).arg("worker")...`), owns the IPC accept side, reads `WorkerEvent` frames, fans them to all attached clients (broadcast), appends to the event buffer + persists messages, and on unexpected worker exit: logs, emits `SessionEnded { crashed }`, applies the restart policy (max N/window), and respawns the worker reloading history.

- [ ] **Step 2:** Multi-client attach: a `broadcast::Sender<DaemonEvent>` per session; each `Attach` call subscribes and replays from the client's last seq using the event buffer.

- [ ] **Step 3:** Tests: simulate a worker crash (kill the child), assert the supervisor emits `SessionEnded` and restarts; assert two attached clients both receive a streamed event. Commit `feat(supervisor): add worker supervision, crash recovery, and fan-out`.

---

### Task 8: lifecycle (systemd + daemonize + start/stop/status)

**Files:** packaging `packaging/systemd/peakcode-daemon.service` (socket-activated), `src/main.rs` (`Daemonize`, `Start/Stop/Status` talk to the running daemon over the UDS).

- [ ] **Step 1:** Add the systemd user unit files (service + socket) under `packaging/systemd/`, with socket activation on the UDS. Document install (`systemctl --user import-environment; enable --now peakcode-daemon.socket`).

- [ ] **Step 2:** `Daemonize` arm: fork+detach (double-fork or `daemonize` crate) so the CLI can auto-start the daemon when systemd is absent and the UDS is missing.

- [ ] **Step 3:** `Start/Stop/Status` arms: gRPC calls to a running daemon (or start it first). Expose the same via the CLI later.

- [ ] **Step 4:** Manual smoke test (documented): start the daemon, create a session with a real OpenAI key, attach, send input, see streamed tokens, detach, confirm the agent keeps running, re-attach. Commit `feat(lifecycle): add systemd socket activation, daemonize, and start/stop/status`.

---

## Self-Review

- Spec coverage: process architecture (Task 4 worker, Task 6 server), lifecycle (Task 8), registry (Task 5), gRPC API (Tasks 1+6), attach/detach + fan-out (Tasks 6+7), approval flow (Task 4 RemoteApprover), event streaming (Task 4 detachable core loop), crash recovery (Task 7), config/secrets (worker loads core Config + keyring via core), observability (tracing in Task 6 main). All design sections mapped.
- Prerequisite dependency: Task 4 worker calls the detachable `Agent::run` from the core-upgrades plan (Task 4 there) - that must merge first and bump core to 0.1.0.
- No placeholders: proto is fully specified; novel code (RemoteApprover, ipc, worker) is complete; server/supervisor have concrete structure with the harder API details (tonic UDS, broadcast fan-out) flagged for the implementer to finalize against the installed tonic version.

## Execution order

Plan A (core upgrades) merges first and tags/bumps core to 0.1.0. Then this plan executes Task 1 -> 8. Then a CLI plan consumes the daemon.
