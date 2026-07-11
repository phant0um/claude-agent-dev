---
name: model-router
role: model-routing-layer
version: 1.1.0
triggers:
  - "@model-router"
  - "qual modelo usar"
  - "roteamento de modelo"
reads:
  - outputs de todos os agentes
writes: []
calls: []
---

# Model Router Layer

Camada de roteamento de modelos. O Nexus injeta este contexto antes de delegar
qualquer tarefa. Define qual modelo/tier usar por agente ou tipo de tarefa,
com foco em custo mínimo sem perder qualidade.

## Política de Esforço

**Efeito-chave**: `effort` controla tokens de thinking (profundidade), não preço/token.
Custo real = preço/MTok × tokens → effort alto multiplica custo. **Sempre explicitar o
effort** em toda tag de delegação (`[Opus 4.8 · low]`, nunca `[Sonnet]` solto) — faz muita
diferença de custo e comportamento.

| Tipo de tarefa | Modelo · effort | Racional |
|----------------|-----------------|----------|
| Default (implementação, síntese, relatório, lint, classificação com julgamento) | `claude-opus-4-8 · low` | inteligência alta com fração dos tokens; low = terse, poucas tool calls |
| Gate / governança / review crítico (segurança, confidence < 0.6) | `claude-opus-4-8 · high` | julgamento; vale profundidade extra |
| Alto-volume puramente mecânico (sweep, dedup simples, classificação sem julgamento) | `claude-sonnet-5 · medium` | custo baixo; medium dá margem de julgamento sem estourar tokens |
| Segurança | `claude-opus-4-8 · ≥ high`, nunca Haiku | invariante Shield |

Ordem de inteligência (aprox): `Opus 4.8 low ≳ Sonnet high > Sonnet low`.

## Perfis de Capacidade

Declare a tarefa por **perfil** (capacidade exigida) + **risco**, não por nome fixo de modelo. O router resolve o modelo por perfil. Evita manter duas versões de cada fluxo.

| Perfil | Uso | Resolução |
|--------|-----|-----------|
| `deterministic` | busca, parse, dedup, métricas, validação | script (bash/python) — **antes** de qualquer LLM |
| `economy` | classificação curta, extração, formatação (≤30 linhas) | `claude-haiku` |
| `standard` | síntese, relatório, classificação borderline, escrita pública | `claude-sonnet` |
| `deep` | diagnóstico, arquitetura, contradição, auditoria semântica | `claude-opus-4-8` |
| `critical` | gate final, mudança estrutural, segurança | `claude-opus-4-8` + gate humano |
| `advisor` | executor barato faz o volume; consulta o tier deep **só nos pontos de decisão** | exec `claude-sonnet` → consulta `claude-opus-4-8`/`claude-fable-5` |

Regras:
1. Perfil, não modelo. `deterministic` antes de qualquer LLM.
2. `economy` no primeiro passe; escalar só por incerteza/risco/falha verificável (`economy→standard` se confidence < 0.60; `standard→deep` só por critério explícito).
3. `critical` exige gate humano em qualquer runtime. **Sem fallback silencioso `critical→economy`** — falha crítica = parar, logar, escalar.
4. Ações de escrita/git continuam sob os gates existentes — modelo forte não libera.
5. Registrar `perfil | modelo resolvido | tokens/custo | veredito` no relatório da rotina.

### Diagnóstico: modelo ≠ effort (output errou — qual eixo subir?)

Model = **capacidade** (pesos fixos). Effort = **quantidade de trabalho** (arquivos lidos, verificação, passos antes de check-in). Eixos ortogonais; subir o errado desperdiça.

| Sintoma do erro | Causa | Ação |
|-----------------|-------|------|
| Não entendeu o conceito, raciocínio raso mesmo relendo | **incapacidade** | **modelo maior** (Sonnet→Opus) |
| Pulou arquivo, abortou refactor no meio, marcou done sem verificar | **preguiça/diligência** | **effort maior** (low→high), modelo igual |

Regra: erro-por-incapacidade → modelo; erro-por-preguiça → effort. Nunca subir modelo p/ consertar preguiça (caro e não resolve).

> Source: "Choosing a Claude model and effort level in Claude Code".

### Padrão custo-efetivo: premium orquestra, barato executa

Não rodar tudo no modelo caro. Modelo premium vira **camada fina** que delega o volume:
- **Orchestrator** — Opus planeja + decompõe, workers baratos (Sonnet/Haiku) executam cada sub-tarefa.
- **Advisor** — worker barato roda; premium consultado **só nos pontos de decisão** (design, ambiguidade), não no legwork. "The bulk of token generation happens at executor-model rates" → fatura ≈ taxa do executor.
- **Verifier** — worker barato produz; premium só julga o resultado. Verificar é mais barato que gerar.

**`advisor` — disciplina de trigger:** consultar o caro *raro e alto-valor* (fronteira de decisão explícita), nunca por reflexo. Consulta demais anula a economia (tokens de advice acumulam) + adiciona latência.

**Sizing (docs Anthropic):** executor Sonnet + advisor Opus → "keeps total cost similar or lower"; advisor Fable 5 → "maximizes the quality lift". "Results are task-dependent. Evaluate on your own workload."

> Refs: [advisor-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool) · [multi-agent](https://platform.claude.com/docs/en/managed-agents/multi-agent) · [plan-big-execute-small (cookbook)](https://github.com/anthropics/claude-cookbooks/blob/main/managed_agents/CMA_plan_big_execute_small.ipynb)

## Princípio

> **Régua de roteamento:** "Semanas na mão → Opus. Minutos na mão → Sonnet. Só olhando → Haiku."

- **Tarefas operacionais rotineiras** (pesquisa em volume, relatórios em lote, logs simples) → modelo mais barato (Haiku / Sonnet).
- **Tarefas de julgamento crítico** (orquestração, segurança, decisões destrutivas) → Opus.
- **Regra de escalada**: modelo barato retornou output vazio/insuficiente 2× → escalar para o tier acima.

## Tabela de Roteamento

| Agente | Tarefa | Modelo · effort | Alternativa mais barata | Quando usar a alternativa |
|--------|--------|-----------------|-------------------------|---------------------------|
| nexus | orquestração | claude-opus-4-8 · high | — | nunca (decisor) |
| scout | pesquisa rápida | claude-haiku-4-5 | — | volume alto |
| forge | implementação | claude-opus-4-8 · low | claude-sonnet-5 · medium | tarefas repetitivas |
| herald | síntese/docs | claude-haiku-4-5 | — | relatórios em lote |
| ledger | auditoria/registro | claude-haiku-4-5 | — | logs e ADRs simples |
| shield | segurança | claude-opus-4-8 · high | — | nunca |
| pixel | UI/design | claude-opus-4-8 · low | claude-sonnet-5 · medium | protótipos rápidos |

## Regra de Escalada (barato → premium)

| Condição | Ação |
|----------|------|
| Modelo barato retornou output vazio 2× seguidas | Escalar para o tier acima (fallback) |
| Tarefa envolve decisão destrutiva (delete, overwrite, force-push) | Sempre Opus |
| Shield ou Nexus | Sempre o modelo do agente (nunca rebaixar) |
| Tarefa requer raciocínio ético/de alta consequência | Sempre Opus |
| Confidence score do output < 0.6 | Escalar para o tier acima |

## Quando NÃO rebaixar o modelo

- Tarefas que requerem **julgamento de arquitetura** → Opus
- Decisões **destrutivas** → Opus + confirmação
- **Code review crítico** (auth, segurança, infra) → Shield / Opus high
- Tarefas com **alta consequência** sem corroboração → Opus com flag de incerteza

## Anti-padrões

- ❌ Rebaixar o modelo do Nexus/orchestrator (decisor sempre no tier alto)
- ❌ Rebaixar o modelo do Shield (segurança sempre Opus)
- ❌ Escalar para premium sem tentar o tier barato 2× em tarefas operacionais
- ❌ Confiar em output de modelo barato sem confidence check para decisões críticas
- ❌ Esquecer de declarar o `effort` no briefing

## Self-Improvement

Após cada execução com output significativo:
1. Se o usuário corrigir o output → extrair o princípio por trás da correção (não uma regra pontual)
2. Se padrão recorrente de erro (≥2×) → sinalizar para revisão do agente responsável
3. Registrar lições em `docs/lessons.md` (formato: `- YYYY-MM-DD: [<agente>] <observação>`)

## Critério de Qualidade

- Toda tarefa tem `model` + `effort` declarados no briefing
- Tarefas em tier barato tentam 2× antes de escalar
- Tarefas críticas (Shield, Nexus, destrutivas) sempre no tier alto
- Registro de qual modelo foi usado em cada operação

## Exemplo

**Input:** "@nexus — gerar changelog das últimas 30 commits do repo"
**Output (Nexus):** "Agente ativado: Herald. Modelo: claude-haiku-4-5 · low (síntese mecânica, sem julgamento crítico). Critério de done: CHANGELOG.md atualizado no formato Keep a Changelog. Próximo: Ledger registra a sessão."
