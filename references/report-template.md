# QA Report Template

Generate this file as `qa-report.md` in the project root after all review phases.

```markdown
# QA Report — {App Name}

**Date:** {YYYY-MM-DD}
**Reviewed by:** QA Orchestrator (Antigravity)
**Stack:** {e.g., Next.js 15, React 19, Prisma 6, PostgreSQL, NextAuth, Tailwind}
**Scope:** {Full review / Incremental (diff) / Targeted (phases X, Y)}

---

## Executive Summary

{2-3 sentences: overall health, biggest risks, top priority actions}

| Severity | Count |
|----------|-------|
| 🔴 Critical | {n} |
| 🟠 High | {n} |
| 🟡 Medium | {n} |
| 🔵 Low | {n} |

---

## Environment Configuration

| File | Environment | Database/Service | Project ID | Status |
|------|------------|-----------------|------------|--------|
| {.env} | {development} | {Supabase} | {abc123} | {active/paused/missing} |
| {.env.test} | {E2E tests} | {Supabase} | {abc123} | {active/paused/missing} |
| {.env.production} | {production} | {Supabase} | {xyz789} | {active} |

**Notes:** {e.g., "Dev and test share same DB", "No .env.example exists", "AGENTS.md missing environment docs"}

---

## 🔴 Critical Findings

### C1: {Title}
- **Phase:** {phase number and name}
- **File:** `{path/to/file.ts}:{line}`
- **Description:** {What's wrong}
- **Business Impact:** {Why this matters to users/revenue/security}
- **Fix:**
```{lang}
// Before
{problematic code}

// After
{fixed code}
```
- **Status:** 🔲 Open / ✅ Fixed in commit {hash}

{Repeat for each Critical}

---

## 🟠 High Findings

### H1: {Title}
- **Phase:** {phase number and name}
- **File:** `{path/to/file.ts}:{line}`
- **Description:** {What's wrong}
- **Business Impact:** {Why this matters}
- **Fix:**
```{lang}
{code fix}
```
- **Status:** 🔲 Open

{Repeat for each High}

---

## 🟡 Medium Findings

### M1: {Title}
- **Phase:** {phase number and name}
- **File:** `{path/to/file.ts}:{line}`
- **Description:** {What's wrong}
- **Business Impact:** {Why this matters}
- **Fix:** {description or code}
- **Status:** 🔲 Open

{Repeat}

---

## 🔵 Low Findings

### L1: {Title}
- **Phase:** {phase}
- **File:** `{path}`
- **Description:** {What's wrong}
- **Fix:** {suggestion}

{Repeat}

---

## Business Logic Concerns

Issues that aren't bugs but represent missing or incomplete business rules.

| # | Concern | Entities Affected | Risk | Recommendation |
|---|---------|-------------------|------|----------------|
| B1 | {e.g., No cancellation policy enforced} | {Booking, Payment} | {What could go wrong} | {What to implement} |
| B2 | ... | ... | ... | ... |

---

## Missing Functionality

Features implied by the domain but not implemented.

| # | Feature | Why It's Needed | Priority | Effort |
|---|---------|-----------------|----------|--------|
| F1 | {e.g., Waitlist system} | {Capacity-full bookings lost} | {High/Medium/Low} | {S/M/L} |
| F2 | ... | ... | ... | ... |

---

## State Machine Analysis

For each entity with status/state:

### {Entity Name} States
```
{state diagram in ASCII or mermaid}
e.g.:
reservada → confirmada → completada
    ↓           ↓
 cancelada   cancelada
```

**Issues found:**
- {e.g., No transition validation in API — any state can jump to any other}
- {e.g., Missing "no-show" state}

---

## Route Protection Audit

| Route | Type | Auth Required? | Role Check? | Data Scoped? | Status |
|-------|------|----------------|-------------|--------------|--------|
| `/api/v1/clases` | API GET | {Yes/No} | {Yes/No} | {Yes/No/N/A} | {✅/❌} |
| `/api/v1/pagos` | API POST | {Yes/No} | {Yes/No} | {Yes/No/N/A} | {✅/❌} |
| `/dashboard` | PAGE | {Yes/No} | {Yes/No} | N/A | {✅/❌} |
| `createBooking()` | SERVER ACTION | {Yes/No} | {Yes/No} | {Yes/No} | {✅/❌} |

**Unprotected routes:** {list any routes without auth that should have it}
**Unprotected Server Actions:** {list any "use server" functions without auth inside the function body}
**Middleware config issues:** {e.g., "matcher excludes /api/admin/* — intentional?"}
**Middleware bypass tested:** {Yes/No — trailing slashes, URL encoding, locale prefixes, /_next/ abuse}

---

## BaaS / Platform Security

| Check | Status | Details |
|-------|--------|---------|
| RLS enabled on all public tables | {✅/❌} | {list tables without RLS} |
| Direct PostgREST attack tested | {✅/❌} | {which tables returned data with anon key?} |
| Storage buckets properly configured | {✅/❌/N/A} | {public vs private, policies} |
| Auth redirect URLs restricted | {✅/❌} | {allowed URLs} |
| service_role key server-only | {✅/❌} | {where it's used} |
| REST API exposure | {✅/❌} | {what's accessible via anon key} |

---

## Webhook & Integration Security

| Check | Status | Details |
|-------|--------|---------|
| Incoming webhook signatures verified | {✅/❌/N/A} | {which providers, how verified} |
| Retry logic is idempotent | {✅/❌/N/A} | {can retries cause duplicate actions?} |
| External call timeouts configured | {✅/❌} | {default vs explicit} |
| Third-party SDKs up to date | {✅/❌} | {outdated packages} |

---

## Data Privacy & PII

| Check | Status | Details |
|-------|--------|---------|
| PII inventory documented | {✅/❌} | {what personal data is stored} |
| Account deletion possible | {✅/❌} | {full delete or just deactivate?} |
| Data export possible | {✅/❌} | |
| Logs contain PII? | {✅/❌} | {email, IP, tokens in logs?} |
| Third-party data sharing | {✅/❌} | {analytics, error tracking — anonymized?} |
| Cookie consent | {✅/❌/N/A} | |

---

## Backup & Disaster Recovery

| Check | Status | Details |
|-------|--------|---------|
| Automated backups | {✅/❌} | {frequency, retention} |
| Restore tested | {✅/❌} | {last test date or "never"} |
| RPO (max data loss) | {value} | {e.g., "24h on free tier"} |
| RTO (max downtime) | {value} | {e.g., "unknown — no runbook"} |
| Runbook documented | {✅/❌} | |

---

## Test Coverage Analysis

| Area | Current Coverage | Critical Gap? | Suggested Tests |
|------|-----------------|---------------|-----------------|
| Auth flows | {None / Partial / Good} | {Yes/No} | {What to add} |
| Core business logic | ... | ... | ... |
| API endpoints | ... | ... | ... |
| UI components | ... | ... | ... |
| E2E flows | ... | ... | ... |

---

## Dead Code & Dependency Report

### Unused Code
| Type | Count | Details |
|------|-------|---------|
| Unused files | {n} | {list or "see appendix"} |
| Unused exports | {n} | ... |
| Unused dependencies | {n} | {package names} |
| Commented-out code | {n} blocks | {locations} |

### Dependency Health
| Package | Issue | Severity | Action |
|---------|-------|----------|--------|
| {name} | {CVE / abandoned / outdated} | {Critical/High/Med/Low} | {Update to X / Replace with Y / Remove} |

---

## Architecture Recommendations

{Numbered list of structural improvements, each with:}

1. **{Title}** — {Description}. 
   *Why:* {Justification}. 
   *How:* {High-level approach}. 
   *Effort:* {S/M/L}

---

## Action Plan

Priority order for addressing all findings:

| # | Priority | Action | Findings | Effort | Impact |
|---|----------|--------|----------|--------|--------|
| 1 | 🔴 Now | {e.g., Fix JWT secret fallback} | C1 | S | Security |
| 2 | 🔴 Now | {e.g., Add rate limiting} | C3, C4 | M | Security |
| 3 | 🟠 This week | ... | H1, H2 | ... | ... |
| 4 | 🟡 This sprint | ... | M1-M4 | ... | ... |
| 5 | 🔵 Backlog | ... | L1-L7 | ... | ... |

---

## Review Metadata

| Phase | Status | Skills Used | Findings |
|-------|--------|-------------|----------|
| 0. Understand App | ✅ | — | — |
| 0A. Environment Detection | ✅ | — | {n} |
| 1. Code Quality | ✅ | code-review-excellence | {n} |
| 2. Business Logic | ✅ | — | {n} |
| 3. Security (Auth/Authz) | ✅ | code-review-security | {n} |
| 3B. Route & Server Action Audit | ✅ | — | {n} |
| 3E. BaaS/Platform Security | ✅ | — | {n} |
| 3F. Webhook Security | ✅ | — | {n} |
| 3G. Data Privacy | ✅ | — | {n} |
| 4. Database | ✅ | — | {n} |
| 5. Clean Code | ✅ | {skill used} | {n} |
| 6. UI Components | ✅ | webapp-testing, ui-ux-pro-max | {n} |
| 7. UX & A11y | ✅ | {skill used} | {n} |
| 8. Performance & SEO | ✅ | {skill used} | {n} |
| 9. Dead Code & Deps | ✅ | {skill used} | {n} |
| 10. API Contracts | ✅ | — | {n} |
| 11. Tests | {✅/⏭️} | — | {n} |
| 12. Error Handling | {✅/⏭️} | — | {n} |
| 13. i18n | {✅/⏭️} | — | {n} |
| 14. Git, Repo & Docs | {✅/⏭️} | — | {n} |
| 15. CI/CD | {✅/⏭️} | — | {n} |
```

## Notes for the Agent

- ALWAYS generate the ACTUAL file, not just chat output
- Use relative paths from project root in all file references
- Include line numbers when possible
- Code fixes must be copy-pasteable (complete, not snippets with "...")
- If a phase found zero issues, include it in metadata as "0 findings" — don't omit it
- The Action Plan is the MOST IMPORTANT section — users read this first after Executive Summary
- Business Logic Concerns and Missing Functionality are what differentiate this from generic linters
