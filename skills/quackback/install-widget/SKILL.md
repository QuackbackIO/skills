---
name: install-widget
description: >-
  Install the Quackback widget in a host application. Redeem a pairing code for
  the host signing secret, init the launcher for every visitor, and identify
  signed-in users with a backend-signed ssoToken when the host already has
  login. Use when setting up Quackback, adding the feedback or messenger
  widget, wiring Quackback("init"), redeeming a pairing code, or implementing
  identify / ssoToken / logout.
---

# Install the Quackback widget

Follow these steps IN ORDER. Do not invent APIs. Make the smallest change that works — add alongside existing code, do not restructure the host app.

## Credentials

The instance URL (example: `https://feedback.example.com`) is enough to show the launcher. Do not ask for a Quackback-host env var. Quackback Cloud and self-host do not define `QUACKBACK_WIDGET_SECRET`.

The install prompt includes a **pairing code**. Redeem it. That writes a **signing secret into the host app server env** (any name) so this install is capable of identify. Never invent a secret. Never ask the user whether they want identify.

## STEP 1: Detect the stack

Look at dependency and lock files (`package.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lock`, `Gemfile`, `composer.json`, `requirements.txt`, `go.mod`, …) to choose the package manager and where root layout / auth live.

If Quackback is already installed and initialized, do not rewrite it. Add only what is missing.

## STEP 2: Redeem the pairing code

If the prompt includes a pairing code, redeem it now (do not print the response secret):

`POST {instanceUrl}/api/widget/install-context` with JSON `{ "code": "<pairing code>" }`.

Success returns `{ instanceUrl, sdkUrl, signingSecret }`. Write `signingSecret` to a **server-only** host env var (any name). Never commit it, log it, or put it in public env (`NEXT_PUBLIC_*` / `VITE_*`). Redeeming also turns on Show on your website.

If redeem fails, ask the user to copy the install prompt again from Admin → Settings → Widget → Install. Do not invent a secret or code.

If there is no pairing code and no signing secret yet, ask the user to copy the signing secret from Admin → Settings → Widget → Install — never invent one.

## STEP 3: Load the widget

Initialize once, in the root layout / app shell — the same place other third-party scripts load. Not on a single page.

If valid values already exist in `.env` / `.env.local`, leave them. Otherwise, when the client needs the instance URL, write a public env var for the URL only (`NEXT_PUBLIC_*` / `VITE_*`). The signing secret is already in server-only env from STEP 2.

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

## STEP 4: Identify signed-in users — only if this app already has them

Look at the host repo. If there is login, a session, or a current-user helper, read [references/identify-users.md](references/identify-users.md) and implement it now.

If this is a static or logged-out-only site, **stop**. Leave the signing secret in server-only env. Do not invent auth, a placeholder `sub`, or a stub identify route that always 401s.

Anonymous visitors need no identify call.

When you do implement identify:

1. Add a **server-only** route that reads the host session, signs a short-lived HS256 JWT with that signing secret, and returns `{ ssoToken }`.
2. Call identify as soon as the host knows who the user is: when the app first loads if they are already signed in, and immediately after login or signup. Once per session — not on every client navigation.
3. If the user is already known at init time, pass `{ ssoToken }` as `identity` on `init` instead of a separate identify call.
4. Call logout from the host logout handler. Always, even if you do not expect a shared computer. Script tag: `Quackback("logout")`. npm: `Quackback.logout()`.

Do not call `Quackback("identify", { id, email })`. That unverified shape is rejected.

## STEP 5: Verify

- Open a page with the snippet. The Install page in Quackback should flip to connected.
- Widget launcher appears on a logged-out page (Show on your website must be on).
- If you implemented identify: after login (or on an already-authenticated load), the server route returns `{ ssoToken }` and the client identifies once. Logout clears identity; the launcher stays. The secret is not in the client bundle.

## Rules

- Match the host app's auth, routing, and package manager. Reuse existing session helpers.
- Do not rename `ssoToken`.
- Do not ask the user whether they want identify.
- If you cannot tell where layout or auth live, ask one question, then continue.
- More detail: https://quackback.io/docs/widget/installation and https://quackback.io/docs/widget/identify-users
