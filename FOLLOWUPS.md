# Follow-ups

Open items deferred during setup. Each is self-contained; pick any.

## 1. Verify the writer + loop end-to-end against a real project
Only the **verifier** has been smoke-tested (`examples/example-pass.md` → PASS on
example.com). The interactive writer (`/browser-flow:write`) and the full
`/browser-flow:loop` have **not** been exercised against real source + a running app.
- Run `/browser-flow:write <feature>` on a real project: confirm it **proposes** the
  scenario set and **asks** before writing (the interactive behavior), not silent writes.
- Then `/browser-flow:loop <feature>` with the app running: confirm it authors, runs each
  flow sequentially, and prints the combined verdict table.
- A DRIFT on the first run is expected (writer never saw the live app).

## 2. Add a "Try the full loop" section to the plugin README
Docs only, ~3 lines in `plugins/browser-flow/README.md`: tell a new user to point
`/browser-flow:write` at a feature in their running app, then `/browser-flow:run` the
generated flow. Not a test fixture — a writer fixture was deliberately rejected (would
need to ship a sample app; the writer is interactive so it can't be a static example).

## 3. Surface the `agent-browser` dependency more clearly
The verifier depends on the external `agent-browser` skill, which is **not bundled**. It's
noted as a prereq in both READMEs, but a fresh installer could still miss it and hit a
confusing failure. Consider a clearer callout, or have the verifier check for the skill
and fail with a pointed message.

## 4. (Conditional) Harden the writer's code-only guarantee
The writer is a main-thread command so it can ask the user — but that traded away the
**tool-enforced** "writer can't open a browser" guarantee (a subagent's restricted
toolset). It's now instruction-only. If in practice the writer guesses labels too much or
ever strays toward the browser, upgrade to a two-phase design: a read-only proposer
**subagent** (Read/Grep/Glob only) returns a scenario proposal → main thread asks the
user → a writer **subagent** (Write only) writes the confirmed files. More machinery;
only do it if a real problem shows up. (YAGNI until then.)

## 5. (Cosmetic) Align the local folder name
The working directory is still `flow-verify/`, while the repo is `cc-plugins` and the
plugin is `browser-flow`. Optional: close the editor and `mv ../flow-verify ../cc-plugins`
(doesn't affect git or the remote).

## 6. (Open question) Marketplace name `cc-plugins`
`cc-plugins` was chosen after `claude-code-plugins` turned out to be reserved by Anthropic.
Revisit only if you want a different marketplace name (avoid `claude*` / `claude-code*` —
reserved). Renaming touches: GitHub repo, `marketplace.json` `name`, and the install
instructions in both READMEs.
