# Nexus Agent System

Sistema de agentes orquestrado pelo **Nexus**: um orquestrador que é writer
único, cinco filhos com capacidade restrita, briefing por contrato (TaskPacket)
e gate humano nos pontos que não se desfazem. Os agentes vivem em
[`../agents/`](../agents/); aqui ficam a arquitetura, o roteamento de modelo e
as policies.

## Arquitetura

```
humano (autoridade final)
└── nexus — orquestrador, writer único
    ├── shield — arquitetura e segurança (veto técnico)
    ├── scout  — pesquisa com fonte citada
    ├── forge  — implementação em escopo fechado
    ├── hill   — hardening de agentes (@harden)
    └── finder — localização mecânica (path:linha)
```

| Agente | Papel | Modelo · effort |
|--------|-------|-----------------|
| [nexus](../agents/nexus.md) | orquestra, delega, materializa | opus-5-5 · medium |
| [shield](../agents/shield.md) | revisão crítica, PASS/FAIL/null | opus-5-5 · high |
| [scout](../agents/scout.md) | briefing com fontes, contradições, lacunas | opus-5-5 · medium |
| [forge](../agents/forge.md) | código e testes sob briefing | opus-5-5 · medium |
| [hill](../agents/hill.md) | endurece agente existente contra falhas | opus-5-5 · medium |
| [finder](../agents/finder.md) | acha, não sintetiza | haiku-4-5 · low |

Fonte única de modelo/effort: [model-router](model-router.md). Regras comuns:
[AGENT-BASE](AGENT-BASE.md).

## Quatro invariantes da arquitetura

1. **Orquestrador é writer único.** Filhos e adapters de análise recebem
   contexto e devolvem resultado (JSON quando alimenta automação); só o Nexus
   valida schema, cobertura, budget e telemetria e então materializa state,
   índices e log de operações.
2. **Filho ⊆ pai.** Deny undeclared capability by default: o agente só tem as
   tools, paths e rede que o contrato declara, e o subagente recebe subconjunto
   da autoridade do pai, nunca superconjunto. No Claude Code o `tools:` do
   frontmatter é o enforcement — o Nexus declara `Agent(scout, forge, shield,
   hill, finder)`, o Finder nem Bash tem.
3. **TaskPacket, não prosa.** Toda delegação declara rota, objetivo
   verificável, escopo de escrita fechado, evidência, execução e `negatives`
   estruturados ([prompt-contracts](policies/prompt-contracts.md)).
4. **Handoff humano.** Ambiguidade cara, operação destrutiva, mudança >10
   arquivos, contradição entre fontes, claim sem corroboração ou erro
   inesperado: o agente para e surfa `[DECISION NEEDED]` com opções e
   recomendação ([nexus](../agents/nexus.md#handoff-humano)).

## Princípios

1. **Contexto mínimo, qualidade máxima.** Antes de incluir arquivo: "sem isso,
   o output seria pior?" Não → fora.
2. **Evidência antes de ação.** Nenhuma mudança sem critério de done
   mensurável. Opinião não substitui teste/log/diff.
3. **Drift é dívida.** Doc desatualizada = bug. Todo ciclo encerra registrando
   estado. Decisão sem ADR = decisão não rastreável.
4. **Escopo fechado por padrão.** Agente faz exatamente o delegado. Expansão
   exige nova delegação explícita.
5. **Falhe cedo, falhe visível.** FAIL com evidência > PASS com ressalvas.
6. **O sistema melhora a cada ciclo.** Hill fecha o loop de melhoria.

## Hierarquia de autoridade

Humano (final) → Nexus (orquestração) → Shield (veto técnico) → executores
(Scout, Forge, Hill, Finder).

## Despacho ≠ spawn

Despacho de domínio é obrigatório (de quem é o trabalho?); spawn de subagente é
discricionário (vale isolar contexto?). Dev e escrita não têm agente: o Nexus
despacha direto para skills. Detalhe em
[subagent-orchestration](policies/subagent-orchestration.md).

## Policies

Cada policy é curta, prescritiva, testável. Agentes carregam só a policy do
gatilho — nunca a stack inteira.

| Policy | 1 linha |
|--------|---------|
| [prompt-contracts](policies/prompt-contracts.md) | TaskPacket, GraphPacket, `negatives` estruturados, conteúdo não confiável |
| [safety-and-approvals](policies/safety-and-approvals.md) | Tiers de risco R0–R4, matriz de decisão, o que conta como aprovação |
| [loop-engineering](policies/loop-engineering.md) | Estado externo + gate objetivo + hard stop; loop guards; repair pós-falha |
| [context-engineering](policies/context-engineering.md) | Camadas de carregamento, compaction por threshold, KV cache |
| [subagent-orchestration](policies/subagent-orchestration.md) | Elegibilidade de spawn, advisor pattern, least-agency |

## Como invocar

Sempre inicie pelo Nexus. Ele lê o state do projeto, decide o agente certo e
delega com contexto mínimo necessário.

> "@nexus — [descrição da tarefa]. Contexto: [link/arquivo relevante]."

## Regras do sistema

1. Nenhum agente acessa mais contexto do que o necessário para sua tarefa.
2. Shield é obrigatório antes de qualquer deploy ou mudança crítica.
3. ADRs são criados para toda decisão que afeta arquitetura ou padrões.
4. Tarefas de julgamento (orquestração, segurança, decisões destrutivas) →
   Claude no tier alto, nunca o mais barato.
5. Step gated / manual (guardrail exige o usuário rodar) → emitir bloco `bash`
   pronto para copiar (`[RUN MANUAL]`, comando exato, sem placeholder).
