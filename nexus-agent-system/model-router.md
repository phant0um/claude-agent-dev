# Model Router

Rotação de modelos para economia de tokens com custo-benefício (custo ×
inteligência × risco). Não é agente invocável — tabela de roteamento lida pelo
[Nexus](../agents/nexus.md). Rotina declara *perfil*; o router resolve o modelo.

## Modelo por agente

Fonte: o frontmatter de cada agente (`model:` + `effort:`), que o Claude Code
aplica no spawn.

| Agente | Modelo | Effort | Por quê |
|--------|--------|--------|---------|
| [nexus](../agents/nexus.md) | `claude-opus-5-5` | medium | orquestração é julgamento |
| [shield](../agents/shield.md) | `claude-opus-5-5` | **high** | security-scope: vuln perdido ≫ custo; único override de `high` |
| [scout](../agents/scout.md) | `claude-opus-5-5` | medium | pesquisa factual exige baixa alucinação |
| [forge](../agents/forge.md) | `claude-opus-5-5` | medium | executor de escopo fechado |
| [hill](../agents/hill.md) | `claude-opus-5-5` | medium | diagnóstico de falha |
| [finder](../agents/finder.md) | `claude-haiku-4-5` | low | grep/glob; o modelo só escolhe padrão |

## Hard Rules (não-negociáveis)

1. Ingest de web/fonte não confiável → perfil **`untrusted-ingest`** (Sonnet 5),
   nunca Haiku.
2. Sem escalada sem evidência nova.
3. Sem downgrade silencioso de alto risco.
4. Verifier determinístico antes de julgamento LLM.
5. Segurança → Opus + gate humano, nunca Haiku.
6. Opus 5.5 roteia `medium`; effort `high` só como advisor (uma consulta por
   fase, sem execução) ou security-gate; nada acima de `high`. Sonnet 5 roteia
   só `low`, só em `untrusted-ingest`.

## Ordem de decisão e escalada

- Loop guards, stop conditions e de-escalada: [loop-engineering](policies/loop-engineering.md).
- Prefixo estável e KV cache — caching é a 1ª alavanca de custo, antes de trocar
  de modelo: [context-engineering](policies/context-engineering.md).
- Advisor pattern e sizing de subagente: [subagent-orchestration](policies/subagent-orchestration.md).
- Nunca trocar modelo mid-thread: o cache é por (modelo × tools × prefixo × effort).
