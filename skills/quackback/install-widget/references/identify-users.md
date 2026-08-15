# Identify users

The widget works anonymously after `init`. Identify links that anonymous session to a real customer as soon as the host app knows who they are.

## When to identify

Call identify:

1. On every authenticated page load (the token is short-lived on purpose).
2. Immediately after login and signup.

Do not wait for the user to open the widget. Do not skip identify because the launcher already appears.

Call `Quackback("logout")` on logout. The launcher stays; identity clears. A later identify replaces the previous one — still call logout on the host logout path so a shared browser does not keep the last user.

## What to send

Identify is verified-only. The browser must not pass raw `id` or `email`.

Your backend signs an HS256 JWT with `QUACKBACK_WIDGET_SECRET` and returns `{ ssoToken }`.

| Claim | Required | Role |
| --- | --- | --- |
| `sub` | yes | Stable host user id (database id). Not email. |
| `email` | yes | Person property for notifications and dedup. |
| `name` | no | Display name. |
| `exp` | yes | ~5 minutes from now. |

`sub` is the durable id. Email can change; `sub` must not.

## Server route

Reuse the host session. Return 401 when nobody is signed in — the client then skips identify.

```ts
import { SignJWT } from 'jose'

const secret = new TextEncoder().encode(process.env.QUACKBACK_WIDGET_SECRET)

export async function GET(request: Request) {
  const user = await getCurrentUser(request) // host session helper
  if (!user) return new Response('Unauthorized', { status: 401 })

  const ssoToken = await new SignJWT({
    sub: String(user.id),
    email: user.email,
    name: user.name,
  })
    .setProtectedHeader({ alg: 'HS256' })
    .setExpirationTime('5m')
    .sign(secret)

  return Response.json({ ssoToken })
}
```

JSON key is `ssoToken`, not `token`.

## Client

After auth resolves, and again on each authenticated load:

```js
const res = await fetch('/api/widget-sso', { method: 'POST' })
if (res.ok) {
  const { ssoToken } = await res.json()
  Quackback('identify', { ssoToken })
}
```

Script-tag hosts use `Quackback("identify", { ssoToken })`. npm hosts use `Quackback.identify({ ssoToken })`.

```js
// logout handler
Quackback('logout')
```

## Do not

- Put `QUACKBACK_WIDGET_SECRET` in client code.
- Call `Quackback("identify", { id, email })`.
- Use email as `sub`.
- Identify only once at signup and never again — tokens expire in ~5 minutes.
- Invent a second identity API.
