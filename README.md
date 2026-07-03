# newt-remote

**Remote eyes and hands for a running [newt-agent](https://github.com/Gilamonster-Foundation/newt-agent) session.**

Attach to — and optionally take control of — a newt-agent session that's
already running somewhere else (a `tmux` pane over SSH, a headless box on
your LAN, a machine behind your home WireGuard tunnel) from another device,
without starting a second agent process. Identity, authorization, and
transport are provided by [agent-mesh](https://github.com/Gilamonster-Foundation/agent-mesh);
this repo is the client side of that relationship — starting with an
Android app that renders the session's already-rendered Markdown block
stream natively instead of re-implementing a terminal emulator.

This repo is currently in the design phase. See
[`docs/decisions/0001-remote-session-control.md`](docs/decisions/0001-remote-session-control.md)
for the full architecture, the capability/authorization model, the
cross-repo implementation plan, and the trade-offs behind it.

## Status

No code yet. The founding PR for this repo is the design document itself —
implementation issues will be filed (here and in agent-mesh / newt-agent /
agent-bridle) once the design settles.

## License

Apache-2.0, matching the rest of the Gilamonster-Foundation stack.
