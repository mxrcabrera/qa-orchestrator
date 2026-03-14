# Fix, Cleanup & Tracking Phases

## Behavior During Fixes

**Read these rules BEFORE executing any fix phase. They are mandatory.**

- Fix findings in order (C1, C2... then H1, H2... etc). Do NOT jump around.
- For each finding: read the current file → apply the fix → verify → commit. That's it.
- ONE fix per commit. Don't bundle. Don't "while I'm here" other changes.
- If a fix doesn't work on first attempt, revert and mark "⚠️ needs human review". Do NOT try 3 different approaches in sequence.
- Don't explain what you're about to do. Don't summarize what you just did. Just do it.
- If two findings conflict (fixing one breaks the other), flag both and move on. Don't try to solve the conflict creatively.

---

## Phase 16: Auto-Fix Findings

After generating qa-report.md, offer to fix findings in severity order.

### Process:
1. Read qa-report.md
2. Fix in order: 🔴 Critical → 🟠 High → 🟡 Medium → 🔵 Low
3. For EACH finding:
   a. Read the CURRENT file content
   b. Apply the fix exactly as described in the report
   c. Run `npx tsc --noEmit` to verify no type errors
   d. If type check fails: revert and flag as "⚠️ needs manual fix" in report
   e. Commit with message `fix: [QA-{ID}] {title}` (e.g., `fix: [QA-C1] JWT secret fallback`, `fix: [QA-H3] missing error boundary`)
4. After ALL findings of a severity level are done, run verification (Phase 16B)
5. Update qa-report.md to mark fixed items as ✅

**Important:** Phase 16 is for BUG FIXES (things that are wrong). Phase 17 (Refactor) is for STRUCTURAL IMPROVEMENTS (extract service, simplify code). Phase 18 (Cleanup) is for REMOVAL (dead code, unused deps). If a finding is "refactor X" or "remove unused Y", skip it here — it belongs in Phase 17 or 18.

### Phase 16B: Post-Fix Verification (MANDATORY)

**This phase is NOT optional. Fixes without verification are not verified.**

Run ALL of these in order. If any step fails, report which fixes broke it:

```bash
# 1. Type check
npx tsc --noEmit

# 2. Lint
npm run lint

# 3. Unit tests
npm test

# 4. Build (catches SSR issues, missing imports, env vars)
npm run build

# 5. E2E tests (if playwright/cypress is configured)
# Detect which E2E framework exists:
#   - If playwright.config.* exists: npx playwright test
#   - If cypress.config.* exists: npx cypress run
#   - If neither: SKIP but WARN "No E2E tests configured — fixes are NOT fully verified"

# 6. Start dev server and smoke test critical flows
# If E2E tests don't exist or are minimal:
#   a. Run `npm run dev &` (background)
#   b. Wait for server ready
#   c. Hit critical API endpoints with curl:
#      - GET /api/health (if exists)
#      - GET /api/v1/dashboard (with auth token if possible)
#   d. Kill dev server
#   e. Report results
```

**Verification report** (append to qa-report.md):
```markdown
## Fix Verification
| Check | Result |
|-------|--------|
| TypeScript | ✅ 0 errors / ❌ N errors |
| Lint | ✅ 0 errors / ❌ N errors |
| Unit tests | ✅ N passed / ❌ N failed |
| Build | ✅ success / ❌ failed |
| E2E tests | ✅ N passed / ❌ N failed / ⚠️ not configured |
| Smoke test | ✅ endpoints responding / ❌ failures |
| Deploy build | ✅ exact deploy command passes / ❌ failed |
```

### Deploy Build Simulation (MANDATORY)
After all other checks pass, simulate the exact deploy:
```bash
# 1. Find the deploy config
cat netlify.toml 2>/dev/null || cat vercel.json 2>/dev/null || cat Dockerfile 2>/dev/null

# 2. Extract the build command and run it EXACTLY as written
# Example for Netlify:
#   [build]
#   command = "npm run lint && npx tsc --noEmit && npm test -- --passWithNoTests && npm run build"
# → Run that FULL string, not just `npm run build`

# 3. If the deploy command includes tests, those tests MUST pass
# 4. If env vars are needed at build time, verify they exist or use dummy values
```
If the deploy build command fails locally, the deploy WILL fail in CI. Fix before pushing.

### Rules:
- Fix ONE finding per commit (atomic commits)
- Never change behavior, only fix the issue
- If the fix requires new dependencies, install them
- If unsure about a fix, mark it "⚠️ needs human review" instead of guessing
- **NEVER say "all tests pass" if you only ran unit tests. Be explicit about WHAT was tested.**
- **If E2E tests exist but fail, the fix is NOT complete — investigate and fix or revert.**
- **After creating or modifying any test file, RUN that specific test before committing.** If a new test fails on first run, it's a bug in the test, not in the code — fix the test or delete it. Never commit a test you haven't run.

---

## Phase 17: Refactor

Use find-skills: `"refactor clean code"`, `"code refactoring SOLID"`

### Process:
1. Read qa-report.md sections: Medium findings, Architecture Recommendations
2. Group related refactoring opportunities
3. For each refactoring:
   a. Create branch: `refactor/qa-{description}`
   b. Apply refactoring in small steps
   c. Run tests after each step
   d. Commit with descriptive message

### Common refactorings:
- **Extract Service**: Business logic stuck in API routes → extract to `lib/services/`
- **Extract Helper**: Repeated patterns (ownerFilter, pagination) → shared utility
- **Consolidate Auth**: Dual auth systems → single source of truth
- **Simplify Functions**: Long functions → smaller, named functions
- **Remove Duplication**: Copy-pasted code → shared module
- **Type Narrowing**: Replace `any` with proper types
- **Constants Extraction**: Magic numbers/strings → named constants file
- **Error Handling**: Scattered try/catch → centralized error handler

### Rules:
- Refactor ONLY what the report identified — don't go on a refactoring spree
- Always keep tests passing
- One logical change per commit
- If a refactoring touches > 10 files, break it into smaller PRs

---

## Phase 18: Cleanup

Use find-skills: `"dead code cleanup knip"`, `"declutter project structure"`

### 18A Dead Code Removal
1. Run detection tools:
   ```bash
   npx knip                    # unused files, exports, dependencies
   npx depcheck                # unused npm dependencies
   npx tsc --noEmit            # type errors that indicate dead paths
   ```
2. Review results, cross-reference with qa-report.md Phase 9
3. For each confirmed dead item:
   - Verify it's truly unused (check dynamic imports, config references, CLI usage)
   - Delete it
   - Run tests
4. Commit: `chore: remove unused {files/deps/exports}`

### 18B Dependency Cleanup
1. Remove unused dependencies: `npm uninstall {package}`
2. Update outdated packages (non-breaking): `npx npm-check-updates -u --target minor`
3. Fix known CVEs: `npm audit fix`
4. Replace abandoned packages with maintained alternatives
5. Move misplaced deps (devDependencies in dependencies or vice versa)

### 18C Project Structure
1. Move misplaced files to correct directories
2. Remove orphaned config files
3. Clean up root directory (keep it under 20 items)
4. Ensure consistent naming conventions
5. Verify .gitignore covers everything it should

### Rules:
- NEVER delete code you're not sure about — mark as "candidate for deletion" instead
- Don't mix cleanup with behavior changes
- Run FULL test suite after cleanup
- Keep a deletion log in the commit message listing what was removed and why

---

## Phase 19: Incremental Review (git diff only)

When triggered by `review changes` or `review diff`:

1. Get changed files:
   ```bash
   # Staged changes
   git diff --cached --name-only
   # Unstaged changes
   git diff --name-only
   # Both
   git diff HEAD --name-only
   ```

2. For each changed file, run applicable phases:
   - .ts/.tsx/.js/.jsx → Phase 1 (code quality) + Phase 5 (clean code)
   - schema.prisma / migrations → Phase 4 (database)
   - supabase/migrations/ → Phase 4 (database) + Phase 3E (RLS check on new/altered tables)
   - API routes → Phase 3 (security) + Phase 10 (API contracts) + Phase 2 (business logic)
   - Files containing "use server" → Phase 3B (auth check inside each action — mandatory)
   - middleware.ts / middleware.js → Phase 3B (route protection audit)
   - Components/pages → Phase 6 (UI) + Phase 7 (UX/a11y)
   - package.json → Phase 9 (dependencies)
   - .env / config → Phase 0A (environment detection) + Phase 14 (git hygiene)
   - CI/deploy files → Phase 15 (CI/CD)
   - Test files → Phase 11 (test quality)
   - supabase/config.toml / storage policies → Phase 3E (BaaS security)

3. Generate `qa-diff-report.md` (shorter format, only changed files)

---

## Phase 20: Compare Reports

When triggered by `compare reports` or when a previous qa-report.md exists:

1. Check for `qa-report.md` and `qa-report.prev.md` (or git history of qa-report.md)
2. Compare:
   - New issues introduced
   - Issues fixed since last report
   - Issues that got worse (severity increased)
   - Issues unchanged (still open)
3. Generate summary:
   ```
   ## Progress Since Last Review
   - Fixed: 5 issues (2 Critical, 1 High, 2 Medium)
   - New: 2 issues (0 Critical, 1 High, 1 Medium)  
   - Remaining: 12 issues
   - Overall: improving ✅ / stagnant ⚠️ / degrading 🔴
   ```
4. Before generating new report, rename current to `qa-report.prev.md`
