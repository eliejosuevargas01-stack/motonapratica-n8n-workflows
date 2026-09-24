# 🔍 Análise de Bugs & Plano de Melhorias — Deep Research Scrapper

> **Workflow:** `Deep Research - Scrapper` (`cwLWc2wG6M8IlcqG`)  
> **Data da análise:** 2026-09-23  
> **Status:** 🟡 Workflow ATIVO mas com bugs de lógica

---

## Estrutura do Fluxo

O workflow tem **3 entry points** via webhook:

### 1. Entry Principal (`/webhook/Scrapper`)
```
POST /webhook/Scrapper
  │
  → Staggering2 (wait 0-5s)
  → verifica que a fase seja garimpar (DataTable: checks Pesquisa.fase)
  → Get topicos pendentes2 (DataTable: status_pesquisa where fase='garimpar' AND Processando=false)
  → Tentativas <= 3?1 (If)
      ├─ TRUE:  Update topicos para processados1 → worker conseguiu atualizar?1 → AGENT
      └─ FALSE: muda status para ok2 (GIVE UP - marca como falho após 3 tentativas)
```

### 2. Entry do Agent Tool (`/webhook/google_search_scrapper`)
```
POST /webhook/google_search_scrapper
  │
  → Edit Fields1 (prepara request)
  → HTTP Request (PROXY_URL + PROXY_SECRET_KEY)
  → texto limpo1 (limpa markdown, remove imagens/links)
  → If (error.status == 429?)
      ├─ TRUE:  Wait5 (0-10s) → [LOOP?] 
      └─ FALSE: Return results to agent
```

### 3. Conexão do Agent
```
Agent: cria as queries para iniciar a busca
  │
  → Tool: Google Search Scrapper1 (chama /webhook/google_search_scrapper)
  → Output: JSON {id_pesquisa, passo_id, topico_principal, descricao_subtopico, informacao_resumida, url_fonte[]}
  │
  → inserta pesquisa no deep research (DataTable: resultado_scrapper_passos)
  → atualiza para auditar (DataTable: status_pesquisa.status='auditar')
  → pega todos os subtopicos (DataTable: all status_pesquisa items)
  → passa apenas quando todos sejam auditar2 (Code: filter)
      ├─ Se NÃO todos prontos → [] (vazio, para execução)
      └─ Se TODOS prontos → [{id_pesquisa, total_subtopicos, status_geral}]
  → If3 (check id_pesquisa exists)
      ├─ TRUE:  Update fase para auditar → chama auditor1 (/webhook/auditor)
      └─ FALSE: chama garimpeiro7 (/webhook/Scrapper) ← DEAD CODE!
```

---

## Data Tables Usadas

| Tabela | Uso |
|--------|-----|
| `Pesquisa` | Controle de fase (planejar, garimpar, auditar) |
| `status_pesquisa` | Status de cada tópico (pendente, auditar, ok) |
| `passos` | Controle de Processando e tentativas |
| `resultado_scrapper_passos` | Resultados da pesquisa (informacao_resumida, url_fonte) |

---

## 🔴 BUGS CRÍTICOS

### BUG 1 — Dois Modelos Conectados ao Agent (Comportamento Indefinido)
**Severidade: ALTA** ⚠️

O agente `cria as queries para iniciar a busca` tem **dois modelos conectados**:
1. `OpenRouter Chat Model1` → `openrouter/free`
2. `OpenAI Chat Model2` → `gemini-3.1-flash-lite`

Ambos conectam via `ai_languageModel`. O n8n não define qual é usado.

**Resultado:** Comportamento imprevisível — pode usar qualquer modelo ou falhar.

**Correção:** Remover `OpenRouter Chat Model1`. Manter apenas `gemini-3.1-flash-lite`.

---

### BUG 2 — Modelo `openrouter/free` Sem Especificação
**Severidade: ALTA** ⚠️

O modelo `openrouter/free` é um alias que muda com o tempo. Pode ser qualquer modelo gratuito disponível no momento, com qualidade variável.

**Problema:** Não há garantia de consistência nas respostas do agent.

---

### BUG 3 — Dead Code: If3 FALSE Branch Nunca Executa
**Severidade: MÉDIA-ALTA**

O If3 tem dois branches:
- **TRUE** (index 0): `Update fase para auditar` → `chama auditor1`
- **FALSE** (index 1): `chama garimpeiro7` (chamada recursiva)

O Code node `passa apenas quando todos sejam auditar2` retorna:
- `[]` (vazio) se nem todos estão prontos → downstream não executa
- `[{id_pesquisa, ...}]` se todos prontos → **SEMPRE tem id_pesquisa**

A condição do If3 é `id_pesquisa exists?`. Como o Code node **sempre** retorna `id_pesquisa` quando retorna algo, o FALSE branch é **dead code** — nunca executa.

**Correção:** Remover o FALSE branch do If3 ou repensar a lógica.

---

## 🟠 BUGS SÉRIOS

### BUG 4 — Loop Infinito Potencial no Retry
**Severidade: MÉDIA**

O retry logic:
```
Tentativas <= 3?
  TRUE: Update passos (Processando=true, tentativas++) → worker conseguiu atualizar?
    TRUE: Agent
    FALSE: Wait 0-9s → Get topicos pendentes (LOOP BACK)
  FALSE: muda status para ok (GIVE UP)
```

O problema: se `worker conseguiu atualizar?1` retornar FALSE repetidamente (porque o update falha), o loop volta para `Get topicos pendentes` que **pega o mesmo tópico novamente** — mas `tentativas` já foi incrementado!

Isso funciona na prática, mas é frágil:
- Se o update falhar parcialmente ( Processando atualizado mas tentativas não ), entra em loop infinito
- Se a DataTable tiver race condition (múltiplos workers), pode pegar o mesmo tópico múltiplas vezes

**Correção:** Adicionar timeout máximo ou contador global de loops.

---

### BUG 5 — 429 Handler Incompleto
**Severidade: MÉDIA**

O If node detecta `error.status == 429`:
```
HTTP Request → If (429?)
  TRUE: Wait5 (0-10s) → ???
```

Mas não há conexão de volta para o `HTTP Request`! O fluxo para depois do Wait.

**Resultado:** Em caso de rate limit (429), o tool handler trava ou perde a requisição.

**Correção:** Loop de retry com backoff exponencial ou retornar erro para o agent tentar novamente.

---

### BUG 6 — Staggering com Math.random() Não Determinístico
**Severidade: BAIXA-MÉDIA**

Todos os Wait nodes usam:
```
={{ Math.random() * N }}
```

Onde `N` varia de 3 a 10 segundos. Isso é bom para evitar thundering herd, mas:
- Não há valor mínimo (pode ser 0 segundos)
- Não há log do tempo esperado (dificulta debug)
- Se o workflow for re-executado durante o wait, o novo valor será diferente

---

## 🟡 PROBLEMAS DE QUALIDADE

### BUG 7 — System Prompt Bem Estruturado
**Severidade: N/A (positivo)**

O system prompt do agent está bem documentado com:
- Regras claras de queries curtas (2-4 palavras)
- Exemplos de errado/certo
- Limite de iterações (10-20 queries)
- Formato de saída JSON obrigatório

✅ **Sem changelog debris** — diferente dos outros workflows!

---

### BUG 8 — PROXY_SECRET_KEY via Variável de Ambiente
**Severidade: N/A (positivo)**

O HTTP Request usa:
```
Authorization: Bearer {{ $json.PROXY_SECRET_KEY }}
```

O secret vem da variável de ambiente/nó anterior, não hardcoded. ✅

---

### BUG 9 — Exclusão de Sites Sociais Hardcoded
**Severidade: BAIXA**

A query do Google Search exclui sites hardcoded:
```
-site:instagram.com -site:facebook.com -site:tiktok.com -site:twitter.com -site:x.com -site:pinterest.com
```

Se quiser adicionar/remover, tem que editar o JSON. Melhor seria uma lista configurável.

---

### BUG 10 — Campo `url_fonte` pode ter URLs inválidas
**Severidade: BAIXA**

O agent retorna `url_fonte: []` como array de strings. Não há validação se as URLs são válidas ou acessíveis.

---

## 📋 PLANO DE IMPLEMENTAÇÃO

### Fase 1 — Crítico (estabilidade)
> **Prioridade:** ALTA

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 1.1 | Remover `OpenRouter Chat Model1` (manter apenas gemini-3.1-flash-lite) | Conexão do modelo | Baixa |
| 1.2 | Remover o FALSE branch do If3 (dead code) ou documentar intenção | `If3` | Baixa |
| 1.3 | Implementar retry loop para 429 handler | `If` + `Wait5` | Média |

### Fase 2 — Resiliência
> **Prioridade:** MÉDIA

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 2.1 | Adicionar timeout/check ao retry loop (máximo X iterações globais) | Lógica retry | Média |
| 2.2 | Adicionar log/métrica do tempo de staggering | `Wait` nodes | Baixa |
| 2.3 | Validar URLs no output antes de salvar | Code node novo | Baixa |

### Fase 3 — Qualidade
> **Prioridade:** BAIXA

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 3.1 | Externalizar lista de sites excluídos para config/env | `HTTP Request` | Baixa |
| 3.2 | Adicionar métricas de sucesso/falha por tópico | DataTable ou logs | Média |

---

## Estimativa de Esforço

| Fase | Itens | Esforço |
|------|-------|--------|
| Fase 1 (Crítico) | 3 itens | ~20 min |
| Fase 2 (Resiliência) | 3 itens | ~30 min |
| Fase 3 (Qualidade) | 2 itens | ~20 min |
| **Total** | **8 itens** | **~70 min** |

---

## Nota: Integração com Pipeline

```
Planner → chama /webhook/Scrapper com {id_pesquisa}
             ↓
Scrapper → processa todos os tópicos (agent + tool)
             ↓
Quando TODOS os tópicos prontos → chama /webhook/auditor
             ↓
Auditor → valida pesquisas
             ↓
Redator → gera dossiê final
             ↓
Escritor → consome callback
```

**Status atual:** O Scrapper está funcional mas com comportamento indefinido no modelo e dead code na lógica de transição.
