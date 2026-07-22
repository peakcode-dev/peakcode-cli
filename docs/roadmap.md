# peakcode roadmap

This is the product-level roadmap for peakcode. It captures directions we have
committed to plan for, not committed to build. Each item gets its own brainstorm +
spec + plan cycle when it becomes the active focus. Pre-alpha scope (the current
focus) is unaffected unless an item is marked as a present-day design constraint.

## Repo layout (target)

- **peakcode-core** - UI-agnostic engine (provider, tool, agent loop, sessions, config). Shared by every frontend.
- **peakcode-cli** - terminal client (TUI). An access point to the daemon.
- **peakcode-daemon** - long-running host for peakcode-core. Owns agent sessions; survives CLI/SSH disconnect.
- **peakcode-server** (future) - API server for remote agent execution + E2E session sync.
- **peakcode-desktop** (future) - native desktop app, built on peakcode-core.

## Direction 1: peakcode-daemon + thin CLI (planned, brainstorm pending)

The CLI is a thin access point; a separate daemon hosts the engine and runs agent
sessions independently.

- **Why:** a user should be able to close the terminal, drop the SSH session, or
  shut the laptop, and have agents keep working. Today (pre-alpha) the agent lives
  in the CLI process, so closing it kills the work. The daemon fixes that.
- **Shape:** `peakcode-daemon` owns `peakcode-core`, manages long-running sessions,
  and exposes them over a local channel/socket. `peakcode-cli` (the TUI) connects to
  the daemon to view live output, approve tools, and start/resume sessions. Multiple
  CLIs can attach to one daemon; closing a CLI detaches but does not stop the agent.
- **Present-day design constraint (binds core now):** the agent loop in
  peakcode-core must be detachable from any frontend process. Core must not assume a
  CLI/TUI process is its lifetime owner. Concretely: core emits events on a channel
  that any frontend (CLI, daemon, future desktop) can consume or detach from without
  killing the loop. This is already mostly satisfied by core being UI-agnostic and
  event-channel based; the remaining gap is that `Agent::run` currently runs the loop
  inline (to completion before returning the receiver). Making the loop spawnable on
  an independent task is a prerequisite for the daemon and must happen before the CLI
  drives a live UI.

## Direction 2: peakcode-server - remote agents (planned, brainstorm pending)

An API server so a developer can run peakcode on a laptop while the agents actually
execute on a remote server (e.g. a Debian box).

- **Why (killer feature):** the maintainer frequently closes their MacBook; in the
  current workflow that kills the agents. With the server, the laptop is just a
  viewer/controller - agents run on the server and survive the laptop sleeping or
  disconnecting. `/sessions` lists remote sessions alongside local ones.
- **Shape:** the server hosts peakcode-core agent sessions and exposes an API. The
  CLI/desktop connects to it. Sessions sync between client and server with
  end-to-end encryption (the server never sees plaintext session content).
- **Depends on:** the daemon architecture (Direction 1) - the server is essentially a
  networked daemon plus E2E session sync.

## Direction 3: peakcode-desktop (planned, brainstorm pending)

A native desktop application built on peakcode-core. Shares the engine with the CLI
and the daemon; provides its own native UI. Depends on peakcode-core staying
UI-agnostic.

## Direction 4: classic-IDE code editing (future, far out)

The ability to edit code in-project like a classic IDE, inside the peakcode
experience. Far future; no design work yet. Noted so the architecture does not paint
us into a corner (e.g. keep the editor/tool surface extensible).

## Status

- Pre-alpha (current): peakcode-core engine + peakcode-cli TUI, OpenAI provider.
- Next brainstorm: the daemon architecture (Direction 1), once the core engine is in.
