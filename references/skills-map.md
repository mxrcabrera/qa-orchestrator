# Skills Map — What to Install Per Phase & Stack

## Always Installed (base kit)
These should already be globally installed:
- `vercel-labs/skills@find-skills` — skill discovery
- `wshobson/agents@code-review-excellence` — code quality review
- `hieutrtr/ai1-skills@code-review-security` — security review
- `anthropics/skills@webapp-testing` — webapp testing
- `ui-ux-pro-max` (via uipro-cli) — UI/UX design review

## Per-Phase Skills (install via find-skills when needed)

### Phase 4: Database
```bash
npx skills find "query efficiency N+1 database"
# Best matches:
# - prisma/skills (if using Prisma — covers Client API, query optimization, migrations)
# - levnikolaevich/claude-code-skills (has database audit capabilities)
# Note: most DB checks are inline (phase-review.md 4A-4D). Skills supplement with tooling.
```

### Phase 5: Clean Code
```bash
npx skills find "clean code SOLID refactoring"
# Best matches:
# - ratacat/claude-skills@clean-code
# - ertugrul-dmr/clean-code-skills (boy-scout orchestrator)
# - sickn33/antigravity-awesome-skills@code-refactoring-refactor-clean
# - asyrafhussin/agent-skills@clean-code-principles
```

### Phase 7: Accessibility
```bash
npx skills find "accessibility WCAG audit"
# Best matches:
# - vercel-labs/agent-skills@web-design-guidelines (100+ rules)
# - sickn33/antigravity-awesome-skills@accessibility-compliance-accessibility-audit
```

### Phase 8: Performance & SEO
```bash
npx skills find "lighthouse core web vitals SEO performance"
# Best matches:
# - addyosmani/web-quality-skills (150+ Lighthouse audits, CWV, SEO, a11y)
# - coreyhaines31/marketingskills@seo-audit
# - squirrelscan/skills@audit-website (requires squirrel CLI)
```

### Phase 9: Dead Code
```bash
npx skills find "dead code knip unused"
# Best matches:
# - laurigates/claude-plugins@knip-dead-code (runs knip, depcheck, ts-prune)
```

### Phase 9: Dependency Audit
```bash
npx skills find "dependency audit vulnerabilities CVE"
# Best matches:
# - levnikolaevich/claude-code-skills (ln-625-dependencies-auditor)
```

### Phase 10: API Contracts
```bash
npx skills find "API contract validation endpoint"
# Best matches:
# - api-contract by Damien Laine
```

### Phase 11: Testing
```bash
npx skills find "testing coverage e2e playwright"
# Best matches:
# - wshobson/agents@e2e-testing-patterns (Playwright + Cypress patterns)
# - testdino-hq/playwright-skill (70+ guides)
```

### Phase 17: Refactoring
```bash
npx skills find "refactor clean code improvement"
# Best matches:
# - sickn33/antigravity-awesome-skills@code-refactoring-refactor-clean
# - baggiponte/skills@code-refactor (3-phase: research → proposal → test plan)
```

### Phase 18: Cleanup
```bash
npx skills find "dead code cleanup declutter"
# Best matches:
# - laurigates/claude-plugins@knip-dead-code
# - wildcard/declutter (project structure cleanup)
```

### Phase 3E: BaaS Security (Supabase/Firebase)
```bash
npx skills find "Supabase security RLS policies"
# No dedicated skill exists as of Feb 2025. Use inline checks from phase-review.md.
# The agent should:
#   1. Read prisma/schema.prisma or supabase/migrations/ to know all tables
#   2. Check RLS via Supabase CLI if available: npx supabase db lint
#   3. Check storage policies, auth config, API exposure manually
#   4. If Supabase Management API key is available, query it directly
```

### Phase 3G: Data Privacy
```bash
npx skills find "GDPR data privacy PII audit"
# No mature skill exists. Use inline checks from phase-review.md.
# Key checks: grep for PII in logs, verify deletion endpoints, check third-party data sharing
```

### Phase 14: Documentation
```bash
npx skills find "documentation README changelog API docs"
# Best matches:
# - levnikolaevich/claude-code-skills (documentation pipeline skills)
```

---

## Per-Stack Skills (install when stack is detected)

### Next.js + React
```bash
npx skills find "React Next.js performance best practices"
# - vercel-labs/agent-skills@vercel-react-best-practices (40+ rules, 8 categories)
# - gohypergiant/agent-skills@nextjs-best-practices
```

### Prisma
```bash
npx skills find "Prisma ORM database"
# - prisma/skills (CLI, Client API, migrations, providers, v6→v7 upgrade)
```

### Supabase
```bash
npx skills find "Supabase database auth"
# - supabase/skills (if exists — check find-skills)
# Key: Supabase security is mostly RLS + auth config, not a skills.sh thing.
# The agent handles this inline via phase-review.md Phase 3E.
# For Supabase CLI commands:
#   npx supabase db lint          # Check RLS, schema issues
#   npx supabase inspect db       # Table sizes, index usage
#   npx supabase functions list   # Edge functions deployed
```

### Rails + Inertia
```bash
npx skills find "Inertia Rails best practices"
# - cole-robertson/inertia-rails-skills (50+ rules, forms, auth, testing, SSR)
```

### Tailwind
```bash
npx skills find "Tailwind CSS design system"
# - wshobson/agents@tailwind-design-system (v4, tokens, variants, responsive)
```

### Playwright (for E2E testing)
```bash
npx skills find "Playwright testing"
# - testdino-hq/playwright-skill (70+ guides: core, CI, POM, migration)
```

---

## Anti-patterns: Skills to AVOID

- `zackkorman/skills@security-review` — FLAGGED AS MALICIOUS (hidden curl|bash command)
- Any skill with Snyk "High Risk" AND no Gen "Safe" rating — review source before installing
- Skills with < 100 installs and no verified author — check source code first
- Multiple skills covering the same thing — pick ONE, don't install duplicates

## Skill Budget

Claude Code / Antigravity has a ~15,000 character limit for skill descriptions in system prompt.
Don't install more than ~10-12 skills globally. Use find-skills to install per-project as needed.

To check current budget usage:
```bash
# Count installed skills
ls ~/.agents/skills/ | wc -l
```

If hitting the limit, consider:
1. Uninstalling skills not relevant to current project
2. Using project-scoped installs instead of global
3. Setting `SLASH_COMMAND_TOOL_CHAR_BUDGET=30000` for more headroom
