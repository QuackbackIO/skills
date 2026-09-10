---
name: install-widget
description: >-
  Install the Quackback widget in a host application and optionally identify
  signed-in users with a verified token. Use when setting up Quackback, adding
  the feedback or messenger widget, wiring Quackback("init"), or implementing
  Quackback identify / ssoToken / logout.
---

# Install the Quackback widget

Follow these steps IN ORDER. Do not invent APIs. Make the smallest change that works — add alongside existing code, do not restructure the host app.

Credentials come from the user or from Admin → Settings → Widget → Install:

- Instance URL (example: `https://feedback.example.com`)
- Signing secret (optional) — only if the user wants signed-in identify. Server-only. Never commit it or ship it to the browser.

The launcher does not need a signing secret. If the user did not paste one from Admin → Settings → Widget → Install, finish after init. Do not invent a secret and do not search Cloud, self-host, or host env for a Quackback-provided one.

## STEP 1: Detect the stack

Look at dependency and lock files (`package.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lock`, `Gemfile`, `composer.json`, `requirements.txt`, `go.mod`, …) to choose the package manager and where root layout / auth live.

If Quackback is already installed and initialized, do not rewrite it. Skip to STEP 3 only if the user provided a signing secret and wants identify.

## STEP 2: Load the widget

Initialize once, in the root layout / app shell — the same place other third-party scripts load. Not on a single page.

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

**SPA** — the snippet or the npm package both work. Prefer the approach that already exists. If you add the package, use the repo's package manager (`npm install` / `pnpm add` / `bun add` / `yarn add`). Do not hand-edit `package.json`.

```js
import { Quackback } from '@quackback/widget'

Quackback.init({ instanceUrl: process.env.NEXT_PUBLIC_QUACKBACK_URL })
```

The widget must appear for anonymous visitors after `init`. Do not gate the snippet on login.

If the user did not provide a signing secret, stop here after verifying the launcher. Identify is optional.

## STEP 3: Identify signed-in users

Skip this step unless the user wants signed-in attribution **and** provided a signing secret from Admin → Settings → Widget → Install.

Read [references/identify-users.md](references/identify-users.md) now. Then implement it.

Anonymous visitors need no identify call.

1. Add a **server-only** route that reads the host session, signs a short-lived HS256 JWT with the signing secret, and returns `{ ssoToken }`.
2. Store the secret in the host app's server-side secret store. It is not a Quackback setting.
3. Call identify as soon as the host knows who the user is: when the app first loads if they are already signed in, and immediately after login or signup. Once per session — not on every client navigation.
4. If the user is already known at init time, pass `{ ssoToken }` as `identity` on `init` instead of a separate identify call.
5. Call `Quackback("logout")` from the host logout handler. Always, even if you do not expect a shared computer.

Do not call `Quackback("identify", { id, email })`. That unverified shape is rejected.

## STEP 4: Store credentials

If a public instance URL env var already exists (`NEXT_PUBLIC_*` / `VITE_*`), leave it. Only when implementing identify: paste the Admin → Settings → Widget → Install secret into the host app’s server-side secret store. Do not look for a Quackback-provided env var.

Never put the signing secret in public env vars, the snippet, or client bundles.

## STEP 5: Verify

- Widget launcher appears on a logged-out page.
- If identify was implemented: after login (or on an already-authenticated load), the server route returns `{ ssoToken }` and the client identifies once. Logout clears identity; the launcher stays. Secret is not in the client bundle.

## Rules

- Match the host app's auth, routing, and package manager. Reuse existing session helpers.
- Do not rename `ssoToken`.
- If you cannot tell where layout or auth live, ask one question, then continue.
- More detail: https://quackback.io/docs/widget/installation and https://quackback.io/docs/widget/identify-users
