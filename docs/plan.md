# Flow-Verify Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a Claude Code plugin (`flow-verify`) of two subagents plus a format spec that author and run natural-language browser-verification flows, with zero application runtime.

**Architecture:** Everything is markdown. A `flow-writer` subagent reads project source and writes flow `.md` files; a `flow-verifier` subagent runs one flow against the browser via `agent-browser` and returns a verdict citing real signals. Two thin slash commands dispatch them. `FORMAT.md` is the single source of truth both subagents read at runtime.

**Tech Stack:** Claude Code plugin (`.claude-plugin/plugin.json`, `commands/`, `agents/`), `agent-browser` skill as the execution engine. No code, no test framework, no dependencies.

## Global Constraints

- Plugin lives in its own repo at `/Users/brayanobisto/Development/flow-verify/`, separate from primero-agents.
- All file content (prompts, docs, identifiers) is written in **English**; example flow content mirrors the target app and stays in **Spanish**.
- The writer is **code-only** — it must never invoke `agent-browser` or open a browser. The browser opens exactly once, in the verifier.
- Every green assertion in a verdict must **cite the concrete evidence** (URL, network status, console output, screenshot) backing it.
- `FORMAT.md` is the only place format rules are defined; both subagents read it at runtime instead of restating the rules.
- Verification is **manual** (run the command, observe) — there is no automated test framework for prompt files.

---

### Task 1: Plugin scaffold

**Files:**
- Create: `/Users/brayanobisto/Development/flow-verify/.claude-plugin/plugin.json`
- Create: `/Users/brayanobisto/Development/flow-verify/README.md`
- Create: `/Users/brayanobisto/Development/flow-verify/.gitignore`

**Interfaces:**
- Produces: a loadable Claude Code plugin named `flow-verify`. Later tasks add `commands/` and `agents/` files, which Claude Code auto-discovers.

- [ ] **Step 1: Init the repo**

```bash
mkdir -p /Users/brayanobisto/Development/flow-verify/.claude-plugin
cd /Users/brayanobisto/Development/flow-verify && git init
```

- [ ] **Step 2: Write `plugin.json`**

```json
{
  "name": "flow-verify",
  "version": "0.1.0",
  "description": "Author and run natural-language browser-verification flows. Two subagents plus a format spec.",
  "author": { "name": "Brayan Obispo Torres" }
}
```

- [ ] **Step 3: Write `.gitignore`**

```
.DS_Store
```

- [ ] **Step 4: Relocate the design docs into this repo**

These were authored in primero-agents' working tree; they belong here. Move both:

```bash
mkdir -p /Users/brayanobisto/Development/flow-verify/docs
mv /Users/brayanobisto/Development/primero-agents/docs/superpowers/specs/2026-06-27-flow-verify-design.md /Users/brayanobisto/Development/flow-verify/docs/design.md
mv /Users/brayanobisto/Development/primero-agents/docs/superpowers/plans/2026-06-27-flow-verify.md /Users/brayanobisto/Development/flow-verify/docs/plan.md
```

Continue executing from `flow-verify/docs/plan.md` after the move.

- [ ] **Step 5: Write `README.md`**

```markdown
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
```

- [ ] **Step 6: Verify the plugin loads**

Add the local plugin to Claude Code (e.g. `/plugin marketplace add /Users/brayanobisto/Development/flow-verify` then install, or your local-plugin config) and restart.
Expected: no load errors. `flow-write`/`flow-verify` won't appear yet (added in Tasks 3–4); confirm the plugin itself is recognized with no manifest error.

- [ ] **Step 7: Commit (includes the relocated design docs)**

```bash
cd /Users/brayanobisto/Development/flow-verify
git add -A
git commit -m "chore: scaffold flow-verify plugin and import design docs"
```

---

### Task 2: FORMAT.md — the flow format spec

**Files:**
- Create: `/Users/brayanobisto/Development/flow-verify/FORMAT.md`

**Interfaces:**
- Produces: `FORMAT.md`, the single source of truth read at runtime by both the writer (Task 4) and verifier (Task 3). Defines frontmatter (`name`, `url`), and the `## Contexto` / `## Pasos` / `## Aserciones` sections.

- [ ] **Step 1: Write `FORMAT.md`**

````markdown
# Flow format

A flow file is a markdown document describing **one scenario with one expected
outcome**. An error case is not a flow that fails — it is a flow whose expected
outcome *is* the error; its assertions affirm the error appears. Error cases use
this same format, as separate files.

## Structure

```markdown
---
name: Login con credenciales válidas
url: http://localhost:3000/login
---

## Contexto            (optional)
State the flow assumes before it starts. Filled by the writer from the code.
Ej: "El usuario test@test.com con password secret123 ya existe."

## Pasos               (required)
1. Escribe "test@test.com" en el campo de email.
2. Escribe "secret123" en el campo de contraseña.
3. Haz click en "Iniciar sesión".

## Aserciones          (required)
- La URL cambia a /dashboard.
- El header muestra "test@test.com".
- No hay errores en la consola.
- El POST a /api/login devuelve 200.
```

## Rules

1. **Frontmatter:** `name` and `url` (entry point) are required. Nothing else.
2. **Pasos:** ordered list, one imperative action per line. Reference elements by
   **visible text / label / role** ("the Iniciar sesión button"), never by CSS
   selector.
3. **Aserciones:** bullet list, each **observable** — it must map to a concrete
   signal the browser can read without judgment:
   - **URL** — "changes to /dashboard"
   - **DOM/text** — "the header shows X"
   - **console** — "no errors"
   - **network** — "POST /api/login → 200"
   - **visual** — "the welcome modal is visible" (last resort, screenshot)

   Vague judgment assertions ("looks good", "works correctly") are not allowed.
4. **Contexto:** free prose, optional, **not executable**. The writer→verifier
   bridge: code-derived preconditions the flow assumes, so the verifier can tell a
   real bug from a missing precondition.

## One file = one scenario

Name files `<feature>-<scenario>.md`, optionally grouped in `flows/<feature>/`.
Example set for login:

- `login-valido.md` — asserts redirect to /dashboard
- `login-password-incorrecto.md` — asserts the error message, still on /login
- `login-cuenta-bloqueada.md` — asserts the blocked message, POST → 403
````

- [ ] **Step 2: Verify it is self-consistent**

Read `FORMAT.md` back. Expected: the example matches the rules (has `name`+`url`, Pasos reference visible text, every Aserción maps to URL/DOM/console/network/visual). Fix any mismatch.

- [ ] **Step 3: Commit**

```bash
cd /Users/brayanobisto/Development/flow-verify
git add FORMAT.md
git commit -m "docs: add flow format spec"
```

---

### Task 3: Verifier subagent + command

**Files:**
- Create: `/Users/brayanobisto/Development/flow-verify/agents/flow-verifier.md`
- Create: `/Users/brayanobisto/Development/flow-verify/commands/flow-verify.md`
- Create (test fixture): `/Users/brayanobisto/Development/flow-verify/examples/example-pass.md`

**Interfaces:**
- Consumes: `FORMAT.md` (Task 2) — read at runtime to parse a flow.
- Produces: subagent type `flow-verifier`, dispatched by `/flow-verify <flow-file>`. Returns a verdict with outcome ∈ {PASS, FAIL, DRIFT}, per-assertion result, and cited evidence.

- [ ] **Step 1: Write `agents/flow-verifier.md`**

```markdown
---
name: flow-verifier
description: Runs one natural-language flow file against the browser and returns a verdict (PASS / FAIL / DRIFT) citing real signals. Use when verifying a flow.
tools: Read, Bash, Skill
---

You verify ONE flow file against a live browser and return a verdict. You do not
fix code and you do not edit the flow.

## Steps

1. Read `FORMAT.md` from the flow-verify plugin to know the flow structure, then
   read the flow file you were given. Parse: `url`, `## Contexto`, `## Pasos`,
   `## Aserciones`.
2. Read `## Contexto` first. It lists the preconditions the flow assumes. If a
   precondition is clearly unmet, the verdict is DRIFT (not a bug).
3. Use the `agent-browser` skill to drive the browser in **headed** mode. Open the
   flow's `url`.
4. Execute the `## Pasos` in order, referencing elements by their visible
   text/label as written. If a step cannot be executed (element not found), stop
   and return **DRIFT** — record which step and why.
5. As you go, capture evidence: console messages, network requests, page
   text/DOM, the current URL, and screenshots at key moments.
6. Evaluate each `## Aserción` against the concrete captured signal.
7. Close the browser when done.

## Verdict

Return exactly one outcome:

- **PASS** — every assertion holds against a real signal.
- **FAIL** — steps ran but at least one assertion fails. This is a real bug.
- **DRIFT** — a step could not execute, or a Contexto precondition was unmet. Not
  a bug; the flow needs adjusting (the app changed, or a label was guessed wrong).

For EVERY assertion, report ✅/❌ AND cite the concrete evidence that backs it —
the URL you saw, the network status, the console output, or a screenshot path.
Never declare an assertion green without quoting its evidence.

Output format:

    Outcome: PASS | FAIL | DRIFT
    Assertions:
      - ✅ <assertion> — evidence: <signal you observed>
      - ❌ <assertion> — evidence: <signal you observed>
    Notes: <for DRIFT: which step/precondition; for FAIL: what broke>
```

- [ ] **Step 2: Write `commands/flow-verify.md`**

```markdown
---
description: Run a flow file against the browser and report a verdict.
argument-hint: <path-to-flow.md>
---

Dispatch the `flow-verifier` subagent to verify this flow file: $ARGUMENTS

Relay its verdict verbatim.
```

- [ ] **Step 3: Write the test fixture `examples/example-pass.md`**

Pick a page that is trivially true to assert against (no app setup needed). Use `https://example.com`:

```markdown
---
name: Example page loads
url: https://example.com
---

## Pasos
1. Espera a que la página cargue.

## Aserciones
- El texto "Example Domain" es visible en la página.
- No hay errores en la consola.
```

- [ ] **Step 4: Reload the plugin and run the verifier**

Restart Claude Code so `flow-verify` appears. Then run:

```
/flow-verify /Users/brayanobisto/Development/flow-verify/examples/example-pass.md
```

Expected: outcome **PASS**, both assertions ✅, each citing evidence (the "Example Domain" text it read, and an empty/clean console). A browser window opens (headed) and closes.

- [ ] **Step 5: Sanity-check the FAIL/DRIFT paths**

Temporarily edit the fixture's assertion to `- El texto "Nonexistent" es visible.` and re-run.
Expected: outcome **FAIL**, that assertion ❌ with evidence that the text was not found. Revert the edit afterward.

- [ ] **Step 6: Commit**

```bash
cd /Users/brayanobisto/Development/flow-verify
git add agents/flow-verifier.md commands/flow-verify.md examples/example-pass.md
git commit -m "feat: add flow-verifier subagent and /flow-verify command"
```

---

### Task 4: Writer subagent + command

**Files:**
- Create: `/Users/brayanobisto/Development/flow-verify/agents/flow-writer.md`
- Create: `/Users/brayanobisto/Development/flow-verify/commands/flow-write.md`

**Interfaces:**
- Consumes: `FORMAT.md` (Task 2) — read at runtime to produce conformant flow files.
- Produces: subagent type `flow-writer`, dispatched by `/flow-write <feature> [target-dir]`. Writes one or more flow `.md` files (verified by the Task 3 verifier).

- [ ] **Step 1: Write `agents/flow-writer.md`**

```markdown
---
name: flow-writer
description: Reads project source and writes natural-language flow files (one scenario per file). Code-only — never opens a browser. Use when authoring flows for a feature.
tools: Read, Grep, Glob, Write
---

You author flow files for a feature by reading the project SOURCE CODE. You never
open a browser and never run agent-browser.

## Steps

1. Read `FORMAT.md` from the flow-verify plugin so you produce conformant files.
2. From the feature description you were given, read the relevant source to
   discover: every scenario, the exact visible strings/labels, the routes/URLs,
   the network endpoints, and the preconditions the feature assumes.
3. Generate the SCENARIO SET — not just the happy path. Include the valid case
   plus the obvious expected-error cases (wrong input, empty fields, unauthorized,
   ...). Write EACH scenario as a SEPARATE file.
4. Put the preconditions you discovered into each file's `## Contexto`.
5. Enforce the observable-assertions rule from FORMAT.md: every assertion must map
   to URL / DOM-text / console / network / visual. Rewrite or drop vague ones.
6. Write files to the target directory (default `flows/<feature>/` in the current
   project) named `<feature>-<scenario>.md`.

## Output

Return a short list of the files you wrote and the scenario each covers, so the
user can curate (delete unwanted ones). Note any assumption you could not confirm
from the source.
```

- [ ] **Step 2: Write `commands/flow-write.md`**

```markdown
---
description: Read project source and write flow files for a feature.
argument-hint: <feature description> [target-dir]
---

Dispatch the `flow-writer` subagent to author flow files for: $ARGUMENTS

Relay the summary of files it wrote.
```

- [ ] **Step 3: Reload the plugin and run the writer against a real feature**

Restart Claude Code. From inside the primero-agents project, run a writer pass against a known feature, e.g.:

```
/flow-write login
```

Expected: it reads source (no browser), and writes a scenario set under `flows/login/` — at least a happy-path file plus one expected-error file, each conforming to FORMAT.md (frontmatter, Pasos referencing visible Spanish labels, observable Aserciones, Contexto with preconditions).

- [ ] **Step 4: Close the loop — verify a generated flow**

Run the verifier on one of the files the writer just produced (requires the primero-agents app running on its dev URL):

```
/flow-verify flows/login/login-valido.md
```

Expected: a verdict. A DRIFT here is acceptable and informative (the writer never saw the live app, so a guessed label may need fixing) — confirm the verdict explains itself with cited evidence.

- [ ] **Step 5: Commit**

```bash
cd /Users/brayanobisto/Development/flow-verify
git add agents/flow-writer.md commands/flow-write.md
git commit -m "feat: add flow-writer subagent and /flow-write command"
```

---

## Notes for the implementer

- Local-plugin install mechanics vary by Claude Code version. The plan assumes you
  can register `/Users/brayanobisto/Development/flow-verify/` as a local plugin and
  restart; if the exact command differs, adapt and confirm the commands appear in
  `/help`.
- The writer's `flows/` output lands in whatever project you run it from, not in
  the plugin repo. That is intended.
