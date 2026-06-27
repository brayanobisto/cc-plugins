# flow-verify

A TDD-like loop for browser verification. The test artifact is a natural-language
markdown flow file — no glue code. Two subagents:

- `/flow-write <feature>` — reads your project source and writes flow files (one
  scenario per file: happy path + expected-error cases).
- `/flow-verify <flow-file>` — runs one flow against the browser via `agent-browser`
  and returns a verdict (PASS / FAIL / DRIFT) that cites real signals.

Flow files live in `flows/` of the project you are verifying, not in this plugin.
See `FORMAT.md` for the flow format.

## Install (local)

Add this directory as a local plugin in Claude Code, then restart. Verify with
`/help` — `flow-write` and `flow-verify` should appear.
