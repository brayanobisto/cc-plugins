---
description: Write flows for a feature, then verify each against the browser.
argument-hint: <feature description> [target-dir]
---

Run the full loop for: $ARGUMENTS

1. Author the flow files interactively, exactly as `/browser-flow:write` does (read the source,
   propose the scenario set, ask the user to confirm/correct, then write). Collect the
   list of files written.
2. Then, for EACH file it wrote, dispatch the `verifier` subagent **one at a
   time** (one scenario per run) and collect its verdict.
3. Present a combined summary: one row per flow file with its outcome
   (PASS / FAIL / DRIFT) and the key cited evidence.

Notes:
- The app must already be running on the flow's dev URL — this command does not
  start it.
- A DRIFT on the first run is expected and informative: the writer never saw the
  live app, so a guessed label may need fixing. Report it as "flow needs adjusting",
  not a failure.
- This combined command skips the manual curation pause. To curate, delete unwanted
  flow files and re-run individual `/browser-flow:run` instead.
