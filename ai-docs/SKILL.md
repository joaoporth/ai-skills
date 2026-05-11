---
name: ai-docs
description: >
  AI-first JSDoc convention so other AIs reading TypeScript code cold
  understand business rules, side effects, errors, and decisions without
  prior conversation. Covers functions, classes, components, hooks,
  types, enums, business-rule constants. Each artifact is documented as
  a mini-skill with sections: what it does, rules (with rationale and
  consequence), typed errors, side effects, cross-references. Principle
  9 (CORE) requires editing code and updating JSDoc in the SAME edit,
  since TypeScript does not validate doc contents and only compliance
  prevents doc rot. Stack-agnostic — works with any TS framework, ORM,
  or validator. USE to write or review docstrings for exported
  artifacts, edit existing code with same-edit doc update, establish
  AI-first convention in a new codebase, audit for doc rot or missing
  context. DO NOT USE for external docs (README, changelog) or to
  comment trivial private helpers (over-doc anti-pattern).
---

# AI-First Code Documentation

A convention so that **the next AI reading your code cold** understands context, rules, and decisions without needing prior conversation. Each documented artifact is an inline mini-skill.

## When to use

Use when:
- ✅ Writing/editing an **action / use case** (business logic)
- ✅ Writing **HTTP procedure / handler** (auth, rate-limit, response shape)
- ✅ Writing a **schema** with non-obvious rules (regex, ranges, refinements)
- ✅ Writing an **MCP tool** or **LLM-callable function** (LLM reads description to decide use)
- ✅ Writing a **public component** (composition, slots, events)
- ✅ Writing a **custom hook** with non-obvious effects
- ✅ Writing a **class** with clear responsibility (calculators, builders, services)
- ✅ Writing a **type/interface** with non-obvious semantics/transitions (state machines, RBAC tuples)
- ✅ Writing an **enum** with values needing explanation (cost, side effects per value)
- ✅ Writing a **business-rule constant** (arbitrary limits with rationale, multipliers, thresholds)
- ✅ **Editing** any of the above (principle 9 — update JSDoc in the same edit)
- ✅ Refactoring an existing function and the docstring is stale

**Don't use** for:
- ❌ Short private helpers (3-10 self-explanatory lines)
- ❌ Trivial getters/setters
- ❌ Re-exports (`export * from ...`)
- ❌ Self-explanatory constants (`export const MAX_RETRIES = 3`) — `as const` + good name suffices
- ❌ Tests (Vitest/Jest descriptions already document)

---

## Core principles

1. **AI-first = cold-read.** The next AI arrives without prior conversation — docs must be **self-contained**. Don't say "as we discussed earlier" or "you know the rest".
2. **WHY > WHAT.** Code already says what. Docs explain **why** + **consequence of breaking the rule**. Skip docs when WHY is obvious from the function name.
3. **Cross-references** to project skills/files (e.g., `your-stack/auth.md §RBAC`) — the next AI loads them on demand. **No refs?** Doc is probably too shallow.
4. **Rules with rationale + consequence.** Pattern: "Don't allow X **because** Y. If allowed, **happens** Z."
5. **Explicit side effects.** Queue jobs, audit logs, events, emails, webhooks — all listed.
6. **Typed errors.** Every `throw new <Error>(...)` documented with the scenario that triggers it.
7. **Non-obvious decisions** with ISO date (`YYYY-MM-DD`) — explains reversals, approach changes, workarounds. **No inline temporal markers** (⭐ NEW, ⭐ CHANGED, "recently", "now") in any other section — they fossilize. Diffs live in git/PR; JSDoc is current state; history goes in `## Non-obvious decisions` (dates don't age).
8. **Don't duplicate TypeScript.** Don't write `@param email - email string` when the signature already says it. Docs add context that TS doesn't capture.
9. **🚨 Editing code = updating JSDoc in the SAME edit.** TypeScript does NOT validate JSDoc contents — only this rule prevents doc rot. Applies to **any documented artifact**: function, class, method, component, hook, type, interface, enum, business-rule constant. Behavior changed? `## What it does`. New rule? `## Rules`. New throw? `## Typed errors`. New queue/audit/event/email? `## Side effects`. Non-obvious decision? `## Non-obvious decisions` with `YYYY-MM-DD`. **Never "leave it for later"** — later means never.

---

## Minimum template (action / use case)

3 **required** sections: `What it does`, `Rules`, `Errors`. Others are optional — include only when relevant.

```ts
/**
 * # <module>.<verbNoun> — <short human title>
 *
 * ## What it does
 * <1-3 sentences describing the operation at business level (not implementation).>
 *
 * ## Business rules (invariants)
 * - **<Short rule>**
 *   - Why: <reason>
 *   - Consequence: <what happens if broken>
 * - **<Another rule>**
 *   - ...
 *
 * ## Typed errors
 * - `<CODE>` — <scenario> (e.g., `CONFLICT` — email already exists)
 * - `<CODE>` — <scenario>
 *
 * --- (sections below are OPTIONAL — include only when applicable) ---
 *
 * ## Side effects
 * - <Job queued / audit log / event emitted / email sent / webhook>
 *
 * ## References
 * - Schema: `./<file>.schema.ts`
 * - Policy X: `@/shared/<file>.ts`
 * - Convention: skill `<skill-name>/<file>.md §<section>`
 *
 * ## Non-obvious decisions (history — only what's NOT obvious from git log)
 * - YYYY-MM-DD: <decision> (rationale if still relevant)
 */
export async function verbNoun(input: Input): Promise<Output> {
  // ...
}
```

### Full example (auth.register)

```ts
/**
 * # auth.register — Register a new user
 *
 * ## What it does
 * Creates User + Profile in a transaction. Queues welcome email via job
 * queue. Returns Session for front-end auto-login after signup.
 *
 * ## Business rules (invariants)
 * - **Unique email** (lowercase comparison)
 *   - Why: two users with the same email break consistent login
 *   - Consequence: `CONFLICT` before insert; DB unique constraint is safety net
 * - **Password ≥ 8 chars with 1 letter + 1 number**
 *   - Why: OWASP policy + compliance requirement
 *   - Consequence: `BAD_REQUEST` with Zod issue before touching DB
 * - **Email does NOT need to be verified to create account**
 *   - Why: product allows browse pre-verification (deliberately light UX)
 *   - But: write features require `emailVerifiedAt != null` (see `your-stack/auth.md §Email Verification`)
 *
 * ## Typed errors
 * - `CONFLICT` — email already exists (issue field: `email`)
 * - `BAD_REQUEST` — password fails policy (issue field: `password`)
 *
 * ## Side effects
 * - Queues job `notifications.send-welcome` (queue `email`)
 * - Writes AuditLog `auth.user.created` with IP + user agent
 * - Emits event `user.registered` on internal event bus
 *
 * ## References
 * - Schema: `./register.schema.ts`
 * - Password policy: `@/shared/password-policy.ts`
 * - Welcome template: `@/modules/notifications/templates/welcome.tsx`
 * - Convention: skill `your-stack/auth.md §Registration` (if applicable)
 *
 * ## Non-obvious decisions
 * - 2026-03-15: removed password case-sensitivity (bcrypt normalizes)
 * - 2026-04-02: audit log added per GDPR/compliance requirement (incident DPA-2026-03)
 */
export async function register(input: RegisterInput): Promise<RegisterOutput> {
  // ...
}
```

---

## Templates per artifact type

### HTTP procedure / handler (any framework — Express, Fastify, Hono, tRPC, etc.)

```ts
/**
 * # billing.createCheckout — POST /api/v1/checkouts
 *
 * ## Type
 * Mutation procedure (auth required + rate-limit 30/min/user)
 *
 * ## What it does
 * Creates a Stripe Checkout Session + local intent record. Returns
 * redirect URL for the hosted checkout. Front-end redirects the user.
 *
 * ## Auth + Permissions
 * - Requires authenticated user
 * - No additional RBAC — any user can create their own checkout
 *
 * ## Response shape
 * - Internal client (e.g., tRPC): `{ url, checkoutId }` directly
 * - Public REST: wrapped in `{ data: { url, checkoutId } }` (envelope convention)
 *
 * ## LLM/MCP exposure (if applicable)
 * - `destructiveHint: false` (checkout itself isn't destructive)
 * - `openWorldHint: true` (creates Stripe charge — external $ effect)
 * - `idempotentHint: false` (calling 2x creates 2 checkouts)
 *
 * ## Typed errors
 * - `PRECONDITION_FAILED` — user has no default payment method
 * - `SERVICE_UNAVAILABLE` — payment provider API down (client may retry)
 *
 * ## References
 * - Action: `@/modules/billing/actions/create-checkout`
 * - Schema: `@/modules/billing/actions/create-checkout.schema`
 */
export const createCheckoutProcedure = /* ... */
```

### Schema with refinements (Zod, Yup, Valibot, ArkType, etc.)

```ts
/**
 * # CreateInvoiceInput — Schema for creating an invoice
 *
 * ## Field rules
 * - `customerId`: prefixed ID `cus_<cuid>` — must match `PREFIXES.customer`
 * - `dueDate`: today or future date
 *   - Why: retroactive invoicing is admin-only (separate procedure `admin.createRetroInvoice`)
 * - `lineItems`: array of 1..50 items
 *   - Why: arbitrary cap to avoid huge payloads; Stripe accepts 25
 *   - If raising: verify provider integration A + provider B limits
 * - `discountPercent`: 0..100
 *
 * ## Cross-field refinement
 * - If `discountPercent > 0`, then `discountReason` is required
 *   - Why: audit trail — finance requires justification per discount
 *   - Consequence: Zod issue on `discountReason` if missing
 *
 * ## References
 * - Action consumer: `./create-invoice.ts`
 * - ID prefixes: `@/shared/id.ts`
 */
export const CreateInvoiceInput = /* ... */
```

### LLM-callable tool (MCP, function calling, agents)

> **Note:** LLM tools have **dual documentation** — JSDoc for the editing AI **and** `description` field that the LLM client reads at call time.

```ts
/**
 * # billing_list_invoices (tool name)
 *
 * ## Base procedure
 * `listInvoicesProcedure` in `routers/billing-router/list-invoices.ts`
 *
 * ## MCP/tool annotations
 * - `readOnlyHint: true` → client doesn't ask for confirmation
 * - `idempotentHint: true` → client can safely retry
 * - `destructiveHint: false`
 * - `openWorldHint: false` → only touches authenticated user data
 *
 * ## LLM-facing description (in .meta or tool registration)
 * "Use when user asks about invoices, billing history, unpaid bills, or
 *  payment status. Returns up to 50 invoices ordered by most recent.
 *  Filter by status: paid (settled), open (issued, not paid), overdue
 *  (past due_date), or void (canceled). Read-only — does NOT modify."
 *
 * ## References
 * - Procedure: `@/routers/billing-router/list-invoices`
 * - Action: `@/modules/billing/actions/list-invoices`
 */
```

### Public React component

```tsx
/**
 * # InvoiceCard — Invoice summary card
 *
 * ## What it renders
 * Semantic `<article>` with status badge, formatted amount, and contextual
 * CTA. Status `open` → "Pay now"; other statuses have no CTA by default.
 *
 * ## Composition (slots)
 * - `actions` — node on the right (overrides default CTA). Useful for "view receipt" in paid invoices
 * - `footer` — node at the bottom. Useful to show `daysOverdue` on overdue invoices
 *
 * ## Events
 * - `onPayClick` — fired when user clicks "Pay now" (only when status='open')
 * - `onViewClick` — fired when user clicks any area of the card
 *
 * ## Accessibility
 * - `<article>` with `aria-labelledby` pointing to the title
 * - CTA has `aria-describedby` pointing to status badge
 * - Click target: full card (mobile-friendly); CTA has `stopPropagation`
 *
 * ## References
 * - Invoice type: `@api/modules/billing/actions/list-invoices.schema`
 */
export function InvoiceCard(props: InvoiceCardProps) { /* ... */ }
```

### Custom hook (React, Vue composable, Svelte store, etc.)

```ts
/**
 * # useDebounce — Debounced value for query triggers
 *
 * ## What it does
 * Returns `value` after `delay` ms of no change. Auto-resets on each
 * change. Type-preserving (`T` in, `T` out).
 *
 * ## When to use
 * - Search input that triggers `useQuery({ q: debounced })`
 * - URL-state filters with TanStack Query (avoids refetch on every keystroke)
 *
 * ## When NOT to use
 * - State that does NOT feed query/network — `useState` directly suffices
 * - Discrete events (click, submit) — no "value changing over time"
 * - Inputs that need immediate feedback (validation, character counter)
 */
export function useDebounce<T>(value: T, delay: number): T { /* ... */ }
```

### Class

```ts
/**
 * # InvoiceCalculator — Calculates invoice totals, discounts, and taxes
 *
 * ## Responsibility
 * Encapsulates invoice calculation logic (subtotal, cascading discounts,
 * jurisdiction-based taxes, final total). Immutable — instantiated once,
 * methods return new values.
 *
 * ## When to use
 * - Preview calculation in `createInvoice` before persisting
 * - Recalculation in `applyDiscount` when user adds a coupon
 *
 * ## Instance invariants
 * - **`lineItems` is frozen in the constructor** (deep frozen)
 *   - Why: calculations may repeat in React renders
 * - **Values in cents** (integer) in all methods
 *   - Why: Money primitive — float breaks fiscal precision
 *
 * ## Public methods
 * - `subtotal(): number` — sum of lineItems without discount/tax
 * - `discount(percent: number): number` — discount amount, max 100%
 * - `tax(jurisdiction: Jurisdiction): number` — depends on tax rules
 * - `total(): number` — subtotal - discount + tax
 *
 * ## References
 * - Money primitive: `@/shared/primitives.ts`
 * - Tax rules: `@/shared/tax-rules.ts`
 */
export class InvoiceCalculator {
  constructor(private readonly lineItems: ReadonlyArray<LineItem>) {
    Object.freeze(this.lineItems)
  }
  // ...
}
```

Each public method can have its own JSDoc if non-trivial. Trivial methods (`subtotal()` summing an array) don't need it.

### Type / Interface

Document when **semantics aren't obvious from the structure**.

```ts
/**
 * # SubscriptionStatus — Canonical Subscription state
 *
 * ## Valid transitions
 * - `trialing` → `active` (after first payment)
 * - `trialing` → `canceled` (canceled during trial)
 * - `active` → `past_due` (payment failed, retrying)
 * - `past_due` → `active` (payment recovered)
 * - `past_due` → `canceled` (3 failed attempts)
 * - `active` → `canceled` (explicit cancellation)
 * - **NEVER revert** `canceled` → other (create a new subscription)
 *
 * ## Difference from Stripe
 * - Stripe has `trialing`/`active`/`past_due`/`canceled`/`unpaid`/`paused`
 * - We map `unpaid` and `paused` to `past_due` (we don't differentiate)
 *
 * ## References
 * - State machine: `@/modules/billing/subscription.state.ts`
 */
export type SubscriptionStatus = 'trialing' | 'active' | 'past_due' | 'canceled'
```

**Don't document** trivial types — values speak for themselves:

```ts
// ✅ Sufficient — values are self-explanatory
export type InvoiceStatus = 'paid' | 'open' | 'overdue' | 'void'
```

### Enum

```ts
/**
 * # NotificationChannel — Notification delivery channels
 *
 * ## Values
 * - `inApp` — toast in front + persisted for history view
 * - `email` — via email provider (queue `email`)
 * - `sms` — via SMS provider (regional SDKs)
 * - `webhook` — POST to customer-configured webhook (with retry)
 *
 * ## When to use each
 * - `inApp` → product notifications (new message, mention)
 * - `email` → confirmation, receipt, weekly report
 * - `sms` → 2FA, critical alert, payment reminder
 * - `webhook` → customer's external integration (product events)
 *
 * ## Cost
 * - inApp: $0
 * - email: ~$0.0001
 * - sms: ~$0.02
 * - webhook: $0 (customer pays their own infra)
 */
export enum NotificationChannel { /* ... */ }
```

### Business-rule constant

Constants with **arbitrary number + rationale** deserve JSDoc.

```ts
/**
 * # MAX_INVOICE_LINE_ITEMS — Maximum items per invoice
 *
 * ## Value
 * 50
 *
 * ## Why
 * - Payment provider accepts up to 25 line items per session
 * - Some of our integrations batch 2x (tax split)
 * - 50 is the real limit, but if raising, review: payment provider + ERP A + ERP B
 *
 * ## How to raise
 * 1. Confirm with integrations (payment, ERP A, ERP B) they handle it
 * 2. Migrate existing invoices at the limit (if any)
 * 3. Update schema (`CreateInvoiceInput.lineItems.max`)
 * 4. Update tests that verify the limit
 */
export const MAX_INVOICE_LINE_ITEMS = 50
```

**Don't document** self-explanatory constants:

```ts
// ✅ Sufficient — good name + obvious value
export const KB = 1024
export const MB = 1024 * KB
```

### File header (module barrel)

```ts
/**
 * @file billing.module — Barrel for the "billing" bounded context
 *
 * ## Responsibility
 * Owner of: Subscription, Invoice, Payment, Plan, Checkout
 *
 * ## What it exports
 * - Runtime actions: createCheckout, listInvoices, payInvoice, cancelSubscription
 * - Types: Invoice, Subscription, Plan, Customer
 * - Schemas: ListInvoicesInput, CreateCheckoutInput, ...
 *
 * ## What it does NOT export
 * - Private helpers (prefix `_` in actions/) — they don't leak from the module
 * - Internal mappers (`billing.mapper.ts`) — only actions consume
 */
export { createCheckout } from './actions/create-checkout'
// ...
```

---

## Keeping docs synced — **principle 9 in practice**

Doc rot is the biggest risk of inline documentation. **TypeScript doesn't catch it** — only compliance with the `editing = updating JSDoc in the SAME edit` rule does.

### Update triggers

Any of these code changes requires JSDoc review in the **same edit**:

| Code change | What to review in JSDoc |
|---|---|
| Renamed function / changed signature | Header `# <name>` + `## What it does` |
| Added a new parameter | `## What it does` if behavior changed; **DON'T** document the param itself (TS already does) |
| Added `throw new Error/TRPCError/...` | `## Typed errors` gets a new entry |
| Removed a `throw` | Remove from `## Typed errors` |
| Added `await sendEmail(...)` / queue / audit / event emit | `## Side effects` gets a new entry |
| Added auth/permission check | `## Business rules` gets an entry with rationale |
| Changed behavior of an existing rule | `## Rules` — update what it does + add to `## Non-obvious decisions` with date |
| Renamed referenced file | `## References` — update path |
| Removed/renamed function that appears in OTHER docs' `## References` | Search callers/refs (`grep`) and update all |
| Refactor without contract change | Typically no JSDoc update — but check if `## Non-obvious decisions` needs a note |
| Added public class method | Method gets its own JSDoc (if non-trivial rule) |
| Added value to enum/union type | `## Values` in the type/enum JSDoc |
| Changed business-rule constant (`MAX_X = 50` → `100`) | `## Value` + `## Why` + `## How to raise` |
| Changed return shape (DTO output) | `## What it does` + callers that document the type |

### Workflow: editing documented code

```
1. Identify the change
   - Pure refactor (no contract/rule/behavior change)? → JSDoc usually stays
   - Changed contract/rule/side effect/throw? → JSDoc needs update

2. Edit the code

3. In the SAME edit, review the JSDoc:
   - [ ] `## What it does` reflects CURRENT behavior?
   - [ ] `## Rules` lists ALL current rules (no removed ones, includes new ones)?
   - [ ] `## Typed errors` matches ALL throws in the function?
   - [ ] `## Side effects` lists ALL (queue, audit, email, event, webhook)?
   - [ ] `## References` points to paths that still exist?
   - [ ] Change is non-obvious (reversal, workaround, approach pivot)?
         → add to `## Non-obvious decisions` with `YYYY-MM-DD: <decision> (<rationale>)`

4. Check downstream impact
   - Did your function appear in `## References` of other docs?
     `grep -rn "<functionName>" --include="*.ts"` in files containing `/**`
   - Do any callers document side effects that changed?
   - Did any referenced type/interface change shape?

5. Check global conventions
   - Did the change break any project skill rule?
   - Does JSDoc still reference the correct `§section` of skill files?
```

### Doc drift detection (heuristic)

TypeScript doesn't catch it, but `git` can — files modified without JSDoc touched are drift candidates:

```bash
# Files modified in the last 7 days
git diff --name-only HEAD@{7.days.ago} HEAD -- "**/*.ts" | while read f; do
  CODE_CHANGED=$(git diff HEAD@{7.days.ago} HEAD -- "$f" | grep -cE "^\+\s*(export\s+)?(async\s+)?function|^\+.*throw\s+new|^\+.*await\s+(send|emit|enqueue|audit)")
  DOC_CHANGED=$(git diff HEAD@{7.days.ago} HEAD -- "$f" | grep -cE "^\+\s*\*\s")
  [ "$CODE_CHANGED" -gt 0 ] && [ "$DOC_CHANGED" -eq 0 ] && echo "⚠️  $f — code changed, JSDoc untouched"
done
```

Run as warning in pre-commit or code review. **Doesn't block commit** — just signals.

---

## When NOT to document (cost-benefit)

**Rotten docs are worse than no docs.** Every doc adds maintenance cost. Don't document if:

| Case | Why |
|---|---|
| Short private helper (3-10 self-explanatory lines) | Good name + code already say it |
| Trivial getters/setters | TS types already document |
| Simple re-exports / barrels | `export * from ...` needs no explanation |
| Self-explanatory constants | `export const MAX_RETRIES = 3` is enough |
| Tests | `it(...)` description already documents |
| Implementation that CHANGES frequently | Doc will rot — extract stable rule first |
| WHY is obvious from name | `function isAdmin(user)` needs no explanation |

**Hard rule:** if you can't write one line that **adds context the code doesn't capture**, DON'T write a docstring.

---

## Anti-patterns

- ❌ **Duplicate TypeScript** (`@param email - the user's email string`) — signature already says it
- ❌ **State the obvious** (`// save user to DB` above `prisma.user.create`) — method name already says it
- ❌ **Rotten doc** — reference to a function/file that was renamed/removed. TS doesn't catch it. Use git-based drift detection before commit
- ❌ **"Hack to fix bug X" without context** — always link to ticket/incident OR fully explain the bug
- ❌ **Document internal implementation** — becomes an anchor for refactor. Document the CONTRACT, not HOW
- ❌ **Over-doc on private helpers** — 3 self-explanatory lines don't need 20 lines of JSDoc
- ❌ **Skip `Typed errors`** when function throws — callers don't know which errors to propagate
- ❌ **Historical decisions without date + without still-relevant reason** — becomes noise. If reason no longer matters, remove
- ❌ **References without `§<section>`** — AI loads the whole 800-line file. Point to a section
- ❌ **Mixed languages** — JSDoc in English but variables/comments in Portuguese (or vice versa) breaks reading. Pick one
- ❌ **Hide side effects** — `createUser` that ALSO sends email without documenting surprises callers
- ❌ **MCP tool without `When to use` in description** — LLM client can't decide when to call
- ❌ **Editing function/class/component/hook/type without reviewing JSDoc in the SAME edit** — principle 9. "Later" = never. Silent doc rot
- ❌ **Adding `throw` without updating `## Typed errors`** — caller doesn't know to handle it
- ❌ **Adding `await sendEmail/audit/enqueue/...` without updating `## Side effects`** — caller surprised
- ❌ **Renaming file/function without updating `## References` in docs that point to it** — silent broken link
- ❌ **Subtle behavior change refactor without note in `## Non-obvious decisions`** — future AI does the same refactor again, or reverts without knowing
- ❌ **Class without header JSDoc** — only on methods. Header explains responsibility + when to instantiate
- ❌ **Type/enum/const with business rule but no JSDoc explaining it** (`MAX_X = 50` no reason) — someone changes to 100 without thinking about integrations
- ❌ **Inline temporal markers** (⭐ NEW, ⭐ CHANGED, ⭐ UPDATED, // ADDED TODAY, "recently added") in JSDoc — fossilize **in 1-2 months**. After that, "NEW" stops being new but the marker stays. Next AI thinks "this is recent, should review" and wastes time. **Diff lives in git/PR description**, JSDoc is current state. Temporal-scoped history goes in `## Non-obvious decisions` with **ISO date `YYYY-MM-DD`** — dates don't age (2026-05-11 is always 2026-05-11)
- ❌ **Time-relative words** ("recently", "now", "at the moment", "currently", "soon", "in the future") — meaning shifts with time. Use ISO date or remove
- ❌ **TODO/FIXME without date + owner + ticket** — becomes eternal garbage. Minimum format: `// TODO(2026-05-11 @owner #TICKET-123): <reason>`. Without that, becomes "old TODO nobody removes for fear"

---

## Verification — how to audit docs

```bash
# 1. Find public functions without JSDoc (heuristic)
grep -rL "/\*\*" src/modules/*/actions/

# 2. Find docs with old TODOs (>30d)
grep -rn "TODO\|FIXME\|XXX" src --include="*.ts"

# 3. Find broken refs (`@/path/...` that doesn't exist)
# Use TS check — TS catches broken imports but NOT docstrings mentioning paths
# Run drift heuristic before commit
```

**Heuristics for review (next AI doing code-review):**

- [ ] Every **public exported artifact** (function, class, component, hook, non-trivial type, enum, business-rule const) has a JSDoc header when applicable?
- [ ] Every business rule has "why" + "consequence"?
- [ ] Every reference to another file uses correct path?
- [ ] `Typed errors` lists ALL `throw new ...` in the function?
- [ ] Side effects (queue, audit, email, event, webhook) listed?
- [ ] Historical decisions have `YYYY-MM-DD` + still-relevant reason?
- [ ] No doc duplicating what TS already says?
- [ ] **Files modified in the last 7 days** had JSDoc reviewed? (run drift heuristic)
- [ ] Class has header explaining responsibility + invariants?
- [ ] Type/enum with non-obvious semantics has JSDoc explaining values/transitions?
- [ ] Arbitrary constant with rationale (`MAX_X = 50`) has JSDoc with "Why" + "How to raise"?
- [ ] **No inline temporal markers** (`⭐ NEW`, `⭐ CHANGED`, "recently", "currently")? History in `## Non-obvious decisions` with ISO date

---

## Adapt to your stack

The principles, structure, and anti-patterns are **stack-agnostic**. Only the inline examples reference specific libraries. Here's how to adapt the templates to your stack:

### HTTP framework

| You use | Adapt the `## Type` section to mention |
|---|---|
| **Express / Fastify** | Middleware chain (auth → rate-limit → handler), response type (`res.json`, `reply.send`) |
| **Hono** | Middleware (`hono/secure-headers`, `hono/cors`), context (`c.req`, `c.json`) |
| **tRPC** | Procedure type (`publicProcedure` / `protectedProcedure` / `mutationProcedure`), `.meta()` for OpenAPI |
| **NestJS** | Decorators (`@Controller`, `@Get`, `@UseGuards`), DI providers, pipes |
| **Next.js Route Handlers / Server Actions** | `route.ts` HTTP method exports, `'use server'` actions, revalidation tags |

### Validator

| You use | What documentation looks like |
|---|---|
| **Zod** | `.refine()`, `.transform()`, `.describe()` per field |
| **Yup** | `.test()`, `.when()` (cross-field), `.label()` |
| **Valibot** | `pipe()` validators, `forward()` for cross-field |
| **ArkType** | Constraints in type expressions, narrowing predicates |
| **Effect Schema** | `Schema.filter()`, `Schema.transform()`, annotations |

For all of them, the JSDoc pattern is the same: explain **why** each non-obvious validation exists.

### ORM

| You use | Adapt `## Side effects` to mention |
|---|---|
| **Prisma** | `prisma.X.create/update/delete`, transactions (`$transaction`), middleware |
| **Drizzle** | `db.insert/update/delete`, relational queries, migrations via `drizzle-kit` |
| **Kysely** | Query builder chains, raw escapes via `sql\`...\`` |
| **TypeORM** | Repositories, decorators (`@Entity`, `@Column`), subscribers |
| **Mongoose** | Model methods, hooks (`pre('save')`, `post('remove')`) — hooks are CRITICAL side effects to document |

### Frontend framework

| You use | Adapt component template `## Composition` to mention |
|---|---|
| **React** | Slots (children, render props), context providers, `forwardRef` |
| **Vue** | Slots (`<slot>`), `defineExpose`, scoped slots, composables |
| **Svelte** | Slots, props (`$$props`), stores, `<svelte:fragment>` |
| **Solid** | Signals, `children` (resolveChildren), props with `mergeProps` |
| **Qwik** | Signals, `$` boundary, `useTask$`, resumability constraints |

### Job queue

| You use | What `## Side effects` describes |
|---|---|
| **BullMQ / Bull** | Queue name, job name, delay/retry config |
| **Inngest** | Function name, triggers (event/cron), step semantics |
| **Trigger.dev** | Task ID, retry strategy, idempotency |
| **AWS SQS / GCP Pub/Sub** | Queue/topic ARN, message format |
| **Temporal** | Workflow ID, activity calls |

### LLM SDK (for AI features)

| You use | Adapt `## LLM exposure` to mention |
|---|---|
| **Vercel AI SDK** | `tool({ description, inputSchema, execute })`, `experimental_telemetry` |
| **OpenAI SDK** | Function calling format (`tools: [{ type: 'function', function: ... }]`) |
| **Anthropic SDK** | `tool_use` content blocks, `input_schema` JSON Schema |
| **MCP server** | Annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`) |
| **LangChain** | Tool class, agent type, callback handlers |

### Cross-references

Replace examples like `your-stack/auth.md §RBAC` with whatever skill/doc collection your project uses:

- Have a `docs/` folder with `.md` files? → `docs/auth.md §RBAC`
- Have a Confluence space? → `confluence://team/auth-conventions#rbac`
- Have a custom Claude skill? → `<your-skill>/auth.md §RBAC`
- Have nothing? → drop the cross-ref or link to the relevant action file directly

The principle is the same: **point the next AI to the next source of truth** so it doesn't have to discover from scratch.

---

## Examples (before/after)

See [examples.md](references/examples.md) for concrete diffs:
- Action without docs → with full AI-first docs
- HTTP procedure without docs → with docs
- Schema with refinement → with docs explaining the refinement
- LLM tool with dry description → with LLM-friendly description
- React component with slots → with composition docs
- Hook with subtle effect → with "when to use / when NOT" docs
- Trivial helper → **kept without docs** (negative case: over-doc anti-pattern)
- Inline temporal markers (anti-pattern) → reformulated using `## Non-obvious decisions` with date
