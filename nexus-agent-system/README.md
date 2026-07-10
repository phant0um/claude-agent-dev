---
title: "Nexus Agent System"
description: "Orquestração multi-agente cost-aware para projetos Claude Code"
version: "3.0.0"
status: active
tags: [agents, claude-code, orchestration, cost-aware]
---

# Nexus Agent System

Sistema de orquestração multi-agente para qualquer projeto rodando em Claude Code.
Um orquestrador (**Nexus**) recebe cada tarefa, decide qual especialista deve agir
e delega com contexto mínimo. Uma camada dedicada de roteamento (**Model Router**)
escolhe o modelo e o `effort` certos por tipo de tarefa — o objetivo é qualidade
máxima com custo mínimo, sem rodar um modelo caro onde um barato resolve.

## O problema

Um único agente genérico faz tudo mal e caro: ou você paga tier premium para
tarefas triviais, ou usa um modelo fraco em decisões que exigem julgamento.
Este sistema separa **quem decide** (Nexus, tier alto) de **quem executa**
(especialistas, tier ajustado à tarefa) e torna a escolha de modelo explícita
e auditável em vez de implícita.

## Os 9 agentes

| Agente | Papel | Modelo · effort padrão |
|--------|-------|------------------------|
| `nexus` | Orquestrador — ponto de entrada, delega, mantém estado | claude-opus-4-8 · high |
| `model-router` | Camada de roteamento — escolhe modelo/effort por tarefa | (contexto injetado) |
| `scout` | Pesquisa, comparação e descoberta → briefings acionáveis | claude-haiku-4-5 |
| `forge` | Implementação, refatoração e testes | claude-opus-4-8 · low |
| `shield` | Segurança e arquitetura crítica — PASS/FAIL com evidência | claude-opus-4-8 · high |
| `pixel` | UI, componentes e design system | claude-opus-4-8 · low |
| `herald` | Documentação, README, changelog, PR description | claude-haiku-4-5 |
| `ledger` | Memória e auditoria — registra sessões e cria ADRs | claude-haiku-4-5 |

## Arquitetura

```
nexus (orchestrator)
│   └── model-router (injeta a decisão de modelo/effort antes de cada delegação)
├── scout   → pesquisa e descoberta        → claude-haiku-4-5
├── forge   → implementação e código       → claude-opus-4-8 · low
├── shield  → validação e segurança        → claude-opus-4-8 · high
├── pixel   → UI/UX e apresentação visual   → claude-opus-4-8 · low
├── herald  → comunicação e documentação    → claude-haiku-4-5
└── ledger  → memória e auditoria           → claude-haiku-4-5
```

## Roteamento de modelo (cost-aware)

O Model Router aplica uma régua simples: **julgamento crítico → tier alto (Opus);
trabalho mecânico ou de volume → tier barato (Haiku/Sonnet)**. Sempre declara o
`effort`, porque ele controla os tokens de thinking e portanto o custo real.

| Agente | Modelo · effort | Alternativa mais barata | Quando usar a alternativa |
|--------|-----------------|-------------------------|---------------------------|
| Nexus | claude-opus-4-8 · high | — | nunca (decisor) |
| Scout | claude-haiku-4-5 | — | volume alto |
| Forge | claude-opus-4-8 · low | claude-sonnet-5 · medium | tarefas repetitivas |
| Shield | claude-opus-4-8 · high | — | nunca (invariante) |
| Pixel | claude-opus-4-8 · low | claude-sonnet-5 · medium | protótipos rápidos |
| Herald | claude-haiku-4-5 | — | relatórios em lote |
| Ledger | claude-haiku-4-5 | — | logs e ADRs simples |

> Detalhes de escalada e anti-padrões: `model-router.md`.

## Ciclo de vida

```
[Scout descobre] → [Nexus decide] → [Forge constrói] → [Shield valida]
       ↑                                                      ↓
[Ledger memoriza] ← [Herald comunica] ← [Pixel apresenta] ←──┘
```

## Como invocar

Sempre inicie pelo Nexus. Ele lê `docs/operations.md`, consulta o Model Router,
decide o agente certo e delega com o contexto mínimo necessário.

Prompt inicial padrão:

> "@nexus — [descrição da tarefa]. Contexto: [link/arquivo relevante]."

Também é possível invocar um especialista direto quando o agente é óbvio:

> "@forge — implemente paginação no endpoint `/users`."

## Estado do projeto (docs esperados)

| Arquivo | Propósito |
|---------|-----------|
| `docs/operations.md` | Estado atual — último ciclo, bloqueios, audit trail |
| `docs/progress.md` | Tarefas, critério de done, próximos passos |
| `docs/standards.md` | Critérios de qualidade e anti-padrões |
| `docs/adr/` | Decisões arquiteturais (ADRs) |
| `docs/lessons.md` | Lições acumuladas (self-improvement) |

## Regras do sistema

1. Nenhum agente acessa mais contexto do que o necessário para sua tarefa
2. Ledger é chamado ao final de todo ciclo — sem exceção
3. Shield é obrigatório antes de qualquer deploy ou mudança crítica
4. `docs/operations.md` é atualizado a cada sessão pelo Nexus
5. ADRs são criados para toda decisão que afeta arquitetura ou padrões
6. **Julgamento crítico** (orquestração, segurança, decisões destrutivas) → tier alto (Opus)
7. **Trabalho mecânico / de volume** → tier barato (Haiku/Sonnet), com escalada se necessário
8. **Step gated / manual** (guardrail exige o usuário rodar) → emitir bloco `bash`
   pronto p/ copiar e colar (`[RUN MANUAL]`, comando exato, sem placeholder). Ver
   `nexus.md` § Comando manual.
