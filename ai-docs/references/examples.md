---
name: ai-docs/examples
description: >
  Concrete before/after examples per code type — action, HTTP procedure,
  schema with refinement, LLM tool, React component, custom hook. Includes
  negative case (trivial helper kept without docs — over-doc anti-pattern)
  and inline-temporal-markers anti-pattern with the correct alternative.
  USE TO copy a template applied to a specific type, see the exact diff
  to add, validate that your docs aren't over-doc.
---

# Before/After Examples

Real diffs. Left side = code without docs (or bad docs). Right side = AI-first compliant.

Examples use a generic TypeScript backend stack (Prisma + Zod + tRPC-style procedures) as illustration — the same patterns apply with any ORM, validator, or framework (see `## Adapt to your stack` in SKILL.md).

---

## 1. Action / use case — `auth.register`

### Before

```ts
import { hash } from 'argon2'
import { prisma } from '@/db'
import { TRPCError } from '@trpc/server'
import { id } from '@/shared/id'
import { sendWelcomeEmail } from '@/modules/notifications/notifications.module'
import { auditLog } from '@/modules/audit/audit.module'
import type { RegisterInput, RegisterOutput } from './register.schema'

export async function register(input: RegisterInput): Promise<RegisterOutput> {
  const existing = await prisma.user.findUnique({ where: { email: input.email.toLowerCase() } })
  if (existing) throw new TRPCError({ code: 'CONFLICT', message: 'email already registered' })

  const passwordHash = await hash(input.password)
  const user = await prisma.user.create({
    data: {
      id: id('user'),
      email: input.email.toLowerCase(),
      passwordHash,
      name: input.name,
    },
  })

  await sendWelcomeEmail({ userId: user.id, email: user.email })
  await auditLog({ event: 'auth.user.created', userId: user.id })

  return { userId: user.id, email: user.email }
}
```

### After

```ts
import { hash } from 'argon2'
import { prisma } from '@/db'
import { TRPCError } from '@trpc/server'
import { id } from '@/shared/id'
import { sendWelcomeEmail } from '@/modules/notifications/notifications.module'
import { auditLog } from '@/modules/audit/audit.module'
import type { RegisterInput, RegisterOutput } from './register.schema'

/**
 * # auth.register — Register a new user
 *
 * ## What it does
 * Creates User in a transaction. Queues welcome email via job queue.
 * Returns userId+email for the front-end to auto-login after signup.
 *
 * ## Business rules (invariants)
 * - **Unique email (lowercase comparison)**
 *   - Why: two users with the same email break consistent login
 *   - Consequence: `CONFLICT` before insert; DB unique constraint is safety net
 * - **Email does NOT need to be verified to create account**
 *   - Why: product allows browse pre-verification (deliberately light UX)
 *   - But: write features require `emailVerifiedAt != null` (see your auth conventions)
 * - **Password hashed with argon2id before touching DB** (security convention)
 *
 * ## Typed errors
 * - `CONFLICT` — email already exists (issue field: `email`)
 *
 * ## Side effects
 * - Queues job `notifications.send-welcome` (queue `email`)
 * - Writes AuditLog `auth.user.created`
 *
 * ## References
 * - Schema: `./register.schema.ts`
 * - Password policy (validators): `./register.schema.ts §RegisterInput.password`
 * - Welcome template: `@/modules/notifications/templates/welcome.tsx`
 *
 * ## Non-obvious decisions
 * - 2026-04-02: audit log added per GDPR/compliance requirement (incident DPA-2026-03)
 */
export async function register(input: RegisterInput): Promise<RegisterOutput> {
  const existing = await prisma.user.findUnique({ where: { email: input.email.toLowerCase() } })
  if (existing) throw new TRPCError({ code: 'CONFLICT', message: 'email already registered' })

  const passwordHash = await hash(input.password)
  const user = await prisma.user.create({
    data: {
      id: id('user'),
      email: input.email.toLowerCase(),
      passwordHash,
      name: input.name,
    },
  })

  await sendWelcomeEmail({ userId: user.id, email: user.email })
  await auditLog({ event: 'auth.user.created', userId: user.id })

  return { userId: user.id, email: user.email }
}
```

**What changed:**
- Header explains WHAT it does at business level
- Each rule has rationale + consequence
- Side effects (email queue, audit) listed — anyone calling this knows email will be sent
- Typed errors documented so callers don't have to duplicate try/catch
- Cross-refs guide the next AI to related docs
- 2026-04-02 decision explains the audit log so someone doesn't remove it "while cleaning"

---

## 2. HTTP procedure — `createCheckoutProcedure`

### Before

```ts
import { mutationProcedure } from '@/trpc/procedures'
import { createCheckout, CreateCheckoutInput, CreateCheckoutOutput } from '@/modules/billing/billing.module'

export const createCheckoutProcedure = mutationProcedure
  .input(CreateCheckoutInput)
  .output(CreateCheckoutOutput)
  .meta({
    openapi: {
      method: 'POST',
      path: '/checkouts',
      tags: ['Billing', 'mcp'],
      operationId: 'billing_create_checkout',
      summary: 'Create checkout',
    },
  })
  .mutation(({ ctx, input }) => createCheckout({ userId: ctx.user.id, ...input }))
```

### After

```ts
import { mutationProcedure } from '@/trpc/procedures'
import { createCheckout, CreateCheckoutInput, CreateCheckoutOutput } from '@/modules/billing/billing.module'

/**
 * # billing.createCheckout — POST /api/v1/checkouts
 *
 * ## Type
 * Mutation procedure (auth required + rate-limit 30/min/user)
 *
 * ## What it does
 * Creates a Stripe Checkout Session + local Checkout intent row. Returns
 * a redirect URL for the hosted checkout. Front-end redirects the user.
 *
 * ## Auth + Permissions
 * - Authenticated user required
 * - No additional RBAC — any user can create their own checkout
 *
 * ## Response shape
 * - Internal client: `{ url, checkoutId }` directly
 * - Public REST: wrapped in `{ data: { url, checkoutId } }` (envelope convention)
 *
 * ## LLM/MCP exposure
 * - `openWorldHint: true` — creates a charge on Stripe (external $$ effect)
 * - `idempotentHint: false` — calling 2x creates 2 checkouts
 * - `destructiveHint: false` — checkout itself isn't destructive, only the charge is
 *
 * ## Typed errors
 * - `PRECONDITION_FAILED` — user has no default payment method
 * - `SERVICE_UNAVAILABLE` — Stripe API down (client may safely retry)
 *
 * ## References
 * - Action: `@/modules/billing/actions/create-checkout`
 * - Schema: `@/modules/billing/actions/create-checkout.schema`
 */
export const createCheckoutProcedure = mutationProcedure
  .input(CreateCheckoutInput)
  .output(CreateCheckoutOutput)
  .meta({
    openapi: {
      method: 'POST',
      path: '/checkouts',
      tags: ['Billing', 'mcp'],
      operationId: 'billing_create_checkout',
      summary: 'Create checkout session',
      description: [
        'Use ONLY when user explicitly asks to start payment or subscribe to a plan.',
        'Creates Stripe Checkout Session and returns redirect URL.',
        'Charges money — confirm with user before calling.',
        'Idempotency NOT guaranteed — calling twice creates 2 checkouts.',
      ].join(' '),
    },
    mcp: {
      annotations: {
        destructiveHint: false,
        openWorldHint: true,
        idempotentHint: false,
      },
    },
  })
  .mutation(({ ctx, input }) => createCheckout({ userId: ctx.user.id, ...input }))
```

**What changed:**
- `## Type` documents it's a mutation with rate-limit
- `## Response shape` explains tRPC vs REST envelope difference — without it, a dev might try `body.url` instead of `body.data.url`
- `## LLM/MCP exposure` explains WHY each annotation is what it is — next AI doesn't change without understanding
- The OpenAPI `description` changed from "Create checkout" (dry) to an LLM-friendly paragraph with "Use ONLY when..."

---

## 3. Schema with refinement

### Before

```ts
import { z } from 'zod'
import { PrefixedId } from '@/shared/primitives'

export const LineItem = z.object({
  description: z.string().min(1).max(200),
  amount: z.number().int().positive(),
  quantity: z.number().int().min(1).default(1),
})

export const CreateInvoiceInput = z.object({
  customerId: PrefixedId('cus'),
  dueDate: z.coerce.date().refine((d) => d >= new Date(), 'invalid'),
  lineItems: z.array(LineItem).min(1).max(50),
  discountPercent: z.number().min(0).max(100).default(0),
  discountReason: z.string().min(3).max(200).optional(),
}).refine(
  (v) => v.discountPercent === 0 || !!v.discountReason,
  { message: 'reason required', path: ['discountReason'] },
)
export type CreateInvoiceInput = z.infer<typeof CreateInvoiceInput>
```

### After

```ts
import { z } from 'zod'
import { PrefixedId } from '@/shared/primitives'
import { startOfToday } from 'date-fns'

/**
 * # LineItem — Invoice line item
 *
 * ## Field rules
 * - `amount`: integer in cents (not float — Money primitive)
 * - `quantity`: ≥ 1; invoices with qty=0 make no sense
 */
export const LineItem = z.object({
  description: z.string().min(1).max(200),
  amount: z.number().int().positive(),
  quantity: z.number().int().min(1).default(1),
})

/**
 * # CreateInvoiceInput — Schema for creating an invoice
 *
 * ## Field rules
 * - `customerId`: PrefixedId('cus') — matches `PREFIXES.customer` in `@/shared/id`
 * - `dueDate`: today or future date
 *   - Why: retroactive invoicing is admin-only (separate procedure `admin.createRetroInvoice`)
 *   - Validation: rejects ISO strings via `z.coerce.date()` + refinement
 * - `lineItems`: array of 1..50 items
 *   - Why: arbitrary cap to avoid huge payloads
 *   - Stripe accepts 25 — if raising past 50, verify provider limit
 * - `discountPercent`: 0..100
 *
 * ## Cross-field refinement
 * - If `discountPercent > 0`, `discountReason` is required
 *   - Why: audit trail — finance requires justification per discount
 *   - Consequence: Zod issue on `discountReason` if missing
 *
 * ## References
 * - Consumer action: `./create-invoice.ts`
 * - PREFIXES: `@/shared/id.ts`
 */
export const CreateInvoiceInput = z.object({
  customerId: PrefixedId('cus'),
  dueDate: z.coerce.date()
    .refine((d) => d >= startOfToday(), 'dueDate must be today or later'),
  lineItems: z.array(LineItem).min(1).max(50),
  discountPercent: z.number().min(0).max(100).default(0),
  discountReason: z.string().min(3).max(200).optional()
    .describe('Required when discountPercent > 0. Used for audit trail.'),
}).refine(
  (v) => v.discountPercent === 0 || !!v.discountReason,
  { message: 'discountReason required when discountPercent > 0', path: ['discountReason'] },
)
export type CreateInvoiceInput = z.infer<typeof CreateInvoiceInput>
```

**What changed:**
- Each refinement documented with RATIONALE (fiscal audit) — anyone removing it "for simplicity" sees the reason
- Cross-field refinement explained — reading only the Zod code, it's easy to miss
- `.describe()` on the field improves MCP/OpenAPI (LLM reads it)
- Refs point to the consumer action (jump from schema to action without grep)

---

## 4. LLM-callable tool — friendly description

### Before (dry description)

```ts
openapi: {
  method: 'POST',
  path: '/tickets',
  operationId: 'support_create_ticket',
  summary: 'Create ticket',
  description: 'Creates a new support ticket.',
},
```

### After (LLM-friendly description)

```ts
/**
 * # support_create_ticket (MCP tool)
 *
 * ## Annotations
 * - `destructiveHint: false` — creating a ticket isn't destructive
 * - `idempotentHint: false` — N calls create N tickets
 * - `openWorldHint: true` — notifies team via Slack/email
 *
 * ## LLM description (in .meta({ openapi: { description } }))
 * Covers: usage scenarios, what NOT to use it for, expected format
 */
openapi: {
  method: 'POST',
  path: '/tickets',
  operationId: 'support_create_ticket',
  summary: 'Create support ticket',
  description: [
    'Use when user reports a problem, asks a question that needs human support,',
    'or requests a feature. Creates a ticket assigned to the support team by category.',
    '',
    'Category MUST be one of: bug, billing, feature_request, account.',
    'Priority: low (default), medium, high, urgent (only for production issues).',
    '',
    'Do NOT use for:',
    '- General FAQ — answer directly if you know',
    '- Sales inquiries — use `crm_create_lead` instead',
    '- Internal team escalation',
  ].join('\n'),
},
mcp: {
  annotations: {
    destructiveHint: false,
    idempotentHint: false,
    openWorldHint: true,
  },
},
```

**What changed:**
- Description became a multi-line block with **when to use**, **valid categories**, **when NOT to use**
- The LLM client reads this and correctly decides to call `support_create_ticket` vs `crm_create_lead`

---

## 5. React component — slots + composition

### Before

```tsx
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import type { Invoice } from '@api/modules/billing/actions/list-invoices.schema'

type InvoiceCardProps = {
  invoice: Invoice
  onPayClick?: () => void
  onViewClick?: () => void
}

export function InvoiceCard({ invoice, onPayClick, onViewClick }: InvoiceCardProps) {
  return (
    <article aria-labelledby={`invoice-${invoice.id}`} onClick={onViewClick}>
      <h3 id={`invoice-${invoice.id}`}>Invoice {invoice.id}</h3>
      <Badge variant={invoice.status === 'paid' ? 'success' : 'warning'}>{invoice.status}</Badge>
      <p>{invoice.amount.currency} {invoice.amount.amount / 100}</p>
      {invoice.status === 'open' && <Button onClick={onPayClick}>Pay now</Button>}
    </article>
  )
}
```

### After

```tsx
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import type { ReactNode } from 'react'
import type { Invoice } from '@api/modules/billing/actions/list-invoices.schema'

/**
 * # InvoiceCard — Invoice summary card
 *
 * ## What it renders
 * Semantic card (`<article>`) with status badge, formatted amount (Money
 * primitive), and a contextual CTA. Status `open` → "Pay now"; other
 * statuses have no default CTA.
 *
 * ## Composition (slots)
 * - `actions` — node on the right (overrides default CTA). Useful for "view receipt" in paid invoices
 * - `footer` — node at the bottom. Useful for showing `daysOverdue` on overdue invoices
 *
 * ## Events
 * - `onPayClick` — fired when user clicks "Pay now" (only when status='open')
 * - `onViewClick` — fired when user clicks anywhere on the card
 *
 * ## Accessibility
 * - `<article>` with `aria-labelledby` pointing to the title
 * - CTA has `aria-describedby` pointing to the status badge
 * - Click target: full card (mobile-friendly); CTA uses `stopPropagation`
 *
 * ## References
 * - Invoice type: `@api/modules/billing/actions/list-invoices.schema`
 */
type InvoiceCardProps = {
  invoice: Invoice
  actions?: ReactNode
  footer?: ReactNode
  onPayClick?: () => void
  onViewClick?: () => void
}

export function InvoiceCard({ invoice, actions, footer, onPayClick, onViewClick }: InvoiceCardProps) {
  // ...
}
```

**What changed:**
- `## Composition (slots)` explains the override pattern — consumer dev knows they can pass custom `actions`
- `## Accessibility` documents a11y decisions (aria, click target, stopPropagation) — refactor won't break a11y
- Cross-refs to the Invoice type

---

## 6. Custom hook — when to use / not to use

### Before

```ts
import { useEffect, useState } from 'react'

export function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value)
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delay)
    return () => clearTimeout(t)
  }, [value, delay])
  return debounced
}
```

### After

```ts
import { useEffect, useState } from 'react'

/**
 * # useDebounce — Debounced value for query triggers
 *
 * ## What it does
 * Returns `value` after `delay` ms of no change. Auto-resets on every
 * change. Type-preserving (`T` in, `T` out).
 *
 * ## When to use
 * - Search input that triggers `trpc.X.search.useQuery({ q: debounced })`
 * - URL-state filters with TanStack Query (avoids refetch on every keystroke)
 *
 * ## When NOT to use
 * - State that does NOT go to query/network — `useState` directly is enough
 * - Discrete events (click, submit) — no "value changing over time"
 * - Inputs that need immediate feedback (validation, character counter)
 */
export function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value)
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delay)
    return () => clearTimeout(t)
  }, [value, delay])
  return debounced
}
```

**What changed:**
- `## When to use` + `## When NOT to use` prevents misuse (validation should be instant, not debounced)
- Next AI sees the hook in autocomplete and knows whether it fits their case

---

## 7. NEGATIVE case — trivial helper kept WITHOUT docs

```ts
// apps/api/src/shared/utils/sleep.ts

// ✅ KEEP AS-IS — good name + good types + short body. JSDoc would be over-doc.
export function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms))
}
```

**Why NOT document:**
- Name `sleep` is universal — every dev/AI knows it
- Types already say everything (`ms: number → Promise<void>`)
- Body is one line
- Adding JSDoc would be **noise** (and more to rot when refactoring)

**Anti-pattern would be writing:**

```ts
/**
 * Sleep for the given number of milliseconds.
 * @param ms - The number of milliseconds to sleep
 * @returns A Promise that resolves after `ms` milliseconds
 */
export function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms))
}
```

**This JSDoc only repeats the type — adds zero context the code doesn't already have. DO NOT WRITE.**

---

## 8. Anti-pattern highlighted: temporal markers that fossilize

### Wrong (inline markers that age badly)

```ts
/**
 * # auth.register — Register a new user
 *
 * ## Side effects
 * - Creates EmailVerification token (24h)                ⭐ NEW
 * - Queues verification email                             ⭐ CHANGED (was welcome)
 * - Schedules cleanup-unverified job                      ⭐ NEW
 *
 * ## References
 * - Schema: ./register.schema.ts
 * - Verify action: ./verify-email.ts                      ⭐ NEW (created today)
 * - Cleanup job: ./cleanup-unverified.ts                  ⭐ NEW
 *
 * Recent changes: now requires mandatory verification.    ❌ "recent" ages
 */
```

**Problem:** in 1-2 months, nobody knows when "NEW" was written. Next AI thinks "this is recent, should review" and wastes time. "Recent changes" stops being recent. Permanent garbage.

### Right (stable current state + history in dedicated section with date)

```ts
/**
 * # auth.register — Register a new user
 *
 * ## Side effects
 * - Creates EmailVerification token (TTL 24h, cryptographically secure)
 * - Queues job `notifications.send-verification` (queue `email`)
 * - Schedules job `auth.cleanup-unverified` in 24h (idempotent)
 * - Writes AuditLog `auth.user.created`
 *
 * ## References
 * - Schema: `./register.schema.ts`
 * - Verify action: `./verify-email.ts`
 * - Cleanup job: `./cleanup-unverified.ts`
 * - Verification template: `@/modules/notifications/templates/verify-email.tsx`
 *
 * ## Non-obvious decisions
 * - 2026-04-02: audit log added per GDPR/compliance requirement
 * - 2026-05-11: email verification became mandatory (incident SPAM-2026-03).
 *   Auto-login removed. Welcome email replaced by verification email.
 *   Cleanup job deletes accounts unverified within 24h.
 */
```

**Why it works:**
- `## Side effects` and `## References` describe **current state** — no marker indicating "when added"
- "Before vs after" diff lives in **commit message** + **PR description** (git log preserves it)
- Important change goes in `## Non-obvious decisions` with **ISO date** — `2026-05-11` will always be 2026-05-11
- Next AI reads and understands: "ok, changed in May 2026 — since then this is how it works"

### Practical rule

**Ask yourself before writing:** "if someone reads this 1-2 months from now, will it still make sense?"

(1-2 months is the realistic threshold — code in an active project goes through dozens of PRs in that window; "NEW" and "recent" fossilize **fast**.)

- ✅ "Creates EmailVerification token (TTL 24h)" — atemporal, always true
- ❌ "Creates token (NEW)" — in 1-2 months, no longer new
- ✅ "2026-05-11: became mandatory" — fixed date, context preserved
- ❌ "recently became mandatory" — in 1-2 months, no longer recent
- ✅ "Replaced welcome email" — describes what IS (not what changed) — optional
- ❌ "Was welcome, now verification" — describes the diff. Diff lives in git, not in stable JSDoc

---

## Final review checklist

Next AI doing code review uses this checklist:

- [ ] Public exported function has JSDoc header if it has **business rule** or **side effect**?
- [ ] Every rule in `## Rules` has **"why"** + **"consequence"**?
- [ ] Side effects (queue, audit, email, event, webhook) listed in `## Side effects`?
- [ ] All `throw new <Error>(...)` documented in `## Typed errors`?
- [ ] References (`./<file>`, `@/<path>`, `skill/<name>.md §<section>`) point to existing paths?
- [ ] No duplication of what TS types already say?
- [ ] Short private helper does **not** have unnecessary JSDoc?
- [ ] Historical decisions have date (`YYYY-MM-DD`) + still-relevant reason?
- [ ] LLM tool description has "Use when..." + "Do NOT use for..."?
- [ ] Public component documents slots/composition/events/a11y?
- [ ] Hook documents "when to use" + "when NOT to use"?
- [ ] **No inline temporal markers** (`⭐ NEW`, `⭐ CHANGED`, "recently", "currently")? History goes in `## Non-obvious decisions` with ISO date
