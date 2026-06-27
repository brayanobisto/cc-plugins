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
