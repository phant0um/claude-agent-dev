# claude-agent-dev — router

Orquestrador + 5 subagentes para Claude Code. Claude lê no boot; carrega o
agent file quando o trigger casa ou o usuário invoca `@nome`. Antes de qualquer
agente: [AGENT-BASE](nexus-agent-system/AGENT-BASE.md) — cada agente contém só o diff.

## Regra de escopo (o que entra aqui)

Entra só o que **serve fora do vault de origem** e é **agente** (identidade +
ciclo de vida + guardrails), não skill. Teste skill-vs-agent: resolve com uma
skill bem escrita? → vai para o
[claude-skills](https://github.com/phant0um/claude-skills). Precisa de
identidade própria e fases? → agente, entra aqui.

## Mapa

| Agente | Trigger | Função |
|--------|---------|--------|
| [nexus](agents/nexus.md) | `@nexus [tarefa]` | Orquestra, delega por TaskPacket, writer único |
| [shield](agents/shield.md) | `@shield`, revisão, segurança, deploy, arquitetura crítica, PRs | Revisão crítica: PASS/FAIL/null com evidência |
| [scout](agents/scout.md) | `@scout`, pesquise, analise opções, compare, investigate, explore | Pesquisa com fontes, contradições, lacunas |
| [forge](agents/forge.md) | `@forge`, implemente, escreva código, crie componente, refatore | Código e testes em escopo fechado |
| [hill](agents/hill.md) | `@harden <slug>` | Hardening de agente existente via hill-climb |
| [finder](agents/finder.md) | `@finder`, onde está, localize, liste ocorrências | Localização mecânica `path:linha` |

## Fluxo

Nexus delega → shield / scout / forge / hill / finder → relatório único →
Nexus valida e materializa. Ambiguidade cara passa por `grill-me` antes de
delegar; operação destrutiva, >10 arquivos ou contradição de fontes param em
`[DECISION NEEDED]`.

Arquitetura, model-router e policies: [nexus-agent-system/](nexus-agent-system/README.md).

## O que saiu

`spec`, `verify`, `guard`, `extend`, `review` e `fullstack-agent-system/`
(além de herald, pixel, ledger) viraram skills no vault de origem — não
publicadas aqui.

## Companion

Agentes acionam skills do pack [phant0um/claude-skills](https://github.com/phant0um/claude-skills)
(grill-me, council, debate, pre-mortem, office-hours, diagnose, trace,
content-design, content-design-review, writing-fragments, writing-shape,
writing-great-skills). Skill citada como "não incluída" não existe em nenhum
dos dois repos.

## Manutenção

- Novo agente genérico no vault de origem → avaliar migração (regra de escopo acima).
- Agente que ganha acoplamento a domínio → sai daqui, volta para o sistema-pai.
- Sem wikilinks, paths do vault ou IDs de ADR internos; todo link relativo resolve.
- Versionar: bump `plugin.json` version no merge (semver).
