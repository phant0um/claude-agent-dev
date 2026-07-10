---
tags: [fullstack-agent, multi-agent-system, claude, dev-team, engineering]
version: "2.1.0"
---

# Fullstack Agent System

A senior engineering team for any Claude Code project — 8 specialized agents coordinated by a central orchestrator. Each agent is a senior IT professional, not a tutor. They think, write compilable code, build integrations, and run validation tests.

**Coordination via File-as-Bus**: state lives in files (`docs/progress.md`), not in conversation history.
**Mandatory Evidence**: no deliverable ships without tests or execution logs.
**Security with technical veto**: Sentinel can block any deploy.

---

## The Team

| Agent | File | Role | Model |
|---|---|---|---|
| **Maestro** | `orchestrator.md` | Central planner — decomposes tasks, delegates, verifies Evidence. Never writes code. | opus-4-8 |
| **Stratum** | `backend-dev.md` | Backend: APIs, DB, microservices, auth | sonnet-4-6 |
| **Facet** | `frontend-dev.md` | Frontend: UI/UX, React/Vue, accessibility, performance | sonnet-4-6 |
| **Bastion** | `infra-cloud.md` | Infra/DevOps: AWS, Terraform, CI/CD, Kubernetes | sonnet-4-6 |
| **Neuron** | `data-ai.md` | Data & AI: ETL, ML, LLMs, RAG, analytics | opus-4-8 |
| **Sentinel** | `security.md` | Security: AppSec, OWASP, compliance — deploy veto | opus-4-8 |
| **Probe** | `probe.md` | Automated security testing: static scan (pre) + dynamic scan (post) | sonnet-4-6 |
| **Forge** | `forge.md` | Code quality: 5E rubric review, migration/query audit, refactoring | sonnet-4-6 |

**Problem-first, honest:** agents reject deliverables that lack Evidence, refuse to hardcode secrets, and surface blocking findings instead of hiding them. Model routing (haiku/sonnet/opus by activity) is estimated to save ~60–75% vs. running everything on Opus.

---

## How it works

```
Maestro reads docs/progress.md
  └─→ Breaks task into sub-tasks, each with a measurable done criterion
      ├─→ Stratum   (APIs, DB, microservices)
      ├─→ Facet     (UI, components, a11y)
      ├─→ Bastion   (IaC, CI/CD, deploy)
      ├─→ Neuron    (pipelines, ML, RAG)
      ├─→ Forge     (5E quality review before merge)
      ├─→ Probe     (static/dynamic security scan)
      └─→ Sentinel  (mandatory review on auth/data/infra — can veto)
          ↓
      Each specialist delivers: Deliverable + Evidence + State Update
          ↓
      Maestro validates Evidence, updates docs/progress.md
```

Delegation format used by Maestro:

```
Agent: [name]
Objective: [1 sentence]
Required context: [minimal files or facts]
Done criterion: [measurable — test passes, endpoint responds, pipeline executes]
Next step: [agent or action after completion]
```

---

## Usage in Claude Code

Drop the whole team into a project, or scope a single specialist to a repo.

### Full team (orchestrated)

```bash
# Use Maestro as the project entry point
cp orchestrator.md ./CLAUDE.md
```

### Single specialist per repo

```bash
cp backend-dev.md ./CLAUDE.md     # backend repo (Stratum)
cp frontend-dev.md ./CLAUDE.md    # frontend repo (Facet)
cp infra-cloud.md ./CLAUDE.md     # infra repo (Bastion)
```

### Conditional rules by path

```bash
.claude/rules/services.md      # paths: ["src/**/*.service.ts"]
.claude/rules/components.md    # paths: ["src/components/**/*.tsx"]
.claude/rules/terraform.md     # paths: ["**/*.tf", "**/*.tfvars"]
.claude/rules/dbt.md           # paths: ["dbt/**/*.sql", "dbt/**/*.yml"]
.claude/rules/auth.md          # paths: ["src/auth/**", "**/middleware/**"]
```

### Loading hierarchy

```
~/.claude/CLAUDE.md    ← Maestro (global, optional)
./CLAUDE.md            ← Specialist agent (project)
.claude/rules/*.md     ← Conditional rules by path
```

---

## Operating principles (the Constitution)

1. **Evidence before code** — a spec exists before implementing; Evidence is mandatory
2. **Security by default** — Sentinel has veto on any deploy
3. **Tests are not optional** — a deliverable without tests is incomplete
4. **Closed scope by default** — each agent does exactly what was delegated
5. **Fail early, fail visibly** — errors go to logs, never silent
6. **The system improves every cycle** — recurrent corrections become principles, not one-off rules

Optional planning/stress-test skills from the public catalog (grill-me, pre-mortem, council, debate, diagnose — phant0um/claude-skills) pair well with Maestro before large tasks.
