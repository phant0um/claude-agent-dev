# claude-agent-dev

**Agentes que constroem agentes (e software) com o Claude Code.** Um lifecycle de 5 subagentes com identidade, guardrails e ciclo de vida — cada um faz uma fase e passa o bastão.

Extraídos de um vault Obsidian que opera como SO de agentes IA. Só os agentes **genéricos** (servem fora do vault) vivem aqui.

## Setup em 30s

```
/plugin marketplace add phant0um/claude-agent-dev
/plugin install claude-agent-dev
```

Invoca com `@nome` (ex: `@spec`, `@guard`).

---

## O lifecycle

Um pedido de feature/agente passa pelas fases na ordem — cada agente é cético do anterior:

```
@spec  →  (build)  →  @verify  →  @guard  →  ship
              ↑                                 │
           @extend ←──── @review (drift) ←──────┘
```

| Agente | Fase | O que faz | Model |
|--------|------|-----------|-------|
| [spec](agents/spec.md) | antes de codar | Spec-driven dev: constitution → specify → clarify → plan → tasks. "Specs become executable." | opus |
| [verify](agents/verify.md) | pós-implementação | Quality gate adversarial: valida código contra spec, behavioral contracts, bloqueia merge. Não elogia — audita. | opus |
| [guard](agents/guard.md) | pré-deploy | Security audit: OWASP LLM Top 10 + Agentic AI Top 10 + guardrails do harness. Roda Opus por padrão — segurança não economiza token. | opus |
| [extend](agents/extend.md) | evolução | Extensão cirúrgica de agente existente: uma mudança por vez, testada em isolamento, usuário na direção. | opus |
| [review](agents/review.md) | higiene | Detecta e corrige drift entre docs, código e config. Mecânico no fix, preciso no relatório. | haiku |

Estes 5 vivem em [`agents/`](agents/).

---

## Sistemas multi-agente

Além dos 5 do lifecycle, o repo traz dois times completos que se coordenam:

### [`nexus-agent-system/`](nexus-agent-system/) — orquestração cost-aware

Orquestrador (`nexus`) delega a especialistas; um **`model-router`** escolhe o tier de modelo (barato vs premium) por tarefa — não queima Opus onde Haiku resolve. 8 agentes + roteamento: nexus, model-router, scout, forge, shield, pixel, herald, ledger.

### [`fullstack-agent-system/`](fullstack-agent-system/) — time de dev sênior

`orchestrator` (Maestro) delega a especialistas de domínio: backend, frontend, data/AI, infra/cloud, security. `probe` testa, `forge` constrói. 8 agentes + bootstrap de projeto.

---

## Os problemas que isto resolve

### O agente pula direto pro código e a spec vira dívida

Sem spec formal, cada decisão de design fica implícita no código — e some. → **`@spec`** produz artefatos executáveis (contratos comportamentais, critérios de done) antes da primeira linha.

### O agente elogia o próprio trabalho medíocre

"Agents tend to respond by confidently praising the work — even when the quality is obviously mediocre." → **`@verify`** é o antídoto: separação deliberada entre quem constrói e quem julga. Cético por padrão.

### Vulnerabilidade de LLM/agente passa pro deploy

Prompt injection, secret hardcoded, tool destrutiva sem gate, excessive agency. → **`@guard`** roda pré-scan determinístico + checklists OWASP/Agentic-AI e bloqueia por severidade.

### Mexer num agente que funciona quebra outra coisa

→ **`@extend`** faz mudança mínima, com smoke test em isolamento, usuário decidindo a direção.

### Docs dizem uma coisa, código faz outra

→ **`@review`** varre e zera o drift entre documentação, código e config.

---

## Companion: skills

Estes agentes acionam skills de raciocínio/escrita que vivem no pack irmão **[phant0um/claude-skills](https://github.com/phant0um/claude-skills)** — `grill-me`, `debate`, `pre-mortem`, `council`, `diagnose`, `content-design`. Instala os dois p/ o fluxo completo (agente orquestra, skill executa a sub-tarefa).

Regra de escopo: **skill** = comportamento reusável sem identidade → vai no claude-skills. **Agente** = identidade + ciclo de vida + guardrails → vai aqui.

---

## Créditos

- Spec-driven development inspirado no fluxo `.specify` (GitHub spec-kit).
- OWASP LLM Top 10 e checklists Agentic AI Top 10 — OWASP GenAI.
- Padrão generate/review (construtor ≠ juiz) e "surgical, tested in isolation" — comunidade Claude Code.

## Licença

MIT — ver [LICENSE](LICENSE).
