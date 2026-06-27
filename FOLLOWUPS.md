# Follow-ups

Open items deferred during setup. Each is self-contained; pick any.

## 1. Support other browser-control backends
Today the verifier hardcodes the `agent-browser` skill. Make the browser backend
**pluggable** so a flow can run with whatever drives the browser: **Playwright MCP**, a
**CLI**, **Claude-in-Chrome** (the claude-in-chrome MCP), etc. This also removes the hard
`agent-browser` dependency (a fresh installer without it currently hits a confusing
failure).
- Decide how the backend is selected: a per-run flag, plugin config, or auto-detect
  whichever is available.
- Keep the flow format backend-agnostic — flows already reference elements by visible
  text/role, not selectors, so the `.md` files shouldn't need to change.
- The verifier prompt (`agents/verifier.md`) is where the backend is currently wired;
  generalize "Use the `agent-browser` skill" into "use the configured backend".

## 2. (Conditional) Harden the writer's code-only guarantee
The writer is a main-thread command so it can ask the user — but that traded away the
**tool-enforced** "writer can't open a browser" guarantee (a subagent's restricted
toolset). It's now instruction-only. If in practice the writer guesses labels too much or
ever strays toward the browser, upgrade to a two-phase design: a read-only proposer
**subagent** (Read/Grep/Glob only) returns a scenario proposal → main thread asks the
user → a writer **subagent** (Write only) writes the confirmed files. More machinery;
only do it if a real problem shows up. (YAGNI until then.)
