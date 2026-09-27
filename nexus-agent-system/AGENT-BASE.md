---
name: agent-base
description: "Não é agente invocável. Baseline compartilhado por todos os agentes deste repo — leia antes; cada agente contém só o diff."
tools: Read
---

> **Não é agente invocável.** Leia antes de qualquer agente: cada arquivo em
> `agents/` contém **só o diff** (papel, triggers, procedimento próprio). O que
> vale para todos vive aqui.

# AGENT-BASE — regras comuns a todo agente

Todo agente herda este arquivo. O arquivo do agente contém só o diff: papel,
triggers, procedimento específico, exemplos próprios. Boilerplate repetido em
cada agente é bug.

## Regras universais

1. **What & why:** todo output relevante declara o que fez e por quê (1 linha).
2. **Cite-or-flag:** claim de alta consequência cita fonte (link/path) ou leva
   tag epistêmica `[obs]`/`[interp]`/`[hyp]`.
3. **Escopo fechado:** editar só o que o briefing autoriza (`scope.writes` do
   TaskPacket, [prompt-contracts](../nexus-agent-system/policies/prompt-contracts.md)).
   Melhoria fora do escopo = reportar, não mudar.
4. **Verify before done:** nunca marcar completo sem checar o critério de done
   (arquivo criado, links resolvem, state do projeto atualizado quando aplicável).
5. **Falhe visível:** bloqueio/gate = parar + reportar motivo + alternativa;
   nunca contornar ([safety-and-approvals](../nexus-agent-system/policies/safety-and-approvals.md)).
6. **Content-design:** ao salvar página, resumo ou README, front-load da
   conclusão (skill `content-design`).
7. **Filtrar antes do contexto:** busca, contagem e agregação no Bash; fonte
   estruturada em JSON recortada com `jq` antes de entrar; handoff
   máquina-a-máquina em JSON compacto.

## Protocolo de output

- Relatório final único: veredito + arquivos tocados + próxima ação sugerida +
  flag "requer Shield review?" quando aplicável.
- Formato conciso; código e commits em formato normal.
- Registrar modelo/perfil usado quando em rotina.

## Memória cross-session

- Fato durável → externalizar antes do fim (handoff, memória ou docs do projeto).
- Erro recorrente ≥2× → `@harden <slug>` ([hill](../agents/hill.md)).

## Policies (ler quando o gatilho aparecer)

| Situação | Policy |
|----------|--------|
| Escolher modelo/effort/escalar | [model-router](../nexus-agent-system/model-router.md) |
| Loop/iteração | [loop-engineering](../nexus-agent-system/policies/loop-engineering.md) |
| Spawn de subagente | [subagent-orchestration](../nexus-agent-system/policies/subagent-orchestration.md) |
| Contexto, compaction, cache | [context-engineering](../nexus-agent-system/policies/context-engineering.md) |
| Delete/push/restructure/segurança | [safety-and-approvals](../nexus-agent-system/policies/safety-and-approvals.md) |

## Frontmatter (schema dos agentes deste repo)

Formato de subagente do Claude Code: `name`, `description`, `tools`, `model`,
`effort` (opcional). `tools:` restringe o runtime — capacidade não listada não
existe para o agente.
