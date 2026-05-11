# ai-skills

> Claude Code skills for **medium and large TypeScript projects** — battle-tested conventions for AI-assisted development at scale.

A curated collection of [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills) focused on the real problems that emerge when AI works alongside humans on production codebases over months and years: doc rot, refactor regressions, lost business context, redundant clarifications, decision reversals.

Each skill is **stack-agnostic**, **single-purpose**, and **self-contained** — drop into any TS project, no dependencies on each other.

## Who this is for

✅ Production SaaS / enterprise codebases (>6 months active development)
✅ Teams using AI agents for non-trivial PRs weekly
✅ Codebases with complex business rules (billing, compliance, multi-tenant)
✅ Open-source projects with rotating contributors
✅ Long-lived codebases where context-loss costs real money

❌ MVPs / throwaway prototypes (overhead > value)
❌ Pure scripts / utilities (one-shot code)
❌ Solo human projects without AI involvement (skill convention adds friction with no payoff)

## Skills available

| Skill | Purpose | Status |
|---|---|---|
| [**ai-docs**](./ai-docs) | AI-first inline TypeScript documentation convention — write JSDoc so the next AI reading your code cold understands business rules, side effects, and decisions without prior conversation memory | ✅ Stable |

More skills coming as they're battle-tested in real projects.

## What are Claude Code skills?

Skills are reusable conventions that AI agents discover and apply automatically. Each skill is a folder containing:

- `SKILL.md` — the convention itself, with frontmatter `description` that triggers the skill when relevant
- `references/` — supporting reference docs loaded on-demand
- `README.md` — human-facing overview

Claude Code (and other agents following the [Anthropic skill format](https://docs.claude.com/en/docs/claude-code/skills)) load the skill when the task matches its description. No manual invocation required for well-designed skills — the description does the discovery.

## Install

Clone or copy individual skills into your Claude Code skills directory:

```bash
# Single skill
git clone https://github.com/joaoporth/ai-skills.git /tmp/ai-skills
cp -r /tmp/ai-skills/ai-docs ~/.claude/skills/

# Or whole collection
git clone https://github.com/joaoporth/ai-skills.git ~/.claude/skills/ai-skills
```

See [Claude Code skills documentation](https://docs.claude.com/en/docs/claude-code/skills) for full setup.

### Enforcing skills in your project

Add to your project's `CLAUDE.md` (or `AGENTS.md`) at the repo root:

```markdown
## AI conventions

This project uses the following skills from `ai-skills`:
- [ai-docs](https://github.com/joaoporth/ai-skills/tree/main/ai-docs) — JSDoc updated in the SAME edit as code (principle 9)

All AI agents working on this codebase MUST apply these skills.
```

## Philosophy

These skills share a common thesis:

> **AI working on a codebase repeatedly over months is a different problem than AI writing code once.** Optimize for the long-term loop, not the first prompt.

Concretely:
- **Cold-read context** — every artifact stands alone, no prior conversation memory required
- **WHY > WHAT** — code says what; docs explain why + consequence
- **Atemporal** — no `⭐ NEW` markers that fossilize; history goes in dated sections (ISO `YYYY-MM-DD`)
- **Update in the same edit** — principle that prevents doc rot (TS doesn't validate doc contents; only this rule does)
- **Stack-agnostic** — adapt to Express/Fastify/Hono/tRPC, Zod/Yup/Valibot, Prisma/Drizzle/Kysely, React/Vue/Svelte/Solid, etc.

## License

Free to use, modify, and distribute. No attribution required (though appreciated).

## Contributing

Issues and PRs welcome:

1. Read the relevant skill's `SKILL.md` to understand its conventions
2. If the skill defines its own contribution rules (`ai-docs` requires JSDoc updated in the same PR as code), follow them
3. Open a PR with clear "what changed and why"

For new skills: open an issue first to discuss scope and fit with the collection's philosophy.

## 🇧🇷 Sobre

Coleção de Claude Code skills focada em **projetos TypeScript de médio e grande porte** — convenções testadas em produção pra problemas reais que aparecem quando IA trabalha em codebases ativos por meses e anos.

Cada skill é **stack-agnostic**, **independente**, e **autocontida** — você adota uma sem precisar adotar as outras.

**Comece por `ai-docs`** — convenção de documentação inline AI-first que economiza horas de re-descoberta de contexto a cada sessão nova de IA. Veja [`ai-docs/README.md`](./ai-docs/README.md) pra análise de custo/benefício e quando vale a pena adotar.
