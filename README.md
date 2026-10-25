# Faustine skills

Skills for the coding agent you already use (Claude Code, Cursor, Codex, …) to
install the [Faustine](https://faustine.ai) Web Agent in your application and
keep it in step with your code. Your agent reads the skill and edits your repo
with you watching; nothing runs on Faustine's side.

```bash
npx skills add faustine-ai/skills                          # all three, for every agent you use
npx skills add faustine-ai/skills --skill install-faustine # just one
```

| Skill | What it does |
|---|---|
| [`install-faustine`](skills/install-faustine/SKILL.md) | Mounts the Web Agent in a Next.js app (App Router or Pages Router) or a client-rendered SPA, with the user's identity signed on your server. The portal's *Install the assistant* card gives the exact command line. |
| [`enable-faustine-adapter`](skills/enable-faustine-adapter/SKILL.md) | Next.js apps whose data lives behind Server Actions or server-only functions: generates the adapter (`faustine/registry.ts` and the `/api/faustine` route handler) with `npx faustine-next init`, so the agent can call them. |
| [`sync-faustine-adapter`](skills/sync-faustine-adapter/SKILL.md) | After actions are added, renamed or removed: audits what Faustine sees against what the registry binds and applies the change with `npx faustine-next sync`, one at a time. |

Each skill is a folder under `skills/` with a `SKILL.md`: frontmatter
(`name`, `description` with the phrases that trigger it, `argument-hint`) and
the steps. Without the CLI, copy the folder you need into your agent's skills
directory (for Claude Code, `.claude/skills/`).

The adapter skills use [`@faustine-ai/next`](https://github.com/faustine-ai/next).

MIT.
