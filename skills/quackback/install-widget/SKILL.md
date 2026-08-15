---
name: install-widget
description: >-
  Install the Quackback widget in a host application and identify signed-in
  users with a verified token. Use when setting up Quackback, adding the
  feedback or messenger widget, wiring Quackback("init"), or implementing
  Quackback identify / ssoToken / logout.
---

# Install the Quackback widget

Follow these steps IN ORDER. Do not invent APIs.

Credentials come from the user or from Admin → Settings → Widget → Install:

- Instance URL (example: `https://feedback.example.com`)
- `QUACKBACK_WIDGET_SECRET` — server-only. Never commit it or ship it to the browser.

If either value is missing, ask once, then continue.

## STEP 1: Detect the stack

Look at dependency and lock files (`package.json`, `pnpm-lock.yaml`, `bun.lock`, `Gemfile`, `composer.json`, `requirements.txt`, `go.mod`, …) to choose the package manager and where layout / auth live.

If Quackback is already installed and initialized, do not rewrite it. Add only what is missing (usually identify).

## STEP 2: Load the widget

Pick the path that matches the repo.

**HTML / any site** — paste before `</body>`:

```html
<script>
  (function(w,d){if(w.Quackback)return;w.Quackback=function(){
  (w.Quackback.q=w.Quackback.q||[]).push(arguments)};
  var s=d.createElement("script");s.async=true;
  s.src="INSTANCE_URL/api/widget/sdk.js";
  d.head.appendChild(s)})(window,document);
  Quackback("init");
</script>
```

Replace `INSTANCE_URL` with the workspace URL, no trailing slash.

**SPA (React, Next, Vue, Svelte, …)** — the snippet or `npm install @quackback/widget` both work. Prefer the approach that already exists in the repo.

```js
import { Quackback } from '@quackback/widget'

Quackback.init({ instanceUrl: process.env.NEXT_PUBLIC_QUACKBACK_URL })
```

The widget must appear for anonymous visitors after `init`. Do not gate the snippet on login.

## STEP 3: Identify signed-in users

Read [references/identify-users.md](references/identify-users.md) now. Then implement it.

Identify is required for signed-in users. Anonymous visitors need no identify call.

1. Add a **server-only** route that reads the host session, signs a short-lived HS256 JWT with `QUACKBACK_WIDGET_SECRET`, and returns `{ ssoToken }`.
2. Call identify as soon as the host knows who the user is: after login, after signup, and on every authenticated page load.
3. Call `Quackback("logout")` from the host logout handler.

Do not call `Quackback("identify", { id, email })`. That unverified shape is rejected.

## STEP 4: Store the secret

Put `QUACKBACK_WIDGET_SECRET` in `.env` / `.env.local` or the host secret store. Reference it only from server code. Never put it in `NEXT_PUBLIC_*`, `VITE_*`, or the snippet.

If the instance URL is needed on the client, a public env var for the URL alone is fine.

## STEP 5: Verify

- Widget launcher appears on a logged-out page.
- After login, the server route returns `{ ssoToken }` and the client calls `Quackback("identify", { ssoToken })`.
- Logout clears identity; the launcher stays.
- Secret is not in the client bundle.

## Rules

- Match the host app's auth, routing, and package manager. Reuse existing session helpers.
- Do not rename `ssoToken`.
- If you cannot tell where layout or auth live, ask one question, then continue.
- More detail: https://quackback.io/docs/widget/installation and https://quackback.io/docs/widget/identify-users
