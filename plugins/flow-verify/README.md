# flow-verify

A TDD-like loop for browser verification. The test artifact is a natural-language
markdown flow file — no glue code. Three commands:

- `/flow-write <feature>` — reads your project source, proposes a scenario set (happy
  path + expected-error cases) and asks you to confirm before writing one flow file per
  scenario.
- `/flow-verify <flow-file>` — runs one flow against the browser via `agent-browser`
  and returns a verdict (PASS / FAIL / DRIFT) that cites real signals.
- `/flow-check <feature>` — the full loop: writes the flows, then verifies each one
  and reports a combined verdict. Skips the manual curation pause.

Flow files live in `flows/` of the project you are verifying, not in this plugin.
See `FORMAT.md` for the flow format.

## Install

Part of the `brayan-plugins` marketplace:

```
/plugin marketplace add brayanobisto/brayan-plugins
/plugin install flow-verify@brayan-plugins
```

Requires the `agent-browser` skill (used by the verifier; not bundled).
