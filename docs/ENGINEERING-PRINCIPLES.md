# Engineering Principles

> **Author:** Personal engineering philosophy
> **Status:** Living document — updated as thinking evolves
> **Scope:** Applies across all projects, phases, and stacks
>
> **Stack note:** U-* principles apply as-is. B-2 (Django async) and all F-* (React/Next.js) sections are placeholders — adapt or replace once this project's stack is confirmed.

---

## Philosophy

These are not rules imposed by a framework or a team. They are principles developed through building, debugging, and maintaining real systems. They exist to make systems understandable, changeable, and operable — not to look clever.

> Good architecture is boring, predictable, and easy to modify.
> Bad architecture looks smart, is impossible to onboard into, and breaks quietly.

---

# UNIVERSAL PRINCIPLES

*Apply to all code regardless of stack — backend, frontend, CLI, or otherwise.*

---

## U-1. Observability by Default

Observability is not a feature added at the end. It is a first-class engineering concern built from the start.

### Core requirements (both stacks)
- **Logging** — structured, queryable, separated by concern
- **Error aggregation** — grouped and prioritised (Sentry, etc.)
- **Tracing** — follow a request or user action through the full system
- **Metrics** — quantitative measurements over time (error rate, latency, usage)
- **Health checks** — is the system alive and ready?

### Backend (Django/Python)
- Use per-app loggers — never a single global logger
  - `logging.getLogger('accounts')`, `logging.getLogger('sessions')`
- Use structured logs (JSON) in production — machine-readable, queryable
- Separate log streams: application / security / audit / background worker
- Never log: passwords, tokens, secrets, sensitive PII
- Ship logs to external systems in production (Grafana/Loki, Elasticsearch, etc.)

### Frontend (React/Next.js)
- Use error boundaries to catch and report unhandled component errors
- Send uncaught errors to an error monitoring service (Sentry, etc.)
- Log user-facing errors with context: which route, which component, what action triggered it
- Use `console.error` deliberately — never use it for normal flow, only for genuine errors

---

## U-2. Every Request Must Be Traceable

Every operation entering the system must carry a unique identifier that flows through all layers.

### Core rules
- Generate a unique ID at the system entry point if none exists
- Propagate the ID through all downstream calls — logs, external APIs, async operations
- Include the ID in error responses so users can report it

### Backend (Django/Python)
- `CorrelationIDMiddleware` generates a UUID4 on every HTTP request
- Correlation ID stored in `ContextVar` — safe across concurrent async requests
- Passed as `X-Correlation-ID` header to all outbound AI API calls

### Frontend (React/Next.js)
- Pass `X-Correlation-ID` header on all fetch calls if the backend assigned one
- In error states shown to users, surface a reference ID they can report
- Errors logged to monitoring should include the current route and user action

---

## U-3. Modular Boundaries Are Mandatory

A codebase must not become one giant folder with random imports everywhere.

Rules:
- Each module owns its domain — no reaching into another module's internals
- Expose interfaces — never internal implementation details
- Minimise circular dependencies
- Business logic in one module must not directly access another module's data layer

### Backend (Django)
```
accounts/ → sessions/ → questions/ → leaderboard/
```
Access goes through service interfaces, not direct model imports.

### Frontend (React/Next.js)
```
features/session/ → features/auth/ → features/leaderboard/
```
Features access each other through exported hooks or context — never through internal component imports.

---

## U-4. The UI Layer Orchestrates — Logic Lives Below It

The rendering layer (view, component) is responsible for one thing: coordinating calls and presenting results. All logic belongs in a dedicated layer beneath it.

> This single principle improves testability more than any other. Logic in a dedicated layer can be tested without a browser or an HTTP client.

### Backend (Django)

```
View → Service → Repository → Model
```

- **View** — parse request, call service, return response. No business logic.
- **Service** — all business logic. No knowledge of HTTP. Accepts plain Python objects, never `request`.
- **Repository** — all DB queries. No business logic.
- **Model** — schema only.

**Bad:** `views.py` with 500 lines of mixed logic
**Good:** Thin view → focused service → clean query layer

### Frontend (React/Next.js)

```
Component → Hook → Utility / Service
```

- **Component** — render UI, handle user events, call hooks. No business logic inline.
- **Hook** — encapsulate stateful logic, data fetching, side effects. Testable in isolation.
- **Utility** — pure functions, formatters, validators. No React dependencies.

**Bad:** A `<SessionPage>` that fetches data, calculates scores, manages timers, and renders everything inline
**Good:** `<SessionPage>` calls `useSession()` hook → hook calls `submitSession()` utility → component only renders

---

## U-5. Abstract Late, Not Early

> Prefer simple and clear implementations first.
> Extract reusable abstractions only after patterns genuinely repeat.

Premature abstraction creates rigid systems. The cost of the wrong abstraction is higher than the cost of duplication.

Rules:
- Duplicate small code once or twice if needed — it is fine
- Abstract only when: behaviour has stabilised, multiple modules truly share the same responsibility, and the abstraction simplifies maintenance rather than just reducing line count
- Prefer composition over inheritance
- Keep interfaces stable once published

---

## U-6. Work Belongs in the Right Layer

Do not perform heavy, blocking, or async work where it is merely convenient. The place that triggers work is rarely the right place to do it.

### Backend (Django)
- User-facing requests must not do heavy synchronous work
- Use background workers (Celery) for: emails, exports, AI processing that is not user-blocking, retries
- Background jobs must be **idempotent** — safe to run more than once
- Job failures must be observable — logged, alerted, tracked

### Frontend (React/Next.js)
- `useEffect` is for synchronising with **external systems** only: browser APIs, event subscriptions, non-React timers
- `useEffect` is NOT for: transforming data, orchestrating component logic, reacting to state changes that could be handled in event handlers
- If the effect doesn't reference a browser API or subscription, it probably shouldn't be an effect

**Bad:**
```tsx
useEffect(() => {
  setFormattedScore(score / 10 * 100)  // derived state, not external sync
}, [score])
```
**Good:**
```tsx
const formattedScore = Math.round(score / 10 * 100)  // derived inline, no effect needed
```

---

## U-7. AI Calls Are External I/O — Treat Them That Way

LLM API calls must be treated with the same discipline as any external network call.

Rules (apply to both backend calling AI APIs and any future frontend AI integration):
- Always set timeouts — never let an AI call block indefinitely
- Implement retries with exponential backoff
- Define graceful fallback behaviour for when the API is unavailable
- Track cost per call — AI API costs can spiral silently
- Log every AI call: parameters, response time, token usage, errors
- Never expose raw LLM errors to end users
- Validate the response structurally before trusting it

---

## U-8. Configuration Must Be Externalised

Never hardcode secrets, URLs, or environment-specific configuration.

Rules:
- Use environment variables for all configuration
- Use a secrets manager for sensitive values in production
- Validate required config at startup — fail loudly if anything is missing
- Development ≠ Staging ≠ Production — never share config between them

---

## U-9. Security Is a Default Responsibility

Security is not an afterthought. It is built in from the start.

### Universal rules
- Validate all inputs at the system boundary
- Sanitise all outputs before rendering
- Enforce RBAC — principle of least privilege
- Rate limit public-facing endpoints
- Audit authentication and authorisation flows
- Internal tools also require security — most breaches happen internally

### Backend additions
- Set secure HTTP headers (HSTS, X-Frame-Options, X-Content-Type-Options)
- Rotate secrets on a schedule
- Never log passwords, tokens, or sensitive PII

### Frontend additions
- **Never store auth tokens in localStorage or sessionStorage** — they are accessible to any JS running on the page. Use httpOnly cookies for refresh tokens; keep access tokens in memory (Zustand) only.
- **Never inject raw HTML without sanitization** — use a sanitizer library (DOMPurify) before rendering any raw HTML content. React's XSS protection does not apply to raw HTML injection via the innerHTML escape hatch.
- Client-side validation is UX — server-side validation is security. Never rely on client validation alone.
- Configure Content Security Policy (CSP) headers to restrict what scripts can execute.

---

## U-10. Testing Strategy Matters More Than Test Count

Do not chase coverage percentage blindly. Test what gives confidence, not what is easy to test.

### The testing hierarchy (both stacks)

| Type | Confidence | Speed | Write when |
|---|---|---|---|
| Static analysis (types, linters) | Low but free | Instant | Always |
| Unit tests | Low | Fast | Isolated pure functions |
| Integration tests | **High** | Medium | **Most tests should be here** |
| E2E tests | Highest | Slow | Critical user journeys only |

> Integration tests give the best return: they test realistic behaviour without the overhead of a full browser or HTTP stack.

### Universal rules
- Test behaviour, not implementation — tests should survive refactors that don't change behaviour
- Tests that break when you rename an internal variable are testing the wrong thing
- A feature that passes its own tests but breaks an existing flow is a broken feature
- Write flow tests before writing features — not after

### Backend (Django/Python)
- Use `pytest-asyncio` for async test coverage
- Test services independently — services accept plain Python, no `request` needed
- Integration tests should cover full session flows end-to-end

### Frontend (React/Next.js)
- Query by role, label, and text — how users actually interact — not by test IDs or internal state
- **No snapshot tests** — they break on every style change and provide false confidence
- Test error states and edge cases — happy paths are usually obvious
- Mock external APIs, not internal logic

### Flow integrity tests are more valuable than feature tests

> The agent implements a feature. It passes its own tests. But it silently breaks an existing flow that shared the same code path.

Rules:
- Write end-to-end flow tests for every core user journey before building features
- Flow tests run on every change, not just when the related feature is touched
- Verify all affected flows still pass after every implementation

**Core flows — these must always pass (project-specific example):**

```
Flow 1 — Hardcoded session:
  User registers → logs in → starts hardcoded session
  → answers all 10 questions → submits → sees result with explanations

Flow 2 — AI-generated session:
  User registers → logs in → starts AI session
  → answers all 10 questions → submits → sees result with explanations

Flow 3 — Domain Expert question creation:
  Super Admin invites Domain Expert → Expert accepts invite
  → Expert logs into admin → creates question
  → question appears in hardcoded session pool

Flow 4 — Public profile and leaderboard:
  User completes sessions → leaderboard updates
  → public profile reflects updated stats and strong topics
```

These flows are system contracts. Any feature that passes its own tests but breaks a flow is a broken feature.

---

## U-11. Backward Compatibility First

Especially for APIs and interfaces consumed by external clients.

Rules:
- Never break clients silently
- Deprecate gradually — give consumers time to migrate
- Version APIs if breaking changes are unavoidable
- Database schema changes should support rolling deployments

---

## U-12. Documentation Is Engineering

Important systems must document:
- Architecture decisions (ADRs) — why, not just what
- Data flow and async workflows
- Deployment and rollback process
- Incident handling playbook

> Future-you is also a developer on the team.

---

## U-13. Monolith First Is Usually the Correct Decision

People split into microservices too early. A well-designed modular monolith can scale very far.

Split into services only when:
- Team scaling genuinely demands independent deployment
- Scaling characteristics differ significantly between components
- Ownership boundaries are stable and well understood

Not because microservices are modern or fashionable.

---

## U-14. Optimise for Change, Not Cleverness

Good architecture is boring, predictable, understandable by someone new, and easy to modify safely. The measure of good engineering is not how impressive the system looks — it is how confidently you can change it six months later.

---

## U-15. No Implicit Fallbacks — Fail Loudly

Implicit fallbacks make a system appear to work while silently misbehaving in production.

### Backend (Python)
```python
# Bad
api_key = os.getenv('OPENAI_API_KEY') or 'hardcoded-default'

# Good
api_key = os.environ['OPENAI_API_KEY']  # KeyError if missing — intentional
```

### Frontend (TypeScript)
```typescript
// Bad
const apiUrl = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:8027'

// Good — fail at startup if not configured
if (!process.env.NEXT_PUBLIC_API_URL) throw new Error('NEXT_PUBLIC_API_URL is required')
const apiUrl = process.env.NEXT_PUBLIC_API_URL
```

Rules:
- If a value is required, its absence must raise an error — not silently fall back
- Fallbacks are only acceptable when the fallback is a **deliberate, documented product decision**
- Required config must be validated at startup — not lazily at first use

---

## U-16. Explicit Error Handling — No Silent Failures

Every error must be explicitly handled. Silent failure is never acceptable.

```python
# Bad (Python)
try:
    result = call_api(params)
except Exception:
    pass  # never

# Good (Python)
try:
    result = call_api(params)
except httpx.TimeoutException as e:
    logger.error('API timeout', extra={'params': params, 'error': str(e)})
    raise APITimeoutError('Request timed out') from e
```

```typescript
// Bad (TypeScript)
try {
  await submitSession(answers)
} catch { /* silently ignore */ }

// Good (TypeScript)
try {
  await submitSession(answers)
} catch (error) {
  logger.error('Session submit failed', { error, sessionId })
  setError('Failed to submit. Please try again.')
}
```

Rules:
- Every catch block must re-raise, log with full context, or handle with a documented reason
- Never return `null`/`undefined` from an error handler without the caller explicitly expecting it
- Use custom exception/error classes for domain errors — do not leak library errors to callers

---

## U-17. Type Checking and Input Validation at Every Boundary

Every piece of data entering the system must be validated and typed before use.

### Backend (Python/Django)
- Python type hints on all functions, method signatures, and return types
- Use DRF serialisers for API input validation — never access `request.data` directly in views
- Validate AI API responses structurally before saving anything to the DB
- Use Pydantic for runtime validation of complex data structures — it is the Python equivalent of Zod

### Frontend (TypeScript/React)
- TypeScript strict mode enabled — no `any`, no `@ts-ignore` without written justification
- **`any` is banned** — use `unknown` when the type is uncertain, then narrow it
- **Runtime validation is separate from TypeScript types** — TypeScript is compile-time only
- Prefer explicit interfaces over `React.FC` — `React.FC` implicitly adds `children` to all components, creating type confusion when children are not expected

```typescript
// Bad — TypeScript says safe but runtime could be malformed
const session = await fetch('/api/sessions/start/').then(r => r.json()) as Session

// Good — runtime validation via Zod
const session = sessionSchema.parse(await fetch('/api/sessions/start/').then(r => r.json()))
```

If parsing fails: log the raw response, surface an appropriate error state, do not let malformed data propagate into the application.

---

## U-18. Dependency Direction Is Strict

Outer layers depend on inner layers — never the reverse.

### Backend (Django)
```
views → services → repositories → models
```
Views import from services. Services import from repositories. Repositories import from models. Models import from nothing.

### Frontend (React/Next.js)
```
pages → components → hooks → utilities
```
Pages import from components and hooks. Components import from hooks and utilities. Hooks import from utilities and external libraries. Utilities are pure — no React dependencies, no component imports.

Circular imports in either stack are a symptom of violated dependency direction. Fix the architecture — do not work around it with lazy imports.

---

## U-19. Performance Is a Feature — Measure Before Optimising

Performance problems that are not measured are not solved — they are guessed at. Optimise based on data, not intuition.

> Premature optimisation is the root of all evil. But ignoring performance until it hurts users is negligence.

### Universal rules
- Establish a performance baseline before any optimisation work
- Optimise the slowest thing first — profiling reveals what actually matters
- Cache at the right layer, not everywhere as a reflex

### Backend (Django/Python)
- N+1 queries are the most common backend performance killer — audit with Django Debug Toolbar or query logging
- Add database indexes for every column used in `WHERE`, `ORDER BY`, or `JOIN` — measure the impact
- Use `select_related` and `prefetch_related` aggressively on any queryset that touches related models
- Background workers (Celery) for work that does not need to block the response
- Cache expensive aggregations (leaderboards, stats) — define TTL explicitly, not indefinitely

### Frontend (React/Next.js)
- Core Web Vitals are the performance contract with users: LCP ≤2.5s, INP ≤200ms, CLS ≤0.1
- Measure on real devices and real networks — a Lighthouse 100 on a fast laptop is not real-world
- Reduce client JavaScript first — Server Components eliminate entire dependency trees from the bundle
- Never use `<img>` — always use Next.js `<Image>` for automatic format selection, sizing, and lazy loading
- Code-split aggressively — dynamic imports for routes, modals, and features not needed on initial load
- Avoid render-blocking resources — fonts via `next/font`, styles via Tailwind (no runtime CSS-in-JS)

---

# BACKEND PRINCIPLES

*Specific to Django / Python / database-backed systems.*

---

## B-1. Database Discipline Is Architecture

The database schema is not "backend stuff." It is part of the system architecture.

- Avoid N+1 queries — use `select_related`, `prefetch_related`
- Wrap related writes in atomic blocks — use transactions properly
- Review critical migrations before running in production — never blindly
- Indexes are a backend engineering responsibility — add them with migrations, not as an afterthought
- **Soft delete is rarely the right default for domain objects.** Use it only when business rules explicitly require it (user-facing trash features, legal hold requirements). The costs are real:
  - Every query must filter `deleted_at IS NULL` — easy to forget, hard to audit
  - Unique constraints silently break (two "deleted" records with the same email)
  - Gives a false sense of data safety without the guarantees of a proper audit trail
  - Prefer an explicit audit log or event log for recoverability
- Audit important data changes explicitly

---

## B-2. Async Django Requires Explicit Discipline

Django's async support is additive, not native. The ORM, middleware, and view layer were built synchronously; async support is layered on top. This means async discipline cannot be assumed — it must be explicitly enforced at every layer. A single sync call in the wrong place blocks the entire event loop.

- All ORM calls in async context use `a`-prefix methods: `aget`, `acreate`, `asave`, `afilter`
- Transactions (`atomic`) do not work natively in async — wrap in `sync_to_async`
- All API views use `adrf` — never plain DRF views in async context
- All middleware must be async-compatible — use `@sync_and_async_middleware` with both sync and async paths defined explicitly
- `httpx.AsyncClient` initialised once at lifespan startup — never instantiated per request
- Never use the `requests` library in async views — it blocks the event loop

---

# FRONTEND PRINCIPLES

*Specific to React / Next.js / TypeScript.*

---

## F-1. Server Components Are Default — `'use client'` Is the Exception

In Next.js App Router, all components are Server Components by default. Mark only what truly needs the client with `'use client'`.

A component needs `'use client'` only if it uses:
- Hooks (`useState`, `useEffect`, `useRef`, etc.)
- Event handlers that fire in the browser
- Browser APIs (`window`, `document`, `localStorage`)

Push the `'use client'` boundary as deep as possible. A page can be a Server Component even if it renders Client Components — only the interactive parts need the client directive.

> The JavaScript you don't ship is the fastest JavaScript.

---

## F-2. Hydration Must Be Deterministic

Server and client must render identical HTML on first render. Hydration mismatches cause subtle bugs — treat them as zero-tolerance errors.

- Never access `window`, `document`, `navigator`, or `localStorage` during initial render
- Never use `Math.random()`, `Date.now()`, or any non-deterministic value in initial render
- Use `useEffect` for browser-specific code — it runs only after hydration
- `suppressHydrationWarning` is a last resort, not a first response

---

## F-3. Classify State Before Choosing a Tool

Not all state is created equal. Using the wrong tool creates unnecessary complexity.

| State type | Right tool |
|---|---|
| Local UI state (toggle, input value) | `useState` |
| Server data (API responses, caching) | TanStack Query |
| Global client state (auth token, user) | Zustand |
| URL-derived state (filters, pagination) | `useSearchParams` / route params |
| Derived values | Compute inline — not state at all |

> Most "state management problems" are actually server state problems. Solve them with TanStack Query before reaching for Zustand.

---

## F-4. Never Mirror Server Data Into Client State

Do not copy API responses into Zustand or `useState`. TanStack Query is the cache for server data — there is no need to duplicate it.

Two copies of the same data → two opportunities for inconsistency → race conditions and stale UI.

Only put data in Zustand when it is genuinely client-owned (auth token, theme preference, UI state that persists across routes).

---

## F-5. Invalidate Cache on Every Mutation

After any write operation (POST, PUT, PATCH, DELETE), explicitly invalidate the relevant TanStack Query cache keys.

```typescript
const { mutate } = useMutation({
  mutationFn: submitSession,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['sessions', 'history'] })
    queryClient.invalidateQueries({ queryKey: ['leaderboard'] })
  },
})
```

Stale data appearing after a successful mutation is a broken experience.

---

## F-6. QueryKeys Must Be Semantic Stable Arrays

Use arrays for query keys, not strings. Arrays enable precise cache invalidation.

```typescript
// Bad
useQuery({ queryKey: 'sessions-history', ... })

// Good
useQuery({ queryKey: ['sessions', 'history', userId], ... })
```

Centralise queryKey definitions in `src/lib/query-keys.ts` to avoid inconsistency across components.

---

## F-7. Error Boundaries Must Be Granular

Do not wrap the entire app in a single error boundary. Wrap risky sections individually.

A broken leaderboard widget should not take down the entire dashboard. In Next.js: use `error.tsx` files at the route level. In components: use `ErrorBoundary` around feature sections, external widgets, and data visualisations.

---

## F-8. Runtime Validation Is Separate From TypeScript Types

TypeScript catches compile-time errors. It cannot validate data arriving at runtime — API responses, user input, URL params.

Use Zod to validate at every external boundary:

```typescript
// TypeScript alone is not enough — runtime data can be malformed
const data = await fetch('/api/leaderboard').then(r => r.json()) as Leaderboard

// Zod validates at runtime — catches malformed responses
const data = leaderboardSchema.parse(await fetch('/api/leaderboard').then(r => r.json()))
```

If parsing fails: log the raw data, surface an appropriate error state, do not let malformed data propagate.

---

## F-9. Semantic HTML Before ARIA

Always prefer semantic HTML elements over generic divs with ARIA attributes.

- `<button>` not `<div role="button">`
- `<nav>` not `<div role="navigation">`
- `<main>`, `<header>`, `<footer>`, `<section>`, `<article>` for layout
- ARIA supplements semantics — it does not replace them

A `<button>` is inherently keyboard-focusable, activatable with Enter/Space, and understood by screen readers. A styled `<div>` is none of these by default.

---

## F-10. Respect `prefers-reduced-motion`

Users with vestibular disorders can experience nausea and disorientation from animations. This is two lines of CSS — there is no excuse for omitting it.

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## F-11. Design Tokens Are the Single Source of Truth

Colors, spacing, font sizes, and border radii must live in `tailwind.config.ts` — never hard-coded in components.

```tsx
// Bad
<div style={{ color: '#e8540a' }}>

// Good
<div className="text-accent">
```

When a brand color changes, it changes in one place. Components never contain raw hex values.

---

## F-12. Composition Over Props Explosion

A component with many props is a component that needs to be broken up or rethought.

- Pass JSX as `children` instead of `title`, `icon`, `header`, `footer` props
- Use compound components (`<Select>` + `<Select.Option>`) for complex interactive components
- Render props for inversion of control when children need access to parent state
- If a component accepts more than 5-6 props, question whether it is doing too much

---

## Summary

### Universal Principles

| # | Principle |
|---|---|
| U-1 | Observability by default |
| U-2 | Every request must be traceable |
| U-3 | Modular boundaries are mandatory |
| U-4 | The UI layer orchestrates — logic lives below it |
| U-5 | Abstract late, not early |
| U-6 | Work belongs in the right layer |
| U-7 | AI calls are external I/O — treat them that way |
| U-8 | Configuration must be externalised |
| U-9 | Security is a default responsibility |
| U-10 | Testing strategy matters more than test count |
| U-11 | Backward compatibility first |
| U-12 | Documentation is engineering |
| U-13 | Monolith first is usually the correct decision |
| U-14 | Optimise for change, not cleverness |
| U-15 | No implicit fallbacks — fail loudly |
| U-16 | Explicit error handling — no silent failures |
| U-17 | Type checking and input validation at every boundary |
| U-18 | Dependency direction is strict |
| U-19 | Performance is a feature — measure before optimising |

### Backend Principles (Django / Python)

| # | Principle |
|---|---|
| B-1 | Database discipline is architecture |
| B-2 | Async Django requires explicit discipline |

### Frontend Principles (React / Next.js / TypeScript)

| # | Principle |
|---|---|
| F-1 | Server Components are default — `'use client'` is the exception |
| F-2 | Hydration must be deterministic |
| F-3 | Classify state before choosing a tool |
| F-4 | Never mirror server data into client state |
| F-5 | Invalidate cache on every mutation |
| F-6 | QueryKeys must be semantic stable arrays |
| F-7 | Error boundaries must be granular |
| F-8 | Runtime validation is separate from TypeScript types |
| F-9 | Semantic HTML before ARIA |
| F-10 | Respect `prefers-reduced-motion` |
| F-11 | Design tokens are the single source of truth |
| F-12 | Composition over props explosion |

---

*This is a living document. Principles are added or refined as experience grows.*
