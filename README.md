# QA Orchestrator

A Claude Code skill that runs autonomous, multi-phase QA reviews on web projects. It reads the entire codebase, audits 20 categories (code quality, security, business logic, database, UI/UX, accessibility, performance, API contracts, tests, CI/CD, cloud infra, containerization, AI integrations), generates a findings report with severity levels, then auto-fixes issues one commit at a time and verifies each fix.

## How to Use

This is a prompt-based skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). To install:

```bash
# Copy the skill directory to your Claude Code skills folder
cp -r . ~/.agents/skills/qa-orchestrator/

# Or install via skills CLI
npx skills add mxrcabrera/qa-orchestrator -g -y
```

Then trigger it in any project with:

| Command | What runs |
|---------|-----------|
| `review` / `qa` | Phases 0-10 + report |
| `full review` / `audit` | All phases 0-15C + report |
| `check security` | Phases 0, 3, 3B, 3C only |
| `test ux` | Phases 0, 6, 7 only |
| `clean up` / `fix issues` | Phases 16-18 (requires existing report) |
| `review changes` | Phase 19 (incremental, git diff only) |

## Phases

### Understand (always runs first)
| # | Phase |
|---|-------|
| 0 | **Understand the App** — reads all config, schema, routes, env files, builds mental model of domain |

### Review (find problems)
| # | Phase |
|---|-------|
| 1 | **Code Quality** — naming, DRY, cyclomatic complexity, dead imports, error handling |
| 2 | **Business Logic** — state machines, edge cases, concurrency, race conditions, data integrity |
| 3 | **Security** — OWASP, auth/authz, injection, secrets, CORS, RLS, data privacy |
| 3B | **AI API Integration** — prompt injection, key exposure, cost limits, output validation |
| 3C | **Secure Dev Methodology** — threat model, input sanitization, dependency supply chain |
| 4 | **Database** — schema quality, indexes, N+1 queries, transactions, migrations |
| 5 | **Clean Code** — SOLID, code smells, function length, coupling |
| 6 | **UI Components** — size, state management, loading/error/empty states, forms |
| 7 | **UX & Accessibility** — WCAG AA, keyboard nav, contrast, responsive, screen reader |
| 8 | **Performance & SEO** — Core Web Vitals, bundle size, images, meta tags, Open Graph |
| 9 | **Dead Code & Dependencies** — unused files/exports/packages, outdated/vulnerable deps |
| 10 | **API Contracts** — response format consistency, error format, status codes, versioning |
| 11 | **Tests** — coverage gaps, test quality, missing E2E, untested critical paths |
| 12 | **Error Handling & Observability** — logging, error recovery, monitoring readiness |
| 13 | **i18n & l10n** — hardcoded strings, date/currency formats, timezone handling |
| 14 | **Git & Documentation** — .gitignore, secrets in history, README, API docs |
| 15 | **CI/CD & DevOps** — pipeline config, linting, type checking, pre-commit hooks |
| 15B | **Cloud Infrastructure** — env vars per environment, IAM, storage, CDN, serverless |
| 15C | **Containerization** — Dockerfile quality, base image pinning, multi-stage builds |

### Fix (after report)
| # | Phase |
|---|-------|
| 16 | **Auto-fix** — fix all findings by severity (Critical > High > Medium > Low), one commit per fix |
| 16B | **Post-fix Verification** — tsc + lint + unit tests + build + E2E |
| 17 | **Refactor** — extract services, simplify functions, apply clean code patterns |
| 18 | **Cleanup** — remove dead code, unused deps, declutter structure |

### Track
| # | Phase |
|---|-------|
| 19 | **Incremental Review** — review git diff only (staged + unstaged) |
| 20 | **Compare Reports** — diff current vs previous qa-report.md |

## Report Output

After review phases, generates `qa-report.md` in the project root with:

- Every finding has: ID, severity, file:line, description, business impact, fix with code
- Severity levels: Critical (exploitable/data loss), High (bug in prod), Medium (tech debt), Low (improvement)
- Sections for: business logic concerns, missing functionality, test coverage gaps
- Action plan table with priority, effort estimate (S/M/L), and impact

See `references/report-template.md` for the full template.

## Key Rules

1. Runs autonomously — no permission prompts between phases
2. Never skips a phase — writes "N/A" if clean
3. Understands the codebase before flagging issues
4. Every finding includes file path, line number, and concrete fix
5. Every finding explains business impact (why it matters to users/revenue)
6. Fixes are atomic — one finding per commit, no bundled changes
7. Post-fix verification is mandatory — tsc, lint, tests, build, E2E
8. Honest about test coverage — if no E2E exists, says so explicitly

## File Structure

```
README.md                          # This file
qa-orchestrator-docs.md            # Extended documentation
references/
  phase-review.md                  # Business logic review instructions
  phase-fix.md                     # Fix phase instructions
  report-template.md               # Report format template
  skills-map.md                    # Skill search queries per tech stack
```

## License

MIT
