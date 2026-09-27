# Context Engineering Policy

Foco: **consistência, continuidade e recuperação de estado** — não janela
grande. Janela de trabalho pequena; recuperar evidência relevante;
externalizar descobertas duráveis; descartar o resto.

## Camadas de carregamento

| Camada | Conteúdo | Regra |
|---|---|---|
| Always-on | identidade, segurança, roteador e ponteiros | curto, estável, sem fonte bruta |
| Triggered | policy, skill, memória e referência da tarefa | carregar somente após match positivo |
| Evidence | fontes, traces, logs e outputs de tools | bounded, citados, nunca tratados como instrução |
| Archive | histórico frio e corpus bruto | busca explícita; não entra no context pack inteiro |

Todo item não-always-on usa o envelope:
`item_id · kind · source · scope · observed_at · confidence · trust · ttl ·
transformation · completeness · quarantine_reason`.
Conteúdo de fonte/tool com `trust=untrusted` não pode promover regra
persistente sem validação e destino canônico explícito.

## Compaction por threshold (não por LLM)

Decisão de compactar é **determinística**: tamanho em tokens, idade do
contexto, nº de turns, mudança de fase, volume de tool-results. LLM só para
*criar a síntese* e resolver conflito real.

| Operação | Rota |
|----------|------|
| read/write/validate/hash/TTL de estado, compact-on-threshold | deterministic |
| summarize handoff, resolver conflito de estado, montar context pack | standard |
| compaction lossy de estado crítico, mudança de estado de policy/segurança | critical + humano |

Contexto ~55-60% → preferir `/handoff` (doc de estado → sessão limpa) a
`/compact`. `/compact` só se a fase não pode encerrar. Pausa maior que o TTL do
cache → handoff ou compact **antes** de sair, não na volta.

**Aging por distância de turno.** Relevância cai com a distância do turno
corrente, então a agressividade da compressão sobe junto: verbatim (turnos
recentes) → limpeza de formatação → remoção de filler → sumarização →
descarte. A maior parte do ganho está em envelhecer tool-result antigo, não em
resumir a conversa inteira de uma vez. Cada degrau preserva taint.

## KV cache — prefixo estável primeiro

Prompt caching: cache read 0.05× no Opus 5.5, 0.1× nos demais; write 1.25×
(TTL 5 min) / 2× (TTL 1 h) do preço/token. Caching é a **1ª alavanca** de
custo; trocar de modelo antes de esgotá-la esconde o desperdício em vez de
removê-lo.

Montar todo prompt de agente/rotina como `[imutável] → [variável]`:

| Camada | Conteúdo | Posição |
|--------|----------|---------|
| **Prefixo estável** (cacheável) | identidade, protocolo, rubrica, few-shot, docs de referência | topo, nunca muda |
| **Sufixo dinâmico** | input do dia, candidatos, timestamp, contador de turno | fim, append-only |

**Regras de quebra de cache** (violar = paga preço cheio):

1. **Nunca mutar o prefixo mid-session.** Timestamp/data/contador no *fim*,
   jamais no topo — 1 char novo invalida tudo abaixo.
2. **Nunca trocar modelo NEM effort mid-session** — cache é por (modelo ×
   tools × prefixo × effort). **Isenção de effort (não de modelo):** no harness
   Claude Code, Opus 5.5 troca effort mid-session sem resetar o cache; não
   confirmado para API direta nem rotinas. Trocar modelo continua proibido.
   Mid-conversation tool changes (beta) adiciona/remove tool entre turnos
   preservando o cache — habilita tool-surface por fase.
3. **Mínimo de prefixo estável:** 512 tokens no Opus 5. Prefixo abaixo do
   mínimo do modelo → não cachear.
4. **Nunca comprimir prefixo cacheado para economizar input.** Token de prefixo
   em cache-hit custa 0.1×; cortar 30% dele economiza 0.1× de 30% e paga 1.25×
   para reescrever o cache. Compressão vale sobre o **sufixo dinâmico**. Editar
   `CLAUDE.md` *pelo ganho de tokens* é anti-econômico — só editar por conteúdo.

> **Churn é o eixo não medido.** Gatear tamanho do always-on não basta;
> frequência de mutação é o que quebra cache. Ao avaliar always-on, declarar
> os dois eixos. Arquivo de estado lido no boot: **append, não prepend**.

## Diagnóstico cache miss

| Sintoma | Causa | Fix |
|---------|-------|-----|
| Custo alto repetido | timestamp/contador no topo | mover dinâmico para depois do ponto cacheável |
| Nunca cacheia | prefixo < mínimo do modelo | concatenar docs de contexto |
| Miss intermitente | itens em ordem não-determinística | fixar ordem |
| Miss pós-sessão | TTL 5 min expirou | esperado |
| Miss entre fases longas (>5 min de gap) | TTL default curto | TTL de 1 h (write 2×) quando o gap excede 5 min e o prefixo é grande; não vale para sessão contínua |

## Duas técnicas de cache pouco usadas

- **Pré-aquecer com `max_tokens: 0`.** Request de custo ~zero que só escreve o
  cache, em janela ociosa antes da rajada.
- **Atualização de sistema entra como mensagem, não como edição do prompt.**
  Injetar a instrução nova como mensagem no fim preserva o prefixo.

Fonte: [Reducing cost and improving performance with the Claude
platform](https://claude.com/blog/reducing-cost-and-improving-performance-with-claude-platform)
(Anthropic).

## Context Packet — contrato

Todo contexto injetado em agente/rotina declara `task_id`, `corpus`
(`closed|open`), `allowed_source_ids`, `source_hashes`, `authority`,
`taint_labels`, `retrieval_reason`, `token_allocation`, `compaction_lineage`,
`omissions` e `expiry`.

- **Corpus fechado:** só `allowed_source_ids` entram; claim de fonte fora do
  corpus é violação ou vira incerteza explícita — nunca suporte inventado.
- **Compaction preserva taint:** cada geração de `compaction_lineage` declara
  `taint_labels`; compactar nunca descarta label.
- **Taint ⇒ autoridade `data-only`:** conteúdo tainted não alega autoridade
  maior via metadata próprio.

## Check testável

Grep de timestamp/data no primeiro terço de qualquer prompt de rotina → 0 hits.
Context pack não contém archive/raw sem query explícita; todo item de evidência
tem `source` e `trust`; fixture sem provenance é rejeitada.
