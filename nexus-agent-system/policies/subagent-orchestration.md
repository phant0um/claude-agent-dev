# Subagent Orchestration Policy

Premium orquestra, barato executa. Subagente existe para **paralelismo +
contexto main limpo**, não para delegar pensamento que o orquestrador deveria
fazer.

## Despacho ≠ spawn

Esta policy governa **spawn de subagente** (discricionário), não **despacho de
domínio** (obrigatório). Confundir os dois é a causa raiz de orquestrador
executar trabalho alheio.

| | Despacho | Spawn |
|---|---|---|
| Pergunta | de quem é o domínio? | vale isolar contexto? |
| Natureza | obrigatório | discricionário |
| Alvo | dono do domínio | executor (filho) |
| Autoridade | própria do dono | filho ⊆ pai |
| Escapar exige | escala ao humano | nada |

**Precedência:** domínio antes de elegibilidade. Domínio com dono é despachado
mesmo cabendo em poucas tool calls.

## Elegibilidade (spawnar subagente quando)

- 3+ tarefas independentes → 1 subagente cada, **em mensagem única** (paralelo).
- Tarefa gera bulk de leitura (ingest multi-fonte, sweep, research 3+ fontes)
  que poluiria o contexto main.
- Tarefa é autocontida com verifier próprio (TaskPacket completo,
  [prompt-contracts](prompt-contracts.md)).

**NÃO spawnar quando:** tarefa 1-arquivo simples; dependência sequencial forte;
decisão que exige contexto da conversa (subagente nasce frio) — só **dentro do
próprio domínio**; fora dele a regra é despacho.

## Regras

1. Todo spawn leva TaskPacket (route/objective/scope/evidence/execution/negatives).
2. Modelo do subagente vem do frontmatter do agente
   ([model-router](../model-router.md)); nunca herdar premium por inércia.
3. Output = relatório final único (veredito + paths + custo). Orquestrador não
   relê o trabalho bruto.
4. Profundidade de delegação máx **2** (subagente pode ter subagente uma vez).
5. Falha de subagente → orquestrador decide (retry com evidência nova /
   escalar / abortar) — subagente não auto-escala tier (Hard Rule 2).
6. **Teto de concorrência: 4 subagentes por fase**; progresso a cada **60 s**
   ou após gate bloqueante; retry da mesma assinatura de verifier no máx
   **2×**. Se um subagente resolve, use um. Não delegar o que se resolve em
   poucas tool calls (dentro do próprio domínio), e **não usar subagente para
   verificar o trabalho de outro** — verificação é o gate determinístico.
7. **Fork vs subagente frio.** Fork herda modelo do pai e cache quente — usar
   quando a tarefa precisa do contexto acumulado. Subagente frio usa o modelo
   do próprio perfil e cache frio — usar para busca mecânica, contexto
   isolado, volume em tier mais barato ou input não confiável.

## Contratos de agência (least-agency)

Cada agente declara `reads`, `writes`, `tools`, `network`, `allowed_actions`,
`forbidden_actions`, `approval_tiers`, `budgets` (tokens/custo/turnos/timeout),
`expected_effects`, `verifier` e `max_concurrent_subagents`. No Claude Code, o
`tools:` do frontmatter é a parte aplicada pelo runtime.

- **Deny undeclared capability by default** — capacidade não declarada não existe.
- **Filho ⊆ pai:** subagente recebe subconjunto da autoridade do pai (tools,
  rede e paths de escrita), nunca superconjunto.
- **Completo ≠ sucesso:** conclusão só conta com `expected_effects` observado
  + `verifier` executado ([safety-and-approvals](safety-and-approvals.md)).

## Advisor pattern (padrão default para julgamento raro)

Worker barato executa volume; tier deep consultado **só em ponto de decisão
explícito** (fronteira declarada no prompt do worker: "em dúvida X, formule
pergunta + contexto mínimo e pare"). Consulta por reflexo anula a economia.

## Swarm (condicional, não default)

3 experts paralelos + 1 merge. Usar SÓ quando perspectivas genuinamente
distintas agregam (revisão adversarial, decisão crítica multi-eixo) E budget
`deep` justificado. Nunca para tarefa mecânica — 3× custo sem 3× qualidade.
Adversarial/red-team: só em `critical` com gate humano.

## Check testável

Spawn sem TaskPacket = violação; relatório de rotina lista subagentes usados +
perfil de cada. Escrita em domínio alheio fora dos paths do contrato = violação.
