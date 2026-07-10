# claude-agent-dev — router

Subagentes genéricos p/ construir agentes e software com Claude Code. Claude lê no boot; carrega o agent file quando o trigger casa ou o usuário invoca `@nome`.

## Regra de escopo (o que entra aqui)

Entra só o que **serve fora do vault de origem** e é **agente** (identidade + ciclo de vida + guardrails), não skill. Teste skill-vs-agent: resolve com uma skill bem escrita? → vai pro [claude-skills](https://github.com/phant0um/claude-skills). Precisa de identidade própria e fases? → agente, entra aqui.

## Mapa

| Agente | Trigger | Fase | Função |
|--------|---------|------|--------|
| spec | `@spec [feature]` | antes de codar | Spec-driven dev: constitution→specify→clarify→plan→tasks |
| verify | `@verify [feature]` | pós-implementação | Quality gate adversarial: código vs spec, behavioral contracts |
| guard | `@guard [alvo]` | pré-deploy | Security audit: OWASP LLM + Agentic AI Top 10 + guardrails |
| extend | `@extend [agente]` | evolução | Extensão cirúrgica: 1 mudança, testada em isolamento |
| review | `@review` | higiene | Detecta+corrige drift docs↔código↔config |

## Fluxo

`@spec` → build → `@verify` → `@guard` → ship. `@review` fecha o loop de higiene; `@extend` evolui um agente existente sem quebrar o resto.

## Companion

Agentes acionam skills do pack [phant0um/claude-skills](https://github.com/phant0um/claude-skills) (grill-me, debate, pre-mortem, council, diagnose). Instalar os dois p/ fluxo completo.

## Manutenção

- Novo agente genérico no vault → avaliar migração (regra de escopo acima).
- Agente que ganha acoplamento a domínio → sai daqui, volta pro sistema-pai.
- Versionar: bump `plugin.json` version no merge (semver).
