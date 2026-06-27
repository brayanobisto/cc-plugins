# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`flow-verify` is a Claude Code **plugin** — ~7 markdown files, **zero application code**.
There is nothing to build, test, lint, or run. The "runtime" is Claude Code itself
plus the `agent-browser` skill. Changes here are edits to prompts and spec text.

It implements a TDD-like loop for browser verification where the test artifact is a
natural-language markdown **flow file** (no glue code): an agent reads the flow and
executes it against a live browser, using real signals (console, network, DOM, URL,
screenshots) as the evidence for its verdict.

## Architecture

One verifier subagent + one spec, wired through slash commands (the writer is a command, not a subagent):

- `commands/flow-write.md` → **interactive, runs in the main thread** (`/flow-write <feature>`).
  Reads project source, **proposes** a scenario set, **asks** the user to confirm/correct
  (especially labels it couldn't verify from source), then writes one file per confirmed
  scenario into `flows/<feature>/` of the *target* project. Code-only by instruction —
  it must not open a browser. It is NOT a subagent: subagents can't ask the user mid-run,
  and asking is the whole point of this step.
- `commands/flow-verify.md` → dispatches the `flow-verifier` subagent (`/flow-verify <flow.md>`).
- `commands/flow-check.md` → orchestrates the full loop in the main thread (`/flow-check <feature>`):
  does the interactive authoring, then dispatches the verifier once per generated flow,
  sequentially.
- `agents/flow-verifier.md` — **browser-only** subagent, drives `agent-browser` headed,
  returns a verdict and cites real evidence. Never edits code or the flow.
- `FORMAT.md` — single source of truth for the flow format. **Both the writer command and
  the verifier subagent read it at runtime**, so format rules live in exactly one place.
  If you change the format, edit `FORMAT.md`; do not duplicate rules into the commands.

Key invariant: the browser opens **exactly once**, in the verifier. The writer derives
labels/strings from source code and confirms the uncertain ones with the user, but it
still never sees the live app — so the first verify run validates those choices, hence
the third verdict (DRIFT), distinct from a real bug.

### Three verdicts (verifier)

- **PASS** — every assertion holds against a real signal.
- **FAIL** — steps ran but an assertion fails → real bug in the feature.
- **DRIFT** — a step couldn't execute (element not found) or a Context precondition was
  unmet → the flow needs adjusting, not a bug.

Anti-complacency rule: every green assertion **must cite the concrete evidence** (URL
seen, network status, console dump, screenshot). No green on vibes. There is a single
verifier (no executor/judge split) by design.

## Flow files live elsewhere

Flow files are **not** part of this plugin. They live in `flows/` of the project being
verified. This plugin ships only the commands, the verifier subagent, the format spec,
and one `examples/example-pass.md`.

## Conventions when editing

- Flows and all prose are authored in **English**; the only exception is literal app
  strings (on-screen labels, URLs), quoted verbatim. `FORMAT.md` section headers are
  English: `## Context` / `## Steps` / `## Assertions`.
- Assertions must be **observable** (URL / DOM-text / console / network / visual), never
  judgment calls ("looks good").
- Deferred features (CI cache, standalone CLI, executable setup, batch runs, format
  variables) are deliberate non-goals (YAGNI) — don't add them without a real flow that
  needs them.
