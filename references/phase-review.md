# Review Phases — Detailed Instructions

## Phase 1: Code Quality
Use **code-review-excellence** skill. Scan ENTIRE codebase.

### 1A Architecture & Structure
- Files in right directories? Consistent organization?
- Business logic in routes vs extracted to services/utils?
- Code duplication across files (copy-paste detection)
- Circular dependencies between modules
- Dead code: unused imports, exports, variables, functions
- TODO/FIXME/HACK comments — list them all with context
- Barrel files (index.ts re-exports): are they causing circular deps or bundle bloat?

### 1B Complexity Metrics
- **Cyclomatic complexity per function:** flag functions with CC > 10 (nested ifs, switch cases, ternary chains)
- **Cognitive complexity:** flag functions where a human has to hold > 3 mental branches to understand the flow
- **Nesting depth:** flag functions with > 3 levels of nesting (if inside if inside loop inside try)
- **Function length:** flag functions > 50 lines (extract helpers)
- **Parameter count:** flag functions with > 4 parameters (use options object)
- **Return points:** flag functions with > 5 return statements (hard to trace flow)
- **File-level metrics:** flag files with > 10 exported functions (god module, split by responsibility)
- **Chained conditions:** flag `if (a && b || c && !d)` — extract to named boolean or function
- If project has ESLint, check if `complexity` rule is configured. If not, recommend adding it.
- Tools: `npx eslint --rule '{"complexity": ["warn", 10]}' src/` or manual inspection

### 1C Logic & Correctness
- Off-by-one errors, boundary conditions
- Null/undefined: missing guards, unsafe optional chaining
- Type safety: `any` abuse, missing types, unsafe type assertions (`as` casts)
- Async/await: missing awaits, unhandled promise rejections, floating promises
- Error handling: uncaught exceptions, swallowed errors, missing try/catch
- Race conditions in concurrent operations
- Date/time: timezone bugs, DST issues, format inconsistency
- Memory leaks: event listeners not cleaned up, intervals not cleared, subscriptions not unsubscribed

### 1D Performance
- N+1 queries (loops with DB calls inside)
- Missing database indexes on filtered/sorted/joined fields
- Unbounded queries (no LIMIT, no pagination cap)
- Expensive operations in hot paths (request handlers)
- Missing caching where appropriate
- Bundle size: large imports that could be tree-shaken or lazy-loaded
- React: unnecessary re-renders (missing memo, useMemo, useCallback where needed)
- Blocking I/O in request handlers
- Large JSON serialization/deserialization in API responses
- Images served without optimization (no next/image, no WebP, no srcset)

### 1E Testing Status
- What test coverage exists? What frameworks?
- What's tested vs what SHOULD be tested?
- Test quality: testing behavior or implementation details?
- Edge cases covered?
- Integration/E2E tests for critical flows?

---

## Phase 2: Business Logic (MOST IMPORTANT PHASE)

### 2A Data Model Integrity
- Do DB relationships correctly model the business domain?
- Missing constraints: unique, not null, foreign keys, check constraints
- Should-be-enums stored as plain strings
- Can the schema represent INVALID states? (payment without student, negative capacity)
- Soft deletes: consistent or mixed with hard deletes?
- Immutable data that isn't protected (e.g., can someone change a completed payment amount?)
- Derived data stored but never recomputed (stale caches, denormalized counts)

### 2B State Machine Validation
For EVERY entity with a status/state field:
- Map ALL possible states
- Map ALL valid transitions (draw the state machine)
- Verify invalid transitions are BLOCKED in code (not just UI)
- Can the system get STUCK? (entity in limbo state forever)
- Are transitions ATOMIC? (no partial state updates that leave inconsistency)
- Who can trigger each transition? (role-based transition permissions)
- Are transitions LOGGED? (audit trail for state changes)

### 2C Business Rule Enforcement
Check rules are enforced at the RIGHT layer (DB > API > Frontend):
- **Booking/Scheduling**: capacity limits enforced? double-booking prevented? scheduling conflicts detected? cancellation policies enforced? waitlist exists?
- **Payments/Billing**: amounts calculated correctly? currency handled? refund logic exists? subscription periods tracked? overdue handling? partial payments?
- **Users/Auth**: each role can do ONLY what it should? account lifecycle complete? invitation/onboarding flow?
- **Recurring items**: generation logic correct? exceptions handled (holidays, cancellations)? edit-one vs edit-series distinction correct?
- **Rules only in frontend?** Any business rule enforced ONLY in UI but not in API? (critical vulnerability)
- **Race conditions**: two users booking the last spot simultaneously? concurrent payment + cancellation?
- **Idempotency**: can the same action be safely retried? (double-submit, webhook retry)
- **Temporal rules**: rules that depend on time (expiration, grace periods, trial periods) — are they enforced by cron/background job or only on request?

### 2D Edge Cases
- What happens at LIMITS? (0 students, max capacity, 0 payments, empty lists)
- What happens at BOUNDARIES? (end of month, year transition, DST, midnight)
- What happens on DELETE? (cascade behavior, orphaned records, referential integrity)
- What happens on CONCURRENT modification? (optimistic locking? last-write-wins?)
- What happens with UNEXPECTED input that passes Zod/validation? (boundary values, unicode, very long strings)
- What happens when external services fail mid-operation? (payment started but callback never arrives)

### 2E Audit Trail
- Who changed what and when? Is there any tracking?
- Financial operations: is there an immutable log?
- Can an admin silently modify data without trace?
- Is there a way to reconstruct what happened? (event sourcing, audit table, or nothing?)

---

## Phase 3: Security
Use **code-review-security** skill.

### 3A Authentication
- Auth flow complete? (signup, login, logout, forgot password, reset, email verify)
- Token: generation, storage, expiration, rotation, revocation
- Cookies: SameSite, httpOnly, Secure flags
- OAuth: token storage (NOT in JWT/session!), scope, callback URL validation
- Password policy: minimum length, complexity, common password check
- Brute force protection: account lockout or rate limiting on login
- Multi-factor authentication available? (if handling sensitive data)
- Session invalidation on password change

### 3B Authorization & Route Protection
**Systematic route audit (MANDATORY):**
1. List ALL API routes:
   ```bash
   # Next.js
   find app/api -name "route.ts" -o -name "route.js" | sort
   # Express/Koa
   grep -rn "router\.\(get\|post\|put\|patch\|delete\)" --include="*.ts" --include="*.js"
   # Rails
   rails routes 2>/dev/null || cat config/routes.rb
   ```
2. List ALL page routes:
   ```bash
   # Next.js App Router
   find app -name "page.tsx" -o -name "page.js" | sort
   ```
3. List ALL Server Actions (Next.js 14+):
   ```bash
   # Find all "use server" directives — these are POST endpoints anyone can call directly
   grep -rn '"use server"' --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" | sort
   ```
   For EACH Server Action found:
   - Does it validate the session/auth INSIDE the function? (middleware does NOT protect Server Actions)
   - Does it validate and sanitize ALL parameters? (attackers can call with arbitrary payloads)
   - Is the user authorized for this action? (not just authenticated — role check too)
   - Flag ANY Server Action without auth check as 🔴 Critical — it's a public POST endpoint

4. For EACH route (API + pages + Server Actions), verify:
   - Is authentication required? (middleware, session check, or OPEN?)
   - Is role-based authorization enforced? (admin-only routes protected?)
   - Is data scoped to the authenticated user? (can user A see user B's data?)
5. Check middleware configuration:
   ```bash
   # Next.js: what does the matcher include/exclude?
   cat middleware.ts 2>/dev/null
   # Look for overly broad exclusions in matcher config
   ```
6. **Test middleware bypass vectors (MANDATORY for Next.js):**
   Known patterns that skip middleware:
   - `/_next/` prefix (static files — does matcher exclude this? can it be abused?)
   - Trailing slashes: `/admin` vs `/admin/` — does middleware catch both?
   - URL encoding: `/api/%61dmin` instead of `/api/admin`
   - Locale prefixes: `/en/admin` if i18n is configured — does middleware apply?
   - Direct fetch to Server Action endpoint (POST with `Next-Action` header)
   If middleware is the ONLY auth layer (no per-route checks), any bypass = full access. Flag as 🔴 Critical.
7. Flag: ANY route that handles user data but has NO auth check

**Authorization checks:**
- RBAC enforcement on EVERY route AND action
- Horizontal escalation: can user A see/edit user B's data?
- Vertical escalation: can student access admin features?
- Every DB query scoped to authenticated user's permissions?
- IDOR: sequential/guessable IDs in URLs/params exploitable?
- API params: can client supply IDs they shouldn't? (e.g., profesorId, userId)
- Bulk operations: can a user modify resources they don't own via batch endpoints?

### 3C Input, Data & Client-Side Secrets
- OWASP Top 10 full check
- EVERY user input validated server-side (not just frontend Zod)
- HTML/SQL/NoSQL/command injection vectors
- File upload security (type validation, size limits, storage location, filename sanitization)
- Sensitive data in: API responses, logs, error messages, JWT payload, client session

**Client-side secret exposure (CRITICAL):**
- Scan ALL `NEXT_PUBLIC_*` / `VITE_*` / `REACT_APP_*` env vars — are ANY of them secrets that should NOT be public?
- Specifically check: service_role keys, API secrets, database URLs, webhook secrets MUST NOT be in public env vars
- Distinguish: `anon` key (OK in client) vs `service_role` key (NEVER in client)
- Search build output for leaked secrets:
  ```bash
  # Check if any secret values appear in client bundle
  grep -r "service_role\|secret\|password\|private_key" .next/static/ 2>/dev/null
  grep -r "service_role\|secret\|password\|private_key" dist/ 2>/dev/null
  ```
- Hardcoded secrets, API keys, tokens in source code
- Check git history for previously committed secrets:
  ```bash
  git log --all -p -S "service_role" --since="2020-01-01" -- '*.ts' '*.js' '*.env*' 2>/dev/null | head -50
  git log --all -p -S "secret" --since="2020-01-01" -- '*.ts' '*.js' '*.env*' 2>/dev/null | head -50
  ```

### 3D Infrastructure Security
- Dependency CVEs: `npm audit` or `bundler audit`
- CORS configuration: is `Access-Control-Allow-Origin: *` used? Should it be restricted?
- Rate limiting: does it ACTUALLY work? (in-memory on serverless = broken per invocation)
- CSP (Content Security Policy) headers present and restrictive?
- Error responses: do they leak stack traces, internal paths, or DB schemas?
- HTTPS enforced? HTTP → HTTPS redirect?
- Security headers present? (`X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`, `Referrer-Policy`)
- DNS: DNSSEC configured? SPF/DKIM/DMARC for email sending?

### 3E BaaS / Platform Security (Supabase, Firebase, etc.)

**Skip this section if the app doesn't use a BaaS platform.**

**Row Level Security (RLS):**
1. List ALL tables in public schema
2. For EACH table, check:
   - Is RLS enabled? (disabled = anyone with the anon key can read/write everything)
   - What policies exist? (SELECT, INSERT, UPDATE, DELETE)
   - Are policies correct? (do they scope to `auth.uid()`?)
   - Are there tables with RLS enabled but NO policies? (= nobody can access, even authenticated users)
3. Run if Supabase CLI is available:
   ```bash
   # Check RLS status on all tables
   npx supabase db lint 2>/dev/null
   ```
4. **Direct PostgREST attack test (MANDATORY if Supabase):**
   This is the #1 most important BaaS security test. An attacker only needs the anon key (which is public in the client bundle) and the project URL to try this:
   ```bash
   # Extract anon key and URL from .env or client bundle
   SUPABASE_URL="https://{project_id}.supabase.co"
   ANON_KEY="{anon_key_from_env}"

   # Try to READ every table without auth
   curl -s "$SUPABASE_URL/rest/v1/{table_name}?select=*&limit=5" \
     -H "apikey: $ANON_KEY" \
     -H "Authorization: Bearer $ANON_KEY"

   # Try to INSERT into every table without auth
   curl -s -X POST "$SUPABASE_URL/rest/v1/{table_name}" \
     -H "apikey: $ANON_KEY" \
     -H "Authorization: Bearer $ANON_KEY" \
     -H "Content-Type: application/json" \
     -d '{"test": "rls_check"}'

   # Try to DELETE from every table without auth
   curl -s -X DELETE "$SUPABASE_URL/rest/v1/{table_name}?id=eq.99999" \
     -H "apikey: $ANON_KEY" \
     -H "Authorization: Bearer $ANON_KEY"
   ```
   If ANY of these return data or succeed → 🔴 Critical. RLS is missing or broken on that table.
   Expected result: 0 rows returned (RLS blocks), or 401/403 error.
5. Check if app uses Prisma/direct connection (bypasses RLS) vs Supabase SDK (respects RLS):
   - If Prisma with connection string: RLS doesn't protect you from YOUR OWN code bugs, but it DOES protect against direct API attacks
   - If Supabase SDK: RLS is your primary security layer

**Storage Buckets:**
- Are storage buckets public or private?
- File access policies: who can upload? who can read? size limits?
- Can users upload executable files? (.html, .svg with scripts, .js)

**Auth Configuration:**
- Which auth providers are enabled? (email, Google, GitHub, etc.)
- Redirect URLs: are they restricted to your domains? Or can an attacker add `evil.com`?
- Email templates: are they customized or default? (phishing risk with defaults)
- JWT expiry: what's the session duration? Is refresh token rotation enabled?
- Can users sign up freely or is it invite-only? (matches the app's intention?)

**API Exposure:**
- PostgREST/REST API: is it exposed publicly? On which tables?
- Realtime subscriptions: can unauthenticated users subscribe to table changes?
- Edge Functions / Serverless functions: are they protected?
- Service role key: is it ONLY used server-side? NEVER in client code or env vars prefixed with `NEXT_PUBLIC_`?

### 3F Webhook & Integration Security
- Incoming webhooks: is the signature/secret verified? (Stripe, payment providers, etc.)
- Outgoing API calls: are secrets rotated? stored securely?
- Timeout configuration: what if an external service hangs? (default timeouts vs explicit)
- Retry logic: is it idempotent? (retrying a payment webhook shouldn't charge twice)
- Circuit breaker: does the app handle external service outages gracefully?
- Third-party SDKs: are they up to date? Do they have known vulnerabilities?

### 3G Data Privacy & PII
- **PII Inventory**: what personal data is stored? (name, email, phone, address, payment info, health data)
- **Account deletion**: can a user delete their account and ALL associated data? (right to be forgotten)
- **Data export**: can a user request a copy of their data? (GDPR Article 15)
- **Data minimization**: is the app collecting more data than it needs?
- **Encryption**: PII encrypted at rest? (database-level encryption, field-level encryption for sensitive fields)
- **Cookie consent**: if using cookies beyond session management, is there a consent mechanism?
- **Data retention**: is old data automatically purged? Or does it accumulate forever?
- **Third-party data sharing**: is user data sent to analytics, logging, or error tracking services? Is it anonymized?
- **Logs**: do application logs contain PII? (user emails, IPs, session tokens in log files)

---

## Phase 3B: AI API Integration Review

**Skip this section if the project doesn't integrate with any AI API (OpenAI, Anthropic, Google AI, Ollama, Replicate, etc.).**

### 3B-A Prompt Injection & Output Safety
- **Prompt injection:** can user input reach system prompts? (concatenation without sanitization)
- **Indirect prompt injection:** does the AI process external data (web pages, emails, documents) that could contain malicious instructions?
- **Output validation:** is AI output sanitized before rendering? (XSS via AI-generated HTML/markdown)
- **Output used in code execution?** (AI generates SQL, code, shell commands — NEVER execute without sandboxing)
- **Jailbreak resilience:** are system prompts robust? or can a user say "ignore previous instructions"?
- **PII in prompts:** is user data sent to AI APIs? Is it necessary? Can it be anonymized first?

### 3B-B API Key & Cost Security
- **API key exposure:** keys stored in env vars (not client-side)? Never in `NEXT_PUBLIC_*` or `VITE_*`?
- **Key rotation:** is there a process to rotate keys without downtime?
- **Cost runaway protection:** are token limits set per request? (`max_tokens`, `max_completion_tokens`)
- **Per-user rate limiting:** can a single user drain the AI budget? (rate limit on AI endpoints)
- **Budget alerts:** are spending alerts configured on the AI provider dashboard?
- **Model selection:** is the most expensive model used everywhere? Could cheaper models handle simpler tasks?
- **Caching:** are identical/similar prompts cached to avoid redundant API calls?
- **Timeout:** what happens if the AI API hangs? Is there a timeout configured? (default can be 10+ minutes)

### 3B-C Reliability & Error Handling
- **Fallback on AI failure:** what happens when the AI API returns 429/500/timeout? Does the app crash or degrade gracefully?
- **Retry logic:** exponential backoff? Or aggressive retries that burn budget?
- **Streaming:** if using streaming responses, is the stream properly closed on error/abort?
- **Response parsing:** is the AI response parsed safely? (JSON.parse on AI output can throw — wrap in try/catch)
- **Token counting:** are inputs checked against the model's context window limit before sending?
- **Logging:** are AI requests/responses logged for debugging? (without logging sensitive user data)

---

## Phase 3C: Secure Development Methodology

### 3C-A Threat Model & Security Documentation
- **Threat model exists?** Even a simple list of: "what are the attack surfaces? what data is sensitive? who are the threat actors?"
- **Security notes in AGENTS.md or docs/:** are auth flows, permission model, and data sensitivity levels documented?
- **Incident response plan:** what happens if a breach is detected? (at minimum: who to contact, how to revoke keys, how to notify users)
- **Security review cadence:** is there a schedule for reviewing dependencies, access controls, and secrets?

### 3C-B Input Sanitization Beyond Zod
- **Zod validates shape, not content.** A string that passes `z.string().min(1)` can still contain: `<script>`, `'; DROP TABLE`, `{{template injection}}`, `../../../etc/passwd`
- **HTML injection:** is user-provided text rendered with `dangerouslySetInnerHTML` or equivalent? Even in markdown renderers?
- **SQL injection:** any raw SQL queries with string interpolation? (Prisma parameterizes by default, but `$queryRaw` with template literals does NOT)
- **Path traversal:** user-provided filenames used in `fs.readFile()`, `path.join()`, or storage paths?
- **SSRF:** does any endpoint fetch a user-provided URL? (can be used to scan internal network)
- **ReDoS:** complex regex patterns on user input that could cause exponential backtracking?
- **Header injection:** user input in HTTP headers (Location, Set-Cookie)?

### 3C-C Dependency Audit & Supply Chain
- **Automated dependency audit:** is `npm audit` or equivalent run in CI? On every PR?
- **Lock file integrity:** is `package-lock.json` committed? Does CI use `npm ci` (not `npm install`)?
- **Typosquatting check:** any dependencies with suspicious names similar to popular packages?
- **Post-install scripts:** do any dependencies run scripts on install? (`preinstall`, `postinstall` in package.json)
  ```bash
  # Check for install scripts in dependencies
  npm query ':attr(scripts, [postinstall])' 2>/dev/null
  ```
- **Pinned versions:** are critical dependencies pinned to exact versions? Or using `^` / `~` that auto-updates?

### 3C-D Secrets Scanning
- **Pre-commit hook for secrets:** is a tool like `gitleaks`, `detect-secrets`, or `trufflehog` configured?
- **CI secrets scanning:** does the pipeline scan for leaked secrets on every push?
- **Secret rotation history:** have secrets been rotated since the project started? (stale secrets = higher risk)
- **Env var documentation:** are all required secrets documented in `.env.example` with descriptions (not values)?

---

## Phase 4: Database

### 4A Schema Quality
- Appropriate data types (String vs Enum, Int vs BigInt, DateTime precision)
- Indexes on ALL fields used in WHERE, ORDER BY, JOIN
- Missing unique constraints (email, slug, composite keys)
- Missing NOT NULL where data is required
- Foreign key cascade/restrict/set null appropriate per relationship
- Computed vs stored: anything stored that should be derived?
- Composite indexes for multi-column queries
- Partial indexes where applicable (e.g., index only non-deleted rows)

### 4B Query Quality
- N+1: loops containing queries, missing includes/joins/eager loading
- Overfetching: selecting all fields when only a few needed (use `select` in Prisma)
- Pagination: all list endpoints paginated? server-side cap on limit? (max 100, not user-supplied 999999)
- Transactions: multiple related writes wrapped in transaction?
- Raw queries: parameterized? (no string interpolation — SQL injection risk)
- Connection pooling configured for production? (PgBouncer, Prisma connection pool)
- Query timeouts configured? (prevent runaway queries from locking the DB)
- Explain plans for complex queries (are they using indexes or doing sequential scans?)

### 4C Data Consistency
- Can ANY code path leave DB in inconsistent state?
- Orphaned records possible? (delete parent, children remain)
- Timestamps: all UTC? consistent format? `createdAt`/`updatedAt` on all tables?
- Migrations: reversible? safe to run on production? (no data loss, no long locks on large tables)
- Seed data: exists? realistic? covers all roles and states?
- Enum values in sync between code and database?

### 4D Backup & Disaster Recovery
- Are automated backups configured? (Supabase: daily backups on Pro, point-in-time on Team+)
- What's the backup retention period?
- Has a restore been tested? (backup you've never tested restoring = no backup)
- What's the Recovery Point Objective? (how much data can you lose? 24h? 1h? 0?)
- What's the Recovery Time Objective? (how fast can you be back online?)
- Is there a documented runbook for "the database is gone, what do I do?"
- For Supabase free tier: backups are NOT automatic — flag as Critical if production data has no backup

---

## Phase 5: Clean Code
Use find-skills: `"clean code SOLID refactoring"`

- Functions > 30 lines → flag
- Files > 300 lines → flag
- Classes/components > 200 lines → flag
- Functions with > 4 parameters → flag
- Nesting depth > 3 levels → flag
- SOLID violations (especially SRP and DIP)
- God objects/files that do too much
- Commented-out code (delete it, git has history)
- Magic numbers/strings (extract to constants)
- Inconsistent patterns across similar files (e.g., some API routes use try/catch, others don't)
- Copy-paste code (extract to shared utility)
- Naming: misleading names, abbreviations, inconsistent conventions (camelCase vs snake_case mix)
- Default exports vs named exports: consistent?

---

## Phase 6: UI Components
Use **webapp-testing** + **ui-ux-pro-max** skills.

- Component size (flag > 200 lines)
- Prop drilling (flag > 3 levels deep)
- State: local vs global appropriate? unnecessary state? state that should be derived?
- Effects: missing deps? unnecessary effects? potential infinite loops? cleanup functions?
- Error boundaries for critical sections?
- Loading states for EVERY async operation?
- Empty states: what does user see with no data?
- Error states: what happens when API fails? network timeout? 500?
- Forms: validation, error display, submit handling, double-submit prevention, dirty state detection
- Reusability: duplicated UI patterns that should be components?
- Controlled vs uncontrolled inputs: consistent approach?
- Key prop usage in lists: using index as key? (causes bugs on reorder/delete)
- Client-only code that should be server-rendered (or vice versa in RSC)

---

## Phase 7: UX & Accessibility
Use find-skills: `"accessibility WCAG audit"`

### 7A Responsive
- Mobile breakpoints in CSS/Tailwind (375px, 768px, 1024px)
- No horizontal scroll at any viewport
- Touch targets minimum 44x44px
- Font sizes readable on mobile (minimum 16px body)
- Forms usable on mobile (input types, autocomplete attributes)
- Tables: horizontally scrollable or restructured for mobile?
- Modals/dialogs: usable on small screens? Not taller than viewport?

### 7B Accessibility (WCAG AA)
- Color contrast: 4.5:1 text, 3:1 large text, 3:1 UI components
- Keyboard: every interactive element reachable via Tab
- Focus indicators visible (not removed via outline:none)
- ARIA labels on: icons, image buttons, inputs without visible labels
- Form labels associated with inputs (htmlFor/id)
- Dynamic content: aria-live regions for updates (toasts, loading states, errors)
- prefers-reduced-motion respected (no animations for users who opt out)
- prefers-color-scheme respected (if dark mode exists)
- Skip to content link
- Heading hierarchy (h1 → h2 → h3, no skips)
- Focus trap in modals (Tab doesn't escape to background)
- Escape key closes modals/dropdowns

### 7C UX Patterns
- Feedback on every user action (toast/notification after save/delete/error)
- Destructive actions require confirmation dialog
- Progress indicators for multi-step processes
- Error messages meaningful and actionable (not "Something went wrong")
- Consistent navigation across all pages
- 404/error pages exist and are helpful
- Back/breadcrumb navigation where needed
- Unsaved changes warning (beforeunload or route guard)
- Undo for destructive actions where possible (soft delete with restore)

### 7D Browser Testing (if app running locally)
- Open app, test primary user flows
- Test at 375px, 768px, 1024px viewports
- Keyboard-only navigation test
- Form submission with valid + invalid data
- Screenshot issues found

---

## Phase 8: Performance & SEO
Use find-skills: `"lighthouse core web vitals SEO performance"`

### Performance
- Bundle size analysis (large dependencies, tree-shaking opportunities)
- Image optimization (format, sizing, lazy loading, next/image usage)
- Code splitting / lazy loading for routes and heavy components
- Server-side rendering used where appropriate?
- Caching strategy (static assets, API responses, DB queries)
- Core Web Vitals risks: LCP (large images/fonts), INP (heavy JS), CLS (layout shifts)
- Font loading strategy (display: swap? preload? self-hosted vs Google Fonts?)
- Third-party script impact (analytics, chat widgets, ads — do they block rendering?)
- Database query time: are any API routes doing >100ms of DB work?
- Serverless cold start: is it acceptable? Can critical paths be kept warm?

### SEO
- Page titles unique and descriptive per route
- Meta descriptions present
- Open Graph tags (og:title, og:description, og:image)
- Canonical URLs
- Sitemap.xml exists
- Robots.txt appropriate
- Structured data (JSON-LD) where applicable
- Semantic HTML (header, main, nav, article, section)
- Image alt texts present and meaningful
- Dynamic routes: are they pre-rendered/ISR or client-only? (client-only = invisible to Google)
- URL structure: clean, descriptive, no query-string dependent content

---

## Phase 9: Dead Code & Dependencies
Use find-skills: `"dead code knip"` then `"dependency audit CVE"`

### Dead Code
- Unused files (no imports pointing to them)
- Unused exports (exported but never imported)
- Unused dependencies in package.json/Gemfile
- Unused variables and functions
- Commented-out code blocks
- Unreachable code (after return/throw)
- Feature flags / TODO code that's been abandoned
- Stale configuration files (configs for tools no longer used)
- Dead API routes (defined but no frontend calls them)

### Dependencies
- Known CVEs: `npm audit` (report ALL, not just critical)
- Outdated packages (major versions behind)
- Abandoned packages (no updates in 2+ years, archived repo)
- Duplicate packages (same purpose, different library)
- Heavy packages with lighter alternatives (moment.js → date-fns, lodash → native)
- Dev dependencies in production bundle? (`devDependencies` vs `dependencies` correct?)
- Phantom dependencies (used in code but not in package.json — resolved by hoisting)
- Lock file integrity (package-lock.json / yarn.lock / pnpm-lock.yaml committed and up to date?)

---

## Phase 10: API Contracts

- Every endpoint returns consistent response format
- Error responses follow a standard format: `{ error: string, code: string, details?: any }`
- HTTP status codes correct (200 vs 201 vs 204, 400 vs 422, 401 vs 403)
- Pagination format consistent across all list endpoints
- Date format consistent (ISO 8601)
- API versioning strategy (URL path vs header)
- Content-Type validation on all POST/PUT/PATCH
- Response doesn't include unnecessary fields (oversharing — leaking internal IDs, timestamps, or relations the client doesn't need)
- HATEOAS or documented endpoint relationships where useful
- Request body size limits configured? (prevent 100MB POST payloads)
- Response compression enabled? (gzip/brotli)
- Consistent naming convention (camelCase vs snake_case — pick one)
- Null vs undefined vs missing field: consistent convention?

---

## Phase 11: Tests
Use find-skills: `"testing coverage e2e playwright"`

- List all existing test files and what they cover
- Identify UNTESTED critical paths (auth, payments, core business flows)
- Test quality: are they testing behavior or mocking everything?
- Are tests deterministic? (no shared state, no time-dependent, no external service calls)
- E2E tests for main user flows? (signup → login → core action → logout)
- Are tests runnable? (try running them, report failures)
- Suggested test plan: what to add, prioritized by risk
- Test data: are fixtures/factories used? or hardcoded IDs that will break?
- Flaky tests: any tests that pass/fail randomly? (identify and flag)
- Test isolation: does each test clean up after itself? or do tests depend on order?

---

## Phase 12: Error Handling & Observability

- Unhandled exceptions: are there global error handlers? (Next.js: error.tsx, Express: error middleware)
- API errors: are they caught and returned with useful messages?
- Client errors: are error boundaries in place?
- Logging: structured? levels? (info, warn, error)
- Production logging: are error objects serialized (not just message)? stack traces included?
- Monitoring readiness: Sentry/LogRocket/Datadog integration? or at least ready to add?
- Health check endpoint exists?
- Retry logic for external service calls?
- Graceful degradation when external services are down?
- Crash recovery: if the process dies mid-operation, does the system recover? (pending transactions, incomplete writes)
- Alert thresholds: are there conditions that should trigger alerts? (error spike, DB connection pool exhaustion, disk space)

---

## Phase 13: i18n & l10n

- Hardcoded user-facing strings (should be in translation files/constants)
- Date formats: consistent? locale-aware? (not hardcoded "DD/MM/YYYY")
- Currency formats: symbol, decimal separator, thousands separator
- Number formats: locale-appropriate
- Timezone handling: stored in UTC? converted for display? user timezone detected or configured?
- Right-to-left (RTL) readiness (if applicable)
- Pluralization handled correctly
- Concatenated strings (breaks in other languages — use template/interpolation)
- Sorting: locale-aware? (á, ñ, ü sort correctly)

---

## Phase 14: Git, Repo & Documentation Hygiene

### Git
- .gitignore: covers .env, node_modules, build output, IDE files, OS files, .next, dist?
- Secrets in git history? (search for API keys, passwords, tokens in past commits)
- .env.example exists with all required vars documented?
- Branch strategy documented or evident?
- Commit messages: conventional? meaningful?
- Large files in repo that shouldn't be? (images, videos, DB dumps, node_modules committed)
- Lock files committed? (package-lock.json, yarn.lock, pnpm-lock.yaml)

### Documentation
- README.md: exists? setup instructions work? (can a new developer clone and run in <10 min?)
- AGENTS.md: exists? documents environment config, architecture, conventions for AI agents?
- API documentation: exists? (OpenAPI/Swagger, or at minimum a list of endpoints with expected params/responses)
- Inline documentation: complex functions have JSDoc/TSDoc? Business logic explained in comments?
- CHANGELOG: maintained? (at least for breaking changes)
- Architecture Decision Records (ADRs): for non-obvious technical choices?
- Environment setup: documented? Which .env files needed, how to get credentials?

### Knowledge Base & Project Rules
- **RULES.md or equivalent** (CLAUDE.md, .cursorrules, CONVENTIONS.md): does the project have a file defining coding standards, naming conventions, and architectural decisions?
- If missing: **generate one** based on observed patterns (language, framework conventions, import style, test patterns, commit message format, file organization)
- Contents should cover: language & style (tabs vs spaces, quotes, semicolons), naming (camelCase vs snake_case, component naming), file organization (where do utils go? services? types?), git conventions, testing expectations
- Is the rules file consistent with actual code? (rules say "no any" but code has `any` everywhere = stale rules)
- Are there conflicting rules across files? (CLAUDE.md says one thing, .eslintrc says another)

### AI-Assisted Documentation
- **Public functions:** do exported functions in `lib/`, `utils/`, `services/` have JSDoc/TSDoc? Flag public APIs without parameter descriptions
- **API routes:** is there a generated or maintained OpenAPI/Swagger spec? If not, flag as Medium — AI tools and frontend devs need contract docs
- **Complex business logic:** are functions with CC > 10 or multi-step algorithms documented with a brief "why" comment?
- **README completeness:** does it cover: what the app does, how to install, how to run dev/test/build, how to deploy, env vars needed, architecture overview?
- **Auto-generation opportunity:** if project has TypeScript, recommend `typedoc` or similar for auto-generated API docs. If API routes follow a pattern, recommend OpenAPI generation

---

## Phase 15: CI/CD & DevOps

- CI pipeline exists? (GitHub Actions, GitLab CI, etc.)
- Pipeline runs: linting? type checking? tests? build?
- Pre-commit hooks? (husky, lint-staged)
- Deploy config exists and is correct? (Netlify, Vercel, Railway, etc.)
- Environment variables managed properly? (not hardcoded per environment)
- Health checks configured?
- Rollback strategy? (can you revert a bad deploy in <5 minutes?)
- Preview/staging environments? (Netlify deploy previews, Vercel preview URLs)
- Build caching: configured? (speeds up CI significantly)
- Notifications: does the team know when a deploy fails? (Slack, email)
- Deploy protection: is production deploy gated behind tests passing?

---

## Phase 15B: Cloud Infrastructure Review

**Skip if the project is a static site or has no cloud-specific configuration.**

### 15B-A Environment Configuration
- **Env vars per environment:** are dev, staging, and production configs clearly separated?
- **Missing env vars:** compare `.env.example` against actual deployment config — any vars missing in production?
- **Env var validation:** does the app validate required env vars on startup? (fail fast, not silent undefined)
- **Secrets management:** are secrets in a vault (AWS Secrets Manager, Vercel env vars, Doppler) or plain text in CI config?
- **Environment parity:** are dev and production using the same services? (SQLite in dev, Postgres in prod = bugs you won't catch)

### 15B-B IAM & Access Control
- **IAM roles (AWS/GCP/Azure):** are roles scoped to minimum required permissions? Or using admin/full-access?
- **Service accounts:** do serverless functions use dedicated service accounts with minimal scope?
- **API gateway:** if using one, are routes authenticated at the gateway level?
- **Multi-tenancy isolation:** if the app serves multiple tenants, is data isolated at the infrastructure level?

### 15B-C Storage & CDN
- **Storage bucket policies:** are buckets private by default? Public read only where needed?
- **Bucket naming:** no sensitive info in bucket names (visible in URLs)
- **CDN configuration:** cache headers correct? Static assets cached? API responses NOT cached?
- **CDN invalidation:** is there a process to invalidate CDN cache on deploy?
- **CORS on storage:** are storage bucket CORS policies restrictive? (not `*`)
- **Upload size limits:** configured at the infrastructure level? (not just app-level validation)

### 15B-D Serverless & Compute
- **Function timeouts:** are they set appropriately? (not default 3s for DB operations, not 900s for simple queries)
- **Memory allocation:** right-sized? (over-provisioned = waste, under-provisioned = OOM crashes)
- **Cold start mitigation:** provisioned concurrency for critical paths? Keep-warm strategy?
- **Concurrency limits:** configured to prevent runaway scaling and cost spikes?
- **Region selection:** is the compute region close to the database region? (cross-region latency)
- **Scaling limits:** min/max instances configured? (prevent surprise $10k bills)

---

## Phase 15C: Containerization Review

**Skip if the project has no Dockerfile or docker-compose.yml.**

### 15C-A Dockerfile Quality
- **Base image pinning:** using specific tag (e.g., `node:20.11-alpine`) not `latest` or just `node`?
- **Multi-stage build:** is the final image slim? (build stage separate from runtime stage)
- **Layer ordering:** are frequently-changing layers (COPY source code) AFTER stable layers (COPY package.json + npm install)?
- **Non-root user:** does the container run as a non-root user? (`USER node` or similar)
- **Image size:** is the final image reasonably small? (Alpine-based? No dev dependencies in production image?)
  ```bash
  # Check Dockerfile for common issues
  cat Dockerfile 2>/dev/null
  ```

### 15C-B Security
- **No secrets baked into image:** no COPY .env, no ARG with secrets, no hardcoded API keys in Dockerfile
- **Secret injection:** are secrets passed via environment variables at runtime, not build time?
- **.dockerignore present?** Does it exclude: `.env*`, `node_modules`, `.git`, `*.log`, test files?
- **Vulnerability scanning:** is the image scanned for CVEs? (Docker Scout, Trivy, Snyk)
- **Read-only filesystem:** can the container run with `--read-only`? (prevents runtime file tampering)

### 15C-C Runtime & Health
- **HEALTHCHECK defined:** does the Dockerfile or compose file define a health check?
- **Graceful shutdown:** does the app handle SIGTERM? (important for rolling deploys)
- **Logging:** does the app log to stdout/stderr? (not to files inside the container)
- **Restart policy:** is `restart: unless-stopped` or equivalent configured in compose?
- **Resource limits:** are CPU/memory limits set in compose or orchestrator config?

### 15C-D Docker Compose (if present)
- **Service dependencies:** are `depends_on` with health checks used? (not just ordering)
- **Volume mounts:** are volumes used for persistent data? (database data, uploads)
- **Network isolation:** are services on isolated networks? (DB not exposed to host network)
- **Environment file:** does compose reference `.env` files correctly?
- **Production vs development compose:** separate files or profiles for dev vs prod?
