# QA Orchestrator — Guía de Uso

## Qué es

Un skill custom que le dice a Claude (o cualquier agente) cómo hacer un review completo de tu codebase. Cubre código, lógica de negocio, seguridad (incluyendo BaaS/RLS, data privacy, webhooks), base de datos, UI/UX, performance, y más. No es un programa — es un set de instrucciones que el agente lee y ejecuta.

## Setup actual

```
Proyecto:    C:\Users\cabre\OneDrive\Desktop\repos\pilates-booking
Skills:      C:\Users\cabre\.agents\skills\qa-orchestrator\
Agente:      Claude Code CLI (Opus 4.6) via terminal en Antigravity
Plan:        Claude Max
```

## Comandos

Todos se escriben en la terminal de Claude Code (donde dice `>`).

### Review completo (fases 0-15)

```
Read C:\Users\cabre\.agents\skills\qa-orchestrator\SKILL.md and all files in C:\Users\cabre\.agents\skills\qa-orchestrator\references\ — then follow those instructions. Run full review, all phases 0 through 15. Generate qa-report.md when done.
```

### Review parcial (fases 0-10)

```
Read C:\Users\cabre\.agents\skills\qa-orchestrator\SKILL.md and references/ — run review phases 0-10. Generate qa-report.md when done.
```

### Solo seguridad (fase 3 completa: auth + BaaS/RLS + privacy)

```
Read the qa-orchestrator skill. Run check security (phases 0 and 3 only — including 3E BaaS security and 3G data privacy). Generate qa-report.md.
```

### Solo BaaS/RLS (si solo querés chequear Supabase)

```
Read the qa-orchestrator skill, specifically references/phase-review.md section 3E. Check RLS status on all tables, storage bucket policies, auth configuration, and API exposure. Report findings.
```

### Solo UX (fases 6-7)

```
Read the qa-orchestrator skill. Run ux review (phases 0, 6, 7 only). Generate qa-report.md.
```

### Solo business logic (fase 2)

```
Read the qa-orchestrator skill. Run business review (phases 0 and 2 only). Generate qa-report.md.
```

### Arreglar issues (fases 16-18) — después del review

```
Read the qa-orchestrator skill (SKILL.md + references/phase-fix.md). Run fix issues (phases 16-18). Use the existing qa-report.md. Fix findings in order (C→H→M→L), one per commit. If a fix fails on first attempt, revert and mark as "needs human review". After ALL fixes, run Phase 16B verification: tsc, lint, unit tests, build, AND E2E tests (npx playwright test). Do not mark anything as fixed unless ALL checks pass.
```

### Review incremental (fase 19) — solo lo que cambió

```
Read the qa-orchestrator skill. Run review changes — only review git diff, not the full codebase. Update qa-report.md.
```

### Comparar reportes (fase 20)

```
Read the qa-orchestrator skill. Run compare reports — diff current qa-report.md against the previous version in git history. Show progress.
```

## Output

El review genera `qa-report.md` en la raíz del proyecto. Commitealo:

```bash
git add qa-report.md
git commit -m "QA: full review report"
```

Así la próxima vez que corras review incremental o compare, tiene contra qué comparar.

## Tokens y uso eficiente

- Un full review (15 fases) consume bastante. Con Max 5x (~88K tokens/ventana de 5hs) puede que necesites 2 ventanas. Con Max 20x (~220K) debería entrar.
- Podés ver cuánto llevás consumido con `/cost` en la terminal de Claude Code.
- Si se te corta a mitad de camino, cuando se resetee la ventana decile: `Continue the qa-orchestrator review from phase X. The partial qa-report.md is already in the project root. Read SKILL.md and references/phase-review.md before continuing.`
- El review incremental (fase 19) consume mucho menos que el full — solo mira el git diff.
- Para ahorrar tokens: usá review parcial (fases 0-10) primero, arreglá todo, y después corré fases 11-15 en otra ventana.
- Si el agente empieza a divagar o repetirse, mandá: `Stop. Follow Output Behavior rules from SKILL.md. One answer per finding, no repetition, no filler.`

## Cuándo correr cada cosa

| Situación | Comando |
|-----------|---------|
| Primera vez en un proyecto | Full review (fases 0-15) |
| Después de arreglar cosas | Review incremental (fase 19) |
| Antes de un deploy | Check security + review changes |
| Refactoring grande | Full review de nuevo |
| Querés ver progreso | Compare reports (fase 20) |
| Email de Supabase Security Advisor | Solo BaaS/RLS (fase 3E) |
| Agregar tabla nueva a la DB | Check security (fase 3) — verifica RLS en la tabla nueva |
| Cambio en .env o auth config | Review parcial (fases 0-3) |

## Herramientas que hacen esto automático (sin que vos tipees nada)

### En cada PR de GitHub (automático)

| Herramienta | Qué hace | Precio |
|---|---|---|
| **CodeRabbit** | Review automático line-by-line en cada PR. Repos públicos gratis. | $0 (público) / $24/mes (privado) |
| **Qodo Merge** | Review de PRs con /review, /improve, /compliance. 75 PRs/mes gratis. | $0 (75 PRs) |
| **Claude Code Security** (GitHub Action) | Scan de seguridad en cada PR. Necesita API key de Anthropic. | Costo de API |

Instalación de CodeRabbit (una vez, en GitHub):
1. Andá a github.com/apps/coderabbitai
2. Install en tu repo pilates-booking
3. Listo — cada PR que abras se revisa solo

Instalación de Qodo Merge:
1. Andá a github.com/apps/qodo-merge
2. Install en tu repo
3. En cada PR escribís `/review` o `/improve` en un comentario

Instalación de Claude Code Security (GitHub Action):
```yaml
# .github/workflows/security-review.yml
name: Security Review
permissions:
  pull-requests: write
  contents: read
on:
  pull_request:
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha || github.sha }}
          fetch-depth: 2
      - uses: anthropics/claude-code-security-review@main
        with:
          comment-pr: true
          claude-api-key: ${{ secrets.CLAUDE_API_KEY }}
```

### Pre-commit (automático antes de cada commit)

Podés hookear Claude Code a git para que revise antes de commitear:

```bash
# Instalar husky
npm install -D husky
npx husky init

# Crear hook pre-commit
echo 'claude -p "Review the staged changes (git diff --staged). Flag any critical security or logic issues. Be brief."' > .husky/pre-commit
```

Esto es más liviano que un full review — solo mira lo que estás por commitear.

### En la terminal (semi-automático con alias)

Agregá estos aliases a tu PowerShell profile (`$PROFILE`):

```powershell
function qa-full { claude -p "Read C:\Users\cabre\.agents\skills\qa-orchestrator\SKILL.md and all files in C:\Users\cabre\.agents\skills\qa-orchestrator\references\ — then run full review, all phases 0-15. Generate qa-report.md." }
function qa-quick { claude -p "Read the qa-orchestrator skill. Run review phases 0-5 only. Generate qa-report.md." }
function qa-security { claude -p "Read the qa-orchestrator skill. Run check security (phase 0 + phase 3 complete including 3E BaaS and 3G privacy). Generate qa-report.md." }
function qa-baas { claude -p "Read the qa-orchestrator skill, specifically references/phase-review.md section 3E. Check RLS on all tables, storage buckets, auth config, API exposure. Report findings." }
function qa-diff { claude -p "Read the qa-orchestrator skill. Run review changes — only git diff. Update qa-report.md." }
function qa-fix { claude -p "Read the qa-orchestrator skill (SKILL.md + references/phase-fix.md). Run fix issues phases 16-18 using qa-report.md. Fix in order, one per commit. Run Phase 16B verification after all fixes." }
```

Después solo tipeás `qa-full`, `qa-security`, `qa-baas`, `qa-fix`, etc.

## Resumen del stack completo

```
DESARROLLO
    │
    ├── Pre-commit hook ──→ Claude Code revisa staged changes
    │
    ├── Push / PR ──→ CodeRabbit + Qodo Merge revisan automáticamente
    │                  Claude Code Security escanea vulnerabilidades
    │
    ├── QA manual ──→ qa-orchestrator full review (fases 0-15)
    │                  qa-orchestrator incremental (fase 19)
    │
    ├── Infraestructura ──→ qa-baas (RLS, storage, auth config)
    │                        Supabase Security Advisor (email automático)
    │
    └── Antes de deploy ──→ qa-security + qa-diff
```

Costo total: **$0** (todo gratis o incluido en tu Max).
