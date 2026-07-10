---
title: Fullstack Agent System — Claude Project Setup
type: project-setup
system: Fullstack Agent System
---

# Fullstack Agent System — Claude Project Setup

Time de engenharia sênior. 7 agentes especializados + orchestrator. Código compilável, evidência obrigatória, segurança com veto técnico.

## System prompt

Usar `orchestrator.md` para sessões completas. Para trabalho focado:

| Sessão | Arquivo | Modelo |
|--------|---------|--------|
| Orquestração + planning | `orchestrator.md` | opus-4-8 |
| APIs, DB, microserviços | `backend-dev.md` | sonnet-4-6 |
| UI/UX, React/Vue | `frontend-dev.md` | sonnet-4-6 |
| AWS, Terraform, CI/CD | `infra-cloud.md` | sonnet-4-6 |
| ML, ETL, LLMs, RAG | `data-ai.md` | opus-4-8 |
| AppSec, OWASP | `security.md` | opus-4-8 |
| Scan de segurança (static/dynamic) | `probe.md` | sonnet-4-6 |
| Code quality + refactor | `forge.md` | sonnet-4-6 |

## Documentos para o projeto

- `docs/constitution.md` — 6 princípios que governam os agentes (sempre)
- `docs/standards-anti-patterns.md` — padrões técnicos
- `docs/progress.md` — estado atual do projeto (File-as-Bus)
- ADRs relevantes de `docs/adr/`

## Ref

- Sistema completo: `README.md`
