# claude-agent-dev

**Um orquestrador e cinco agentes filhos para o Claude Code.** O Nexus planeja
e delega; Shield revisa, Scout pesquisa, Forge implementa, Hill endurece
agentes e Finder localiza. Só o Nexus escreve o resultado canônico, e cada
filho tem subconjunto da autoridade do pai.

Extraído de um vault Obsidian que opera como SO de agentes IA, e reescrito
para funcionar em qualquer projeto.

## Setup em 30s

```
/plugin marketplace add phant0um/claude-agent-dev
/plugin install claude-agent-dev
```

Comece sempre pelo Nexus: `@nexus [tarefa]`. Leia antes
[AGENT-BASE](nexus-agent-system/AGENT-BASE.md): as regras comuns vivem ali e cada agente
contém só o diff.

---

## Os agentes

| Agente | Trigger | Função | Modelo · effort |
|--------|---------|--------|-----------------|
| [nexus](agents/nexus.md) | `@nexus [tarefa]` | Orquestra: classifica, delega por TaskPacket, valida e materializa (writer único) | opus-5-5 · medium |
| [shield](agents/shield.md) | `@shield`, revisão, segurança, deploy | Revisão crítica de segurança e arquitetura: PASS/FAIL/null com evidência | opus-5-5 · high |
| [scout](agents/scout.md) | `@scout`, pesquise, compare | Pesquisa com fonte citada, contradições e lacunas declaradas | opus-5-5 · medium |
| [forge](agents/forge.md) | `@forge`, implemente, refatore | Código e testes em escopo fechado; não decide arquitetura | opus-5-5 · medium |
| [hill](agents/hill.md) | `@harden <slug>` | Endurece agente existente: eval → diagnóstico → lever, validado por McNemar | opus-5-5 · medium |
| [finder](agents/finder.md) | `@finder`, onde está, localize | Tabela `path:linha` de todo acerto; não sintetiza | haiku-4-5 · low |

## Fluxo

```
            humano
              │  @nexus [tarefa]
              ▼
            nexus ── ambíguo e caro? → grill-me → [DECISION NEEDED]
              │  TaskPacket (objetivo · escopo · evidência · negatives)
   ┌─────────┬┴────────┬─────────┬─────────┐
 shield    scout     forge      hill     finder
   └─────────┴────┬────┴─────────┴─────────┘
                  ▼  relatório único (veredito + paths)
            nexus valida e materializa
```

Arquitetura, roteamento de modelo e policies:
[`nexus-agent-system/`](nexus-agent-system/README.md).

---

## O que saiu

A versão 0.1 trazia `spec`, `verify`, `guard`, `extend`, `review` e o
`fullstack-agent-system/` (orchestrator, backend, frontend, data-ai, infra,
security, probe, forge, project-setup), além de herald, pixel e ledger no
nexus-agent-system. No vault de origem esses papéis viraram **skills** —
comportamento reusável sem identidade própria — ou foram absorvidos pelos
agentes atuais. Essas skills não são publicadas aqui.

## Companion: skills

Os agentes acionam skills do pack irmão
**[phant0um/claude-skills](https://github.com/phant0um/claude-skills)**:
`grill-me`, `council`, `debate`, `pre-mortem`, `office-hours`, `diagnose`,
`trace`, `content-design`, `content-design-review`, `writing-fragments`,
`writing-shape`, `writing-great-skills`. Instale os dois para o fluxo completo
(agente orquestra, skill executa a sub-tarefa). Skills citadas nos agentes como
"não incluída" existem só no vault de origem.

Regra de escopo: **skill** = comportamento reusável sem identidade → vai no
claude-skills. **Agente** = identidade + ciclo de vida + guardrails → vai aqui.

## Licença

MIT — ver [LICENSE](LICENSE).
