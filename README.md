# Faustine skills

Skills the developer's coding agent (Claude Code, Cursor, Codex, …) runs inside
the developer's repository to install and keep Faustine in step with the app.
Faustine runs no agent of its own here: the developer's agent reads the skill,
writes into the developer's repo, with the developer watching — the model Vercel
Shop uses (`docs/initiatives/nextjs_web_agents/research/03`).

```bash
npx skills add faustine-ai/skills            # every skill, every agent you use
npx skills add faustine-ai/skills --skill install-faustine
```

| Skill | Does | Iteration |
|---|---|---|
| [`install-faustine`](skills/install-faustine/SKILL.md) | Mounts the Web Agent in a Next.js app or a SPA; identity from the server | I1 |
| [`enable-faustine-adapter`](skills/enable-faustine-adapter/SKILL.md) | Generates the adapter for Server Actions / data layer (`npx faustine-next init`) | I2 |
| [`sync-faustine-adapter`](skills/sync-faustine-adapter/SKILL.md) | Audit · plan · apply: brings the registry in step after new, renamed or removed actions (`npx faustine-next sync`), one change at a time | I3 |

Layout follows the `skills` CLI convention: one folder per skill under
`skills/`, a `SKILL.md` with frontmatter (`name`, `description` with the
trigger phrases, `argument-hint`) and the steps; `.claude/skills` is a symlink
to `skills/` for repos that install without the CLI.
