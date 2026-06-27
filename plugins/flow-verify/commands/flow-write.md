---
description: Interactively author flow files for a feature — propose, confirm, then write.
argument-hint: <feature description> [target-dir]
---

Author flow files for: $ARGUMENTS — interactively, in THIS thread. Do not dispatch a
subagent, and do not open a browser (this step is code-only; the browser opens only in
`/flow-verify`).

1. Read `FORMAT.md` from the flow-verify plugin so you produce conformant files.
2. Read the relevant project source to discover: every scenario, the exact visible
   strings/labels, the routes/URLs, the network endpoints, and the preconditions the
   feature assumes.
3. **Propose before writing.** Present:
   - the scenario set you intend to create (happy path + expected-error cases), one line
     each;
   - the key labels / routes / endpoints you found;
   - explicitly flag any label or precondition you could NOT confirm from the source —
     these are exactly what otherwise causes DRIFT on the first verify.
4. **Ask the user** which scenarios to generate and to confirm or correct the flagged
   assumptions.
5. Write ONLY the confirmed scenarios as separate files to the target dir (default
   `flows/<feature>/`), named `<feature>-<scenario>.md`. All prose in English; literal
   app strings (labels, URLs) quoted verbatim — see the Language rule in FORMAT.md.
6. Report the files written and any assumption still unconfirmed.
