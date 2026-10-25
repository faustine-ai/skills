---
name: enable-faustine-adapter
description: Add the Faustine adapter to this Next.js app so its Web Agent can call Server Actions and server-only data functions — generates faustine/registry.ts and the /api/faustine route handler with `npx faustine-next init`, mounts the agent, verifies in the browser. Use when the user asks to enable, add or set up the Faustine adapter, pastes a Faustine setup code (fns_…), or says the agent cannot reach their Server Actions.
argument-hint: "--setup <code> [--operations a.b,c.d]"
---

# Enable the Faustine adapter

The Web Agent's generated SDK runs in the browser and can only call what has
a URL. In this app some data lives behind Server Actions (`'use server'`) or
server-only data functions that Server Components call. The adapter gives
those a URL inside the app itself: a generated registry of bindings
(`faustine/registry.ts`) and a three-line catch-all route handler
(`app/api/faustine/[...op]/route.ts`). Faustine proposes the list; you and the
user review it in the diff. Faustine's own runtime never runs on this server.

Everything mechanical is done by the generator shipped with the package,
`npx faustine-next`. Your job is the parts it cannot know: which operations
the user wants exposed, where the identity comes from, where the component
mounts, and whether it all works in the browser.

## Before editing

1. **Classify the app.** Read `package.json`, `next.config.*`, the root layout
   (`app/layout.tsx` or `src/app/layout.tsx`), `proxy.ts` / `middleware.ts`
   and the auth helper. Confirm:
   - **App Router** (`app/` or `src/app/` with a `layout.*`). The adapter is
     App Router only; on a Pages Router app stop and say so.
   - `src/` or root layout; the `@/` alias in `tsconfig.json`.
   - A workspace (`pnpm-workspace.yaml`, `workspaces`, `turbo.json`) → ask
     which app, and run everything from that app's directory.
2. **Classify the installation:**
   - **fresh** — no `@faustine-ai/next` in dependencies, no `faustine/registry.ts`.
   - **present** — the registry exists: this is a sync, not an install. Run
     `npx faustine-next sync --dry-run`, show the diff, and change only what
     the user asked.
   - **partial** — the package is installed but the registry or the route
     handler is missing: run `init` again; it is idempotent and only writes
     what differs.
3. **The setup code.** The user gets it from the portal: the agent's Overview
   → *Install the assistant* → **Create install command** (it is the `--setup`
   argument of `/install-faustine`, which hands over to this skill). It
   looks like `fns_…`, expires in 30 minutes and is exchanged once: the first
   run that uses it (the dry-run) stores the adapter token on this machine,
   scoped to this Web Agent. Repeating the same `--setup <code>` afterwards
   reuses that token, and once the registry exists a run needs no code at all.
   If several tokens are stored and there is no registry yet, `--web-agent <id>`
   picks one. If you do not have a code, ask; never invent one.
4. **Decide with the user, before writing:**
   - **Which operations.** Run `npx faustine-next init --setup <code> --dry-run`
     first: it prints every operation Faustine found (Server Actions and data
     functions) and the ones it leaves out. Show the list; ask what should
     stay private; pass the kept ones with `--operations a.b,c.d` when the
     user wants fewer than all.
   - **Identity.** Does the app have a server-side session (Auth.js / NextAuth,
     Clerk, Supabase, a custom `cookies()` read)? Then the identity JWT is
     signed on the server with `FAUSTINE_IDENTITY_SECRET` (portal → Settings →
     Identity verification). Ask the user to add it to the environment; never
     paste it into the repo.
   - **Where to mount** `<FaustineAgent/>`: the root layout, unless the agent
     should appear only in a section of the app.

Noninteractive (no one to answer): stop and report what you need instead of
guessing which operations to expose.

## Steps

### 1. Install the package

```bash
npm install @faustine-ai/next @faustine-ai/web-agent   # or pnpm add / yarn add, matching the lockfile
```

### 2. Generate the adapter

```bash
npx faustine-next init --setup <code> --dry-run          # the plan: files and operations
npx faustine-next init --setup <code> [--operations …]   # writes the files, reports the registry hash to Faustine
```

Without a terminal (CI, a scripted run) add `--yes`.

The generator writes:
- `faustine/registry.ts` (or `src/faustine/registry.ts`) — one line per
  operation: Faustine's name, an import of the real function and `params`, the
  parameter list read from its signature (positional arguments, `FormData`
  form actions and `useActionState` actions bind as they are — the handler maps
  the agent's input to the call). Read it with the user; a removed line is the
  way to keep something private. If a line has no `params`, the generator could
  not read the signature (a re-export, an unusual declaration): the function
  then receives the input as its one argument — say so to the user.
- `app/api/faustine/[...op]/route.ts` — three lines mounting the handler.
- A Faustine block in `AGENTS.md` so future agents keep the registry in step
  (`npx faustine-next sync`).

Never hand-write the registry or rename its operations: the names are what
the agent knows.

### 3. Mount the agent (if the app does not have it yet)

A client component rendered once in the root layout. Pattern, not a file to
copy — match the repo's folders and style:

```tsx
// app/faustine-agent.tsx
'use client';
import { FaustineAgent } from '@faustine-ai/next/client';
export default function Agent({ identityJwt }: { identityJwt?: string | null }) {
  return <FaustineAgent agentId="<AGENT_ID>" identityJwt={identityJwt} env={process.env.NEXT_PUBLIC_FAUSTINE_ENV} />;
}
```

```tsx
// app/layout.tsx — a Server Component: sign the identity here
import { faustineIdentityToken } from '@faustine-ai/next';
const user = await getSession();            // the app's own auth
const identityJwt = user ? faustineIdentityToken({ id: user.id, name: user.name, email: user.email }) : null;
// … <Agent identityJwt={identityJwt} /> inside <body>, after {children}
```

The agent id is the `webAgent` of the registry. Strict Mode's double effect is
safe. Optional: wrap `next.config` with `withFaustine()` so the handler reports
the commit it runs (the portal shows "app at a1b2c3").

### 4. Environment

- `FAUSTINE_IDENTITY_SECRET` (server only) when identity is used; document it
  in `.env.example` with an empty value.
- `NEXT_PUBLIC_FAUSTINE_ENV` only when the app's deployments map to different
  environments of this Web Agent in the portal (its *Deployments*: Production,
  Staging…); default `Production`. Nothing else: the package knows where
  Faustine is.

## Files touched

- `faustine/registry.ts`, `app/api/faustine/[...op]/route.ts`, `AGENTS.md` (generated).
- The mount: one client component + one line in `app/layout.tsx`.
- `package.json` / lockfile, `.env.example`, optionally `next.config.*`.

Nothing else. No changes to the Server Actions, the data functions, the auth
or the routing: the adapter imports them as they are.

## Verification

1. `npm run build` passes (a bad registry fails here, by design).
2. Start the app; `curl localhost:3000/api/faustine/manifest` answers the
   registry hash, the bound operations and `enabled` (what Faustine serves
   right now — empty until the Web Agent publishes a version with the adapter
   methods, which is expected on a first install).
3. `npx faustine-next doctor` agrees with the portal (same registry hash; no
   drift warnings other than "nothing enabled yet" on a fresh install).
4. In the browser: the launcher appears once; with a session, Logs show
   **Verified**. After the user publishes in the portal, ask the agent to do
   something that goes through a Server Action — the page shows the result
   without a reload (`sdk.refresh()`).

Then: commit, push, deploy with the app's own pipeline. The portal shows the
adapter as *deployed* once the running app has fetched its manifest.

## Guardrails

- Never call Server Actions from the browser directly, never read the RSC
  payload: the adapter is the only door.
- Never commit or print `FAUSTINE_IDENTITY_SECRET` or a setup code / adapter
  token (`fns_…`, `fna_…`).
- Never put a function in the registry that the user did not see in the
  dry-run list; never widen the list "to be safe".
- Reads written inline in a Server Component are not bindable; the surface
  map suggests the function to extract — propose it as a separate change, do
  not do it silently.
- Noninteractive runs stop when the operation list or the identity source is
  unresolved.
