# Flow format

A flow file is a markdown document describing **one scenario with one expected
outcome**. An error case is not a flow that fails — it is a flow whose expected
outcome *is* the error; its assertions affirm the error appears. Error cases use
this same format, as separate files.

## Structure

```markdown
---
name: Login with valid credentials
url: http://localhost:3000/login
---

## Context             (optional)
State the flow assumes before it starts. Filled by the writer from the code.
E.g.: "A user test@test.com with password secret123 already exists."

## Steps               (required)
1. Type "test@test.com" into the email field.
2. Type "secret123" into the password field.
3. Click the "Iniciar sesión" button.

## Assertions          (required)
- The URL changes to /dashboard.
- The header shows "test@test.com".
- There are no console errors.
- The POST to /api/login returns 200.
```

## Rules

1. **Frontmatter:** `name` and `url` (entry point) are required. Nothing else.
2. **Steps:** ordered list, one imperative action per line. Reference elements by
   **visible text / label / role** ("the Iniciar sesión button"), never by CSS
   selector.
3. **Assertions:** bullet list, each **observable** — it must map to a concrete
   signal the browser can read without judgment:
   - **URL** — "changes to /dashboard"
   - **DOM/text** — "the header shows X"
   - **console** — "no errors"
   - **network** — "POST /api/login → 200"
   - **visual** — "the welcome modal is visible" (last resort, screenshot)

   Vague judgment assertions ("looks good", "works correctly") are not allowed.
4. **Context:** free prose, optional, **not executable**. The writer→verifier
   bridge: code-derived preconditions the flow assumes, so the verifier can tell a
   real bug from a missing precondition.
5. **Language:** always author flows in **English** — folder name, file name, and
   all prose (`name`, Context, Steps, Assertions) — regardless of the conversation
   language. The only exception is **literal app strings** (on-screen labels,
   URLs): quote them verbatim as they appear in the app, even if that text is in
   another language.

## One file = one scenario

Name files `<feature>-<scenario>.md`, optionally grouped in `flows/<feature>/`.
Example set for login:

- `login-valid.md` — asserts redirect to /dashboard
- `login-wrong-password.md` — asserts the error message, still on /login
- `login-blocked-account.md` — asserts the blocked message, POST → 403
