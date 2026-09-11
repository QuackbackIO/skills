# Identify users

The widget works anonymously after `init`. Identify links that anonymous session to a real customer as soon as the host app knows who they are.

Anonymous activity from before login moves onto the identified user automatically. Do not wait for the user to open the widget. Do not skip identify because the launcher already appears.

Only implement this file when the host already has login, a session, or a current-user helper. If it does not, leave the signing secret in server-only env and stop. Do not invent auth.

## Signing secret

The signing secret lives in the **host app** server environment (any name). It is not a Quackback Cloud or Quackback-host env var.

Get it by:

1. Redeeming a pairing code from the copied install prompt:

   `POST {instanceUrl}/api/widget/install-context` with `{ "code": "<pairing code>" }`.

   Write `signingSecret` from the JSON response to a server-only host env var. Never print it.

2. Or copying it from Admin → Settings → Widget → Install (Reveal / Copy).

Never invent a secret. Never put it in client code, public env vars, commits, or logs. Do not search Cloud, self-host, or host env for a Quackback-provided one — Quackback never injects one.

## When to identify

Call identify as soon as you are able. Typically:

1. When the app first loads, if the user is already signed in.
2. Immediately after login or signup.

Once per session. Do not identify again on every client-side navigation.

The JWT is short-lived (~5 minutes) so the *identify call* cannot be replayed later. Mint a fresh token at the moment you identify. After a successful identify, the widget keeps its own session — you do not need to re-identify until the next full load or a new login.

If the user is already known when you call `init`, pass the token there and skip a separate identify:

```js
Quackback('init', { identity: { ssoToken } })
```

## What to send

Identify is verified-only. The browser must not pass raw `id` or `email`.

Your backend signs an HS256 JWT with the host-app signing secret and returns `{ ssoToken }`. Store that secret in the host app server-side secret store (example: `WIDGET_SIGNING_SECRET`). It is not a Quackback Cloud or self-host environment variable.

| Claim | Required | Role |
| --- | --- | --- |
| `sub` | yes | Stable host user id (database id). Unique string. Not email. |
| `email` | yes | Person property for notifications and dedup. |
| `name` | no | Display name. Pass it when you have it. |
| `exp` | yes | ~5 minutes from now. |

`sub` is the durable id. Email can change; `sub` must not. Never use `null`, `undefined`, `true`, `"anonymous"`, or a shared placeholder as `sub` — two users with the same `sub` are merged.

Each time you identify, include every person claim you have (`email`, `name`, and any configured custom attributes). That keeps the profile current.

## Server route

Reuse the host session. Return 401 when nobody is signed in — the client then stays anonymous.

```ts
import { SignJWT } from 'jose'

// Host app server env, any name — not a Quackback-host variable.
const secret = new TextEncoder().encode(process.env.WIDGET_SIGNING_SECRET)

export async function GET(request: Request) {
  const user = await getCurrentUser(request) // host session helper
  if (!user?.id || !user.email) return new Response('Unauthorized', { status: 401 })

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

When the app loads and a session exists, or right after login/signup:

```js
const res = await fetch('/api/widget-sso')
if (res.ok) {
  const { ssoToken } = await res.json()
  Quackback('identify', { ssoToken })
}
```

Script-tag hosts use `Quackback("identify", { ssoToken })`. npm hosts use `Quackback.identify({ ssoToken })`.

## Reset on logout

Always call logout on the host logout path, even if you do not expect a shared computer. Otherwise the next person on that browser keeps the previous identity.

```js
Quackback('logout')
```

The launcher stays. A later identify replaces the previous identity; still call logout so a shared browser does not keep the last user.

## Do not

- Put the signing secret in client code or a Quackback-host env file.
- Call `Quackback("identify", { id, email })`.
- Use email, `null`, or a generic string as `sub`.
- Identify on every route change.
- Identify only at signup and never again on later visits — call it on each authenticated app load.
- Invent a second identity API.
- Invent a signing secret or pairing code, or search Cloud / self-host / host env for a Quackback-provided signing secret.
- Invent auth or a placeholder user so you can "finish" identify on a logged-out-only site.
