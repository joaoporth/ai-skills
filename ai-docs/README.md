# ai-docs

> AI-first inline TypeScript documentation — write JSDoc so the **next AI reading your code cold** understands business rules, side effects, and decisions without prior conversation memory.

A Claude Code skill that establishes a **stack-agnostic convention** for documenting TS code (functions, classes, components, hooks, types, enums, business-rule constants) as inline mini-skills. Works with any framework, ORM, validator, or runtime.

## The problem

LLMs editing code today:

- Don't see your prior conversation
- Read only the file in front of them
- Don't know **why** rules exist, **what** side effects happen, **when** behavior changed

Result: refactors silently break business rules, side effects vanish unnoticed, decisions get reversed without context.

## The convention

Each exported artifact gets a JSDoc header structured as a mini-skill:

```ts
/**
 * # auth.register — Register a new user
 *
 * ## What it does
 * Creates User in a transaction. Queues welcome email. Returns Session
 * for the front-end to auto-login.
 *
 * ## Business rules (invariants)
 * - **Unique email (lowercase comparison)**
 *   - Why: two users with the same email break consistent login
 *   - Consequence: `CONFLICT` before insert; DB unique constraint is safety net
 * - **Password ≥ 8 chars with 1 letter + 1 number**
 *   - Why: OWASP policy + compliance requirement
 *
 * ## Typed errors
 * - `CONFLICT` — email already exists
 * - `BAD_REQUEST` — password fails policy
 *
 * ## Side effects
 * - Queues job `notifications.send-welcome` (queue `email`)
 * - Writes AuditLog `auth.user.created`
 *
 * ## Non-obvious decisions
 * - 2026-04-02: audit log added per GDPR requirement (incident DPA-2026-03)
 */
export async function register(input: RegisterInput): Promise<RegisterOutput> {
  // ...
}
```

The next AI reads this and **infers everything**: behavior, rules, errors, side effects, history — without asking the human, without prior context.

## What's in the skill

- **9 core principles** (cold-read, WHY > WHAT, cross-references, etc.)
- **Templates per artifact type**: function, class, HTTP procedure, schema, LLM tool, component, hook, type/interface, enum, business-rule constant
- **Principle 9 (CORE)**: editing code = updating JSDoc in the SAME edit (TS doesn't validate doc content; only this rule prevents doc rot)
- **Anti-patterns**: inline temporal markers (`⭐ NEW`, "recently") that fossilize, duplicating TS types, over-documenting trivial helpers, etc.
- **Verification heuristics**: git-based doc drift detection (warning, not block)
- **"Adapt to your stack" section**: map principles to Express/Fastify/Hono/tRPC/NestJS, Zod/Yup/Valibot/ArkType, Prisma/Drizzle/Kysely/TypeORM, React/Vue/Svelte/Solid, etc.

## Install

Place the `ai-docs/` folder under your Claude Code skills directory. See [Claude Code skills documentation](https://docs.claude.com/en/docs/claude-code/skills) for setup.

Once installed, Claude Code (or other agents that read Anthropic skill format) discovers the skill via the `description` field in `SKILL.md` and applies it automatically when:

- Writing or editing a documented artifact
- Reviewing existing code for doc compliance
- Auditing a codebase for doc rot

### Enforcing skill usage in your project

To make every code edit trigger ai-docs automatically, add to your project's `CLAUDE.md` (or equivalent agent instructions):

```markdown
## Documentation convention

All TypeScript code follows `ai-docs` — JSDoc updated in the SAME edit
as the code change (principle 9). See ai-docs/SKILL.md for templates.
```

For maximum compliance, also add a pre-commit hook running the [drift detection heuristic](SKILL.md#doc-drift-detection-heuristic) — non-blocking warning that flags files modified without JSDoc touched.

## Before/after examples

See [`references/examples.md`](references/examples.md) for concrete diffs:

| Example | What it shows |
|---|---|
| Action without docs → with full AI-first docs | Business rules, side effects, errors |
| HTTP procedure → with docs | Auth, response shape, LLM annotations |
| Schema with refinement → with rationale | Why each non-obvious validation exists |
| LLM tool dry description → LLM-friendly | "Use when..." + "Do NOT use for..." |
| React component with slots → with composition docs | Slots, events, a11y decisions |
| Hook → with when-to-use / when-NOT-to-use | Misuse prevention |
| Trivial helper → kept WITHOUT docs (negative case) | Over-doc anti-pattern |
| Inline temporal markers → reformulated with dated history | Fossilization prevention |

## Key anti-patterns

- ❌ **Inline temporal markers** (`⭐ NEW`, `⭐ CHANGED`, "recently") — fossilize in 1-2 months. Use `## Non-obvious decisions` with ISO date instead.
- ❌ **Duplicating TypeScript types** (`@param email - email string`) — TS already documents it.
- ❌ **Editing code without reviewing JSDoc in the same edit** — silent doc rot.
- ❌ **Hiding side effects** — `createUser` that also sends email without documenting surprises callers.
- ❌ **Over-doc on private helpers** — 3 self-explanatory lines don't need 20 lines of JSDoc.

Full list in [`SKILL.md`](SKILL.md#anti-patterns).

## 🇧🇷 Quando usar e quando NÃO usar (Português)

> Esta seção responde uma pergunta comum: *"essa convenção não consome mais crédito de IA?"* — sim, no curto prazo. **Mas economiza muito mais no médio prazo** se o projeto tem características certas. Use o guia abaixo pra decidir.

### Custo real

**Adiciona no curto prazo:**
- Cada função/classe/component documentado ganha 20-40 linhas de JSDoc (~150-300 tokens)
- IA lê esse JSDoc toda vez que abre o arquivo
- Princípio 9 obriga atualizar JSDoc na mesma edição = mais tokens de output

**Economiza no médio prazo:**
- IA não precisa redescobrir regras de negócio a cada conversa nova
- IA não reverte decisões antigas (`## Decisões não-óbvias` evita ciclo refactor → revert → refactor)
- Menos rodadas de clarification com você (*"o que essa função faz exatamente?"*)
- Cross-refs (`§seção`) carregam **sob demanda** — não custam upfront

### Break-even típico

| Edição | Sem ai-docs | Com ai-docs |
|---|---|---|
| **1ª escrita** | ~200 tokens (só código) | ~450 tokens (código + JSDoc) — **+250** |
| **2ª edição** (3 dias depois) | ~2-3k tokens (IA re-infere regras + pergunta pra você) | ~450 tokens (IA lê JSDoc, entende) — **-2k** |
| **Refactor 4 meses depois** | ~5-10k tokens (IA "limpa" código achando que é morto → você reverte no PR review) | ~450 tokens (IA vê `## Decisões` e não toca) — **-5k a -10k** |

**Conclusão prática:** depois de 2-3 edições na mesma função, ai-docs já pagou o custo inicial. Em projetos longos, o ROI é gigante.

### ✅ Use ai-docs.skill quando

- **Projeto de longo prazo** (>6 meses) com várias IAs/sessões trabalhando
- **Time pequeno** que depende de IA pra acelerar — múltiplas sessões em paralelo
- **Codebase com regras de negócio complexas** (financeiro, billing, compliance, multi-tenant) — ROI altíssimo
- **Trabalho recorrente de IA** (PR não-trivial por semana) — economia compounds
- **Você quer que outras IAs sigam o mesmo padrão** sem briefing manual a cada sessão
- **Vai compartilhar codebase com outros devs** (open-source, time crescendo, freelancers entrando/saindo)
- **Projeto produtivo** que precisa ser auditável (LGPD, SOX, finanças, healthcare)

### ❌ NÃO use ai-docs.skill quando

- **MVP descartável** — IA passa 1 vez, código vira lixo em 3 semanas. JSDoc é overhead puro
- **Codebase com churn altíssimo** (rewrite a cada sprint) — JSDoc rota antes de pagar o investimento
- **Time só humano sem IA** — IA não vai ler, humano lê código direto. Skill perde propósito
- **Protótipo / spike técnico** descartável após validar hipótese
- **Scripts utilitários one-shot** (migration que roda 1x, script de cleanup) — não vale doc
- **Quando o motivo da função é óbvio do nome** — `function isAdmin(user)` não precisa de doc; `function calculateProRatedRefund` precisa

### ⚠️ Use com moderação quando

- **Helper privados curtos** (3-10 linhas auto-explicativas) — não documente. Princípio "Don't document trivial"
- **Componentes triviais** (botão wrapper, ícone) — não precisa de doc de slots/eventos
- **Constantes simples** (`MAX_RETRIES = 3`) — nome já diz tudo
- **Tests** — `it(...)` description já documenta o comportamento

### Resumo em uma linha

> **Use ai-docs em código que VAI ser editado por IA várias vezes ao longo de meses. Pule em código que vai ser descartado em semanas.**

---

## Stack-agnostic

The principles, structure, and anti-patterns apply to **any TypeScript stack**. The `## Adapt to your stack` section in [`SKILL.md`](SKILL.md#adapt-to-your-stack) maps templates to:

- **HTTP frameworks**: Express, Fastify, Hono, tRPC, NestJS, Next.js Route Handlers / Server Actions
- **Validators**: Zod, Yup, Valibot, ArkType, Effect Schema
- **ORMs**: Prisma, Drizzle, Kysely, TypeORM, Mongoose
- **Frontend frameworks**: React, Vue, Svelte, Solid, Qwik
- **Job queues**: BullMQ, Inngest, Trigger.dev, AWS SQS, Temporal
- **LLM SDKs**: Vercel AI SDK, OpenAI SDK, Anthropic SDK, MCP, LangChain

## Why this matters

A codebase documented AI-first becomes a **multiplier**: every AI editing it carries forward the same context, decisions, and constraints — instead of rediscovering (and sometimes reverting) them every time.

The cost is one JSDoc per exported artifact, written once when the code is fresh in your head — and updated in the same edit as the code (principle 9).

The benefit is compounding: 6 months from now, a different AI on a different task picks up exactly where the previous one stopped.

## License

Free to use, modify, and distribute. No attribution required (though appreciated).

## Contributing

Issues and PRs welcome. Conventions of the skill apply to itself — if you change a section, update the corresponding examples and anti-patterns in the same PR.
