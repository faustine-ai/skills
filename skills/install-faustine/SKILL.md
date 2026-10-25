---
name: install-faustine
description: Install the Faustine Web Agent in this application — a Next.js app (App Router or Pages Router) or a client-rendered SPA (React, Vue, Angular, Svelte). Use when the user asks to add, install, mount or set up the Faustine agent, assistant or widget, or pastes a Faustine Web Agent id (a UUID), a Faustine setup code (fns_…) or an embed snippet.
argument-hint: "[agent id] [--setup <code>] [--env <name>] [--debug]"
---

# Install the Faustine Web Agent

You are adding the Faustine widget to the application in the working directory.
The widget and the agent's generated SDK load at runtime from Faustine; nothing
of Faustine's runs on this server. The job is to mount one client component (or
one script tag), pass a verified identity when the app has a session, and check
it in the browser.

## Before editing

1. **Find the agent id.** It is a UUID, shown in the portal under the Web
   Agent's *Install the assistant* card, which gives this whole command line. If the user did not give one, ask for it — never invent one.
   `--setup <code>` (a `fns_…` setup code) means the app also gets the
   Faustine adapter: see step 5.
2. **Classify the app** — read `package.json`, the top-level folders and the
   lockfile before touching anything:
   - `next` in dependencies and `app/` or `src/app/` with a `layout.*` → **Next.js App Router**.
   - `next` and `pages/` or `src/pages/` without an app tree → **Next.js Pages Router**.
   - No `next` → **SPA**: find the entry (`main.tsx`, `src/index.tsx`, `index.html`).
   - A workspace (`pnpm-workspace.yaml`, `workspaces` in `package.json`, `turbo.json`, `nx.json`) → ask which app before continuing.
3. **Classify the installation:**
   - **fresh** — no `@faustine-ai/web-agent` in dependencies, no `Faustine.init`, no `webagent.faustine.ai` script → install.
   - **present** — any of those exists → extend, do not duplicate: report where it is mounted and only change what the user asked (a new agent id, an env, identity).
4. **Decide, then ask only what the repo cannot answer:**
   - Package or script tag? Default to the package when the app has a bundler, the script tag otherwise.
   - Does the app have a server-side session the agent should identify users with? Look for Auth.js / NextAuth (`auth()`, `getServerSession`), Clerk (`auth()`, `currentUser()`), Supabase (`createServerClient`), or a custom `cookies()` read. If yes, the identity JWT is signed on the server (step 3 below) and needs `FAUSTINE_IDENTITY_SECRET` from the portal (*Settings → Identity verification*); ask the user to add it to the environment — never paste a secret into the repo.
5. **Adapter or widget only?** When the user passed `--setup <code>`, when
   `faustine/registry.ts` already exists, or when the app keeps data behind
   Server Actions / server-only data functions and the user has a setup code:
   hand over to **`enable-faustine-adapter`** (same skills package,
   `../enable-faustine-adapter/SKILL.md`). It generates the registry and the
   route handler, mounts the component and wires the identity — its steps
   replace steps 1–4 below; come back here only for the verification. Without
   a code, install the widget only and tell the user where the command with a
   code comes from: the agent's Overview → *Install the assistant* → **Create
   install command**.

Noninteractive (no one to answer): stop and report what you need instead of guessing.

## Steps

### 1. Install the package

```bash
# Next.js
npm install @faustine-ai/next @faustine-ai/web-agent   # or pnpm add / yarn add, matching the lockfile
# SPA
npm install @faustine-ai/web-agent
```

Skip for the script-tag install (SPA / static site only; a Next.js app never
uses the script tag).

### 2. Mount the agent

**Next.js (App Router)** — `<FaustineAgent/>` from `@faustine-ai/next/client`,
rendered once from the root layout through a small client component. Pattern,
not a file to copy: match the repo's folder conventions (`components/`,
`src/`) and its style.

```tsx
// app/faustine-agent.tsx
'use client';
import { FaustineAgent } from '@faustine-ai/next/client';

export default function Agent({ identityJwt }: { identityJwt?: string | null }) {
  return (
    <FaustineAgent
      agentId="<AGENT_ID>"
      identityJwt={identityJwt}
      env={process.env.NEXT_PUBLIC_FAUSTINE_ENV}   // only when deployments serve different environments
    />
  );
}
```

Render `<Agent identityJwt={identityJwt} />` inside `<body>` of
`app/layout.tsx`, after `{children}`. The layout stays a Server Component.
Strict Mode's double effect is safe. With `--debug`, pass `debug` (dev/staging
only: the page opens a debug session for the portal's Tests tab — never in a
production build; read it from an environment variable if the repo deploys
both). If the layout cannot read the session itself (Cache Components,
`cookies()` outside a dynamic scope), read it in a small Server Component
rendered inside `<Suspense>` and pass the token down from there.

**Next.js (Pages Router)** — the same component rendered once in
`pages/_app.tsx`; the identity JWT comes from a route handler
(`pages/api/faustine-identity.ts`) the component fetches after mount.

**SPA** — call `Faustine.init({ agentId, env })` once in the app's entry module
(after the router is created, before the first render is fine). With a session
in the app, call `agent.identify({ jwt })` when the user is known and
`agent.logout()` on sign-out. `debug: true` in the same options for `--debug`.

**Script tag** (SPA / static site) — in the document head:

```html
<script src="https://webagent.faustine.ai/embed.js?id=<AGENT_ID>" async></script>
```

### 3. Identity from the server (when the app has a session)

Sign a short-lived HS256 JWT with the secret from the portal and pass it to
the client component as a prop. Only `sub` (the user id) is required; `name`
and `email` show in Logs. The secret is read on the server only.

Next.js: `faustineIdentityToken` from `@faustine-ai/next` does it, in the
Server Component that reads the session:

```ts
import { faustineIdentityToken } from '@faustine-ai/next';
const identityJwt = user ? faustineIdentityToken({ id: user.id, name: user.name, email: user.email }) : null;
```

Sign only when `FAUSTINE_IDENTITY_SECRET` is set, so a checkout without the
secret still runs (anonymous visitor). SPA: a route of the app's own backend
signs the same token (`jose` or `jsonwebtoken`, HS256, `sub` = user id,
`exp` ≈ 1h) and the client calls `agent.identify({ jwt })`.

### 4. Environment

- `NEXT_PUBLIC_FAUSTINE_ENV` (or the app's equivalent public variable) only
  when the app's deployments map to different environments of this Web Agent
  in the portal (its *Deployments*: Production, Staging…); default
  `Production`. Nothing else: the package knows where Faustine is.
- `FAUSTINE_IDENTITY_SECRET` without a public prefix, documented in
  `.env.example` with an empty value.

## Files touched

- The mount: `app/faustine-agent.tsx` + one line in `app/layout.tsx` (App Router,
  plus the small Server Component when the layout cannot read the session),
  `pages/_app.tsx` (Pages Router), or the entry module (SPA).
- `package.json` / lockfile when installing the package.
- `.env.example` when a variable was added.

Nothing else. No Faustine code in API routes, middleware or build config.

## Verification

1. `npm run build` (or the repo's build) passes; `lint` if the repo has it.
2. Start the dev server and open a page: the launcher bubble appears once, also
   after a client-side navigation and after a hard refresh.
3. With a session: sign in, open the widget and send a message — the portal's
   Logs show the conversation with a **Verified** badge. Without one, the
   visitor is anonymous and that is expected.
4. Next.js App Router: ask the agent to open a page (`sdk.navigate`) — the page
   renders, not only the URL. After an action that changes data, the page shows
   the new data without a manual reload.

## Guardrails

- Never commit or print `FAUSTINE_IDENTITY_SECRET`.
- Never mount the agent more than once, never in a component that re-renders per route.
- Do not change the app's auth, routing or data code; this skill installs the widget only.
- If the app's data lives in Server Actions or server-only data functions and
  there is no setup code, say so: the agent reaches route handlers and external
  APIs now; those operations need the adapter, which this skill runs when the
  command carries `--setup <code>` (the Overview's install card writes it).
