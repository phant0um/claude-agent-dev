# Loop Engineering Policy

Um loop exige **estado externo + gate objetivo + hard stop**. Sem os três, é só
um agente se autoavaliando. Aplica-se a hill-climb, pipelines e qualquer rotina
iterativa.

## Taxonomia

| Tipo | Forma | Contrato |
|---|---|---|
| `task` | uma transformação verificável | TaskPacket |
| `chain` | etapas seriais com dependência real | TaskPacket + dependencies |
| `loop` | repete ação até gate/stop | LoopContract |
| `graph` | nós, rotas, fan-out/fan-in e falhas | GraphPacket |

Ordem textual não prova dependência. Workflow multi-fase declara
`workflow_kind`; `graph` usa o GraphPacket de [prompt-contracts](prompt-contracts.md).

## LoopContract

Todo loop declara:
`objective · trigger · procedure_ref · state_ref · verifier · budget · guards ·
stop_conditions · acceptance_authority`.

Checkpoint mínimo:
`state · decision · evidence_refs · failure_signature · next_step · updated_at`.
Checkpoint sem evidência ou próximo passo não autoriza retomada.

## Estados do loop

`discover → plan → execute → verify → iterate → escalate → stop`

## Obrigações de regressão (gate cumulativo)

Fase selada não é fase esquecida. Todo gate que passou em nó anterior do mesmo
fire permanece **obrigação ativa** até o loop parar.

- `chain`/`graph` declara `regression_gates`: `check_id` herdados de todo nó já
  concluído. O nó corrente roda `regression_gates` + gate próprio.
- Regressão detectada = falha do **nó corrente** (`error_class: exec`), não
  reabertura da fase selada; o repair é bounded ao que quebrou.
- Herança declarada vazia exige motivo no próprio nó (`regression_gates: []` +
  `reason`): lista vazia sem motivo é indistinguível de esquecimento.
- **O resultado de cada gate é selado no receipt do nó** (`gate_results`) — é o
  baseline que torna "passou antes" verificável. Gate não selado não gera
  baseline e não pode produzir regressão.
- Quebra de gate com baseline `pass` emite `regression_events`; sem baseline
  selado emite `unverified_gates` e **não** conta como regressão.

Sem isto, "verify passou" mede só o último nó. Evidência: LoopsBench (arXiv
2608.00267) registra regressão em **todas** as configurações avaliadas: a falha
é de loop, não de geração por unidade.

## Stop conditions (todo loop declara TODAS)

- `acceptance_criteria_passed` — gate objetivo passou.
- `max_attempts_reached` — teto de tentativas.
- `max_same_failure_reached` — mesma assinatura de falha 2×.
- `budget_exhausted` — teto de custo/tokens.
- `timeout_reached`.
- `human_gate_required` — risco/irreversibilidade.

## Loop guards (limites de execução)

Escalada resolve *incapacidade*; guards cortam *runaway* (max rounds · token
budget · sameStop). Princípio: **budget-aware design > model intelligence**.
Toda parada registra `runtime|perfil|turnos|tokens|motivo`.

| Guard | Gatilho (default) | Ação |
|-------|-------------------|------|
| **Teto de turnos** | `economy` 5 · `standard` 10 · `deep`/`critical` 15 tool-calls sem entregar | `economy`/`standard` no teto **não escalam tier automaticamente** — seguem o roteamento pós-falha abaixo; upgrade de perfil só com evidência de incapacidade. `deep`/`critical` no teto → direto **humano**; por isso `long_horizon` não se aplica a eles. `economy`/`standard` com `long_horizon: true` ganham teto 40 turnos, budget $2.00, checkpoint a cada 10 |
| **Budget/task** | `economy` $0.30 · `standard` $0.60 · `deep` $2.00 · `long_horizon` $2.00 | parar, logar, escalar humano. `critical` sem auto-cap (já gated) |
| **Stall / no-progress** | sameStop (output verbatim repetido) · mesmo tool-call (args iguais) 2× · 3 turnos sem arquivo novo/edit/decisão | parar — é loop, não trabalho. Logar como stall (≠ falha) |
| **Hard abort** | 2× teto de turnos OU budget 2× estourado — sob `long_horizon`: 80 turnos ou $4.00 | parar imediato, sem escalar |

## Matriz de verificação

| Tipo | Rota padrão | Escalada |
|------|-------------|----------|
| Build, teste, lint, typecheck, schema | deterministic | nunca para "opinião"; só interpretar falha |
| Interpretação de logs / próxima ação | standard | deep se falha repetida COM evidência nova |
| Diagnóstico causal, trade-off | deep | critical se impacto irreversível |
| Ingest web não confiável | perfil `untrusted-ingest` obrigatório | nunca downgrade por custo |
| Segurança, credenciais, deploy, dados destrutivos | critical + humano | bloqueio seguro |

## Roteamento pós-falha do verifier

Hierarquia obrigatória, nesta ordem — **nunca pular degraus**:

1. Determinístico / tools
2. Mesmo modelo, effort +1 → só se o sintoma for **preguiça** (pulou check, deu done sem verificar)
3. Self-correct mesmo modelo com **evidência exata injetada**
4. Stop / humano — se a mesma `failure_signature` aparecer 2×
5. Upgrade de perfil — **último recurso**, só incapacidade conceitual + evidência nova

| `error_class` | Ação |
|---|---|
| `schema`, `syntax` | repair mesmo modelo, ≤2 ciclos |
| `exec` | repair com a evidência (stderr / saída do check), não com prosa |
| `scope`, `policy` | stop / humano — **nunca** uptier |
| `timeout` do verify | ampliar timeout 1× ou estreitar escopo; senão stop |
| signature repetida 2× | stall stop |

`max_cycles: 2`. Ciclo 3 = stop, não terceiro modelo. **Sem
`EVIDÊNCIA_DE_FALHA` não se entra em self-correct.** Todo self-correct cita a
assertion que falhou (check_id + evidência), a hipótese mudada e a próxima
ação bounded.

**Aviso que justifica a regra:** "critique seu texto e melhore" sem sinal
externo costuma piorar o resultado. O gargalo é feedback confiável, não refino.

### Template de repair

Evidência de execução pede patch mínimo de comportamento; evidência
estruturada de check pede patch mecânico da lista. Misturar faz o modelo
"limpar o arquivo inteiro".

```
OBJETIVO: {verificável}
EVIDÊNCIA_DE_FALHA: {check_id + evidence verbatim}    # obrigatório
JÁ TENTADO: {failure_signatures anteriores}
CONSTRAINTS: {scope.writes, negatives}
TAREFA: corrija APENAS a causa da evidência.
NÃO: refatorar além do necessário; não reabrir escopo; não silenciar o check.
RERUN: {comando idêntico ao do gate}
```

Saída de linter (regra + path + linha) → pack de lista, top-K, uma regra por
ciclo. Falha de execução → primeiro erro real + ~20 linhas, nunca o log inteiro.

### O que NÃO escrever

Esta política é sobre roteamento pós-falha de verificador determinístico.
Fica **proibido**:

- instrução mandando o agente revisar/reconferir o próprio output antes de responder;
- subagente cuja única função é verificar trabalho de outro agente;
- pedir "explique seu raciocínio passo a passo na resposta".

Regra de bolso: **se o gatilho é um exit code, é policy legítima; se o gatilho
é "por via das dúvidas", é over-verification.**

## Contrato EXEC (delegação a executor)

Delegação para executor (subagente, outro modelo, outro harness) segue
contrato máquina: planner escreve spec, executor roda dentro do teto declarado
e devolve report. Toda execução delegada declara `budgets` (max_turns ≤18,
max_tool_calls, max_budget_usd) e `on_budget_exit` (`stop_and_report |
stop_and_revert | stop_and_escalate`).

Saídas canônicas: `end_turn | budget_spent | escalate_human`. **`end_turn` só é
saída com checklist fechada ou bloqueio declarado.** Turno que termina só com
texto é relatório, não done: se há item aberto sem bloqueio, o loop manda
continuação nomeando os itens abertos; máximo 3 continuações, depois
`escalate_human`. Algo em background ainda rodando também não é done.
`budget_spent` → replanejar (dividir a tarefa), não aumentar teto.

## De-escalada (obrigatória)

Subiu de tier pela sub-tarefa difícil → volta ao perfil declarado quando ela
resolve. Não arrastar o volume no tier caro.

## Long-horizon (>15 passos)

Rotina que precisa exceder o teto default declara `long_horizon: true` no
bloco `loop:` — o **único** jeito de rodar >15 turnos sem contar como runaway.

- **Teto sob a flag:** 40 turnos, **checkpoint obrigatório a cada 10** —
  estado externo + re-injeção do `objective` (contra goal-drift).
- **Invariant monitor:** invariantes de segurança checados no fundo.
- Failure modes: *redundant exploration* → stall guard; *premature abandonment*
  → escalada; *constraint/goal drift* → re-injetar objetivo no checkpoint.

## Check testável

Toda rotina iterativa tem bloco `loop:` declarando stop conditions + guards.
Loop sem trigger/procedure/state ou checkpoint sem `evidence_refs`/`next_step`
falha. Negativo plantado: quebrar de propósito um gate de nó já concluído tem
de reprovar o nó corrente; se passar, a obrigação é decorativa.
