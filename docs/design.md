# Flow-Verify — Design Spec

**Date:** 2026-06-27
**Status:** Approved design, pending implementation plan
**Type:** Standalone Claude Code plugin (separate repo from primero-agents)

## Problem

While building a small feature, verifying it in the browser means re-explaining
to the agent — every single time — how to exercise it. There is no durable,
versionable artifact that says "this is how you check this feature works."

## Goal

A TDD-like loop for browser verification, where the test artifact is a
**natural-language markdown file** with no glue code. An agent reads the file
and executes it directly against the browser, using real browser signals
(console, network, DOM, URL, screenshots) as the evidence for its verdict.

The tool is two subagents plus a format spec, shipped as a Claude Code plugin.
It runs inside Claude Code, with `agent-browser` as the execution engine — so
there is **zero application runtime to build**: no agent loop, no model-API
wiring, no browser driver. Just markdown.

## Non-goals (deliberately deferred — YAGNI)

- **No deterministic compile/cache** of NL → actions. That solves CI
  repeatability; this is a personal dev loop, not CI.
- **No standalone CLI** with its own agent loop. Claude Code + agent-browser
  already provide the loop and the browser.
- **No executable setup/teardown.** Preconditions are described, not run.
- **No batch verification** (whole folder). One scenario per run.
- **No variables, loops, or per-assertion type tags** in the format.
- **No separate executor/judge split** in the verifier (see "Verifier").

Each is added only when a real flow needs it.

## 1. Flow format

A flow file is a markdown document with frontmatter and up to three sections.
**One file = one scenario = one expected outcome.** An error case is not a
flow that fails — it is a flow whose *expected outcome is the error*; its
assertions affirm the error appears. Error cases use the same format, as
separate files.

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

### Rules

1. **Frontmatter:** `name` and `url` (entry point) required. Nothing else.
2. **Pasos:** ordered list, one imperative action per line. Reference elements
   by **visible text / label / role** ("the Iniciar sesión button"), never by
   CSS selector — having no selectors is the point.
3. **Aserciones:** bullet list, each **observable** — it must map to a concrete
   signal agent-browser can read without judgment:
   - **URL** — "changes to /dashboard"
   - **DOM/text** — "the header shows X"
   - **console** — "no errors"
   - **network** — "POST /api/login → 200"
   - **visual** — "the welcome modal is visible" (last resort, screenshot)

   Vague judgment assertions ("looks good", "works correctly") are not allowed;
   the writer rewrites them to something observable or drops them.
4. **Contexto:** free prose, optional, **not executable**. The writer→verifier
   bridge: code-derived preconditions the flow assumes, so the verifier can
   tell a real bug from a missing precondition.

`FORMAT.md` in the plugin is the single source of truth for these rules; both
subagents read it at runtime.

## 2. Writer subagent (`flow-writer`)

Autonomous. From a short description (`/flow-write login`), produces one or
more flow files matching the format.

- **Code-only. Never opens the browser.** (So the browser opens exactly once,
  in the verifier.)
- Reads the source to discover: all scenarios, exact strings/labels, routes,
  and preconditions.
- Generates the **scenario set**, not just the happy path: the valid case plus
  the obvious expected-error cases (wrong password, empty fields, …), each as a
  **separate file**. The user curates by deleting unwanted files.
- Writes discovered **preconditions into `## Contexto`**.
- Enforces the observable-assertions rule: rewrites or drops vague assertions.
- Reads `FORMAT.md` so it doesn't reinvent the structure.

Autonomous (not interactive) because a dispatched subagent cannot ask the user
mid-run. Curating extra files afterward is trivial.

## 3. Verifier subagent (`flow-verifier`)

A true subagent: runs alone, browser-heavy, returns a verdict.

- **Input:** path to one flow file (`/flow-verify flows/login-valido.md`).
- Parses the flow; **reads `## Contexto` first** to know the assumed
  preconditions.
- Opens agent-browser at `url` in **headed** mode (dev loop — you want to watch).
- Executes the `Pasos` in order, referencing visible text/labels.
- Captures evidence throughout: console, network, DOM/text, URL, screenshots.
- Evaluates each `Aserción` against the concrete signal.

### Verdict — three outcomes, not just green/red

| Outcome | Meaning |
|---|---|
| ✅ **PASS** | All assertions hold against real signals. |
| ❌ **FAIL (bug)** | Steps ran but an assertion fails → real bug in the feature. |
| ⚠️ **BLOCKED / DRIFT** | A step couldn't execute (element not found) → the app changed, a `Contexto` precondition is missing, or the writer guessed a label wrong (it never saw the live app). Not a bug — the flow needs adjusting. |

The DRIFT outcome is the natural consequence of a code-only writer: the first
verify run also validates the writer's guesses.

### Anti-complacency: cite the evidence

A single verifier (no executor/judge split). The discipline that fights the
"complacent judge" is: **every green assertion must cite the concrete evidence**
that backs it (the URL seen, the network status, the console dump, the
screenshot). "Show your work" — if it must quote the real signal, it cannot
declare green on vibes. This buys ~90% of the impartiality of a two-agent split
without the extra dispatch and evidence-passing overhead.

A separate executor/judge split is added only if false greens show up in
practice.

## 4. Packaging

A standalone repo, structured as a Claude Code plugin (~6 markdown files, zero
application code):

```
flow-verify/
  .claude-plugin/
    plugin.json          plugin manifest
  commands/
    flow-write.md        /flow-write  → dispatches the flow-writer subagent
    flow-verify.md       /flow-verify → dispatches the flow-verifier subagent
  agents/
    flow-writer.md       authoring subagent (code-only)
    flow-verifier.md     verification subagent (browser)
  FORMAT.md              the format spec — single source of truth
  README.md
```

- **`commands/`** are the entry points you type; they are thin — each dispatches
  its subagent, passing the argument.
- **`agents/`** hold the two subagent system prompts.
- **`FORMAT.md`** is read at runtime by both subagents, so the format rules live
  in one place.
- **Flow files do not live in the plugin.** They live in `flows/` of the project
  being verified (primero-agents, or any project). The plugin provides commands,
  subagents, and the format; the flows are the user's, in their repo.
- Built as a plugin from day one (costs only a `plugin.json`). Installed locally
  to use now; "share later" = publish the repo, nothing to rewrite.
