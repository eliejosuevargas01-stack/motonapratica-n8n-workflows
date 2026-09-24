# 🔍 Análise de Bugs & Plano de Melhorias — Deep Research Auditor

> **Workflow:** `Deep Research - Auditor` (`Vz7KzzYWavLw3meH`)  
> **Data da análise:** 2026-09-23  
> **Status:** 🟡 Workflow ATIVO mas com bugs de lógica CRÍTICOS

---

## Estrutura do Fluxo

```
POST /webhook/auditor?id_pesquisa=XXX
  │
  → Staggering1 (wait 0-5s)
  → verifica que a fase seja auditar (DataTable: Pesquisa.fase == 'auditar')
  → obtem items com status auditar (DataTable: status_pesquisa where status='auditar')
  → auditando (DataTable: update status='auditando', Processando=true)
  → obtem a informação resumida (DataTable: resultado_scrapper_passos)
  → agrupa por passo_id para o auditor (Code: merge data)
  → auditor1 (Agent: avalia cada subtópico)
      │
      → Output: {passo_id, status: 'ok'|'pendente', informacao_consolidada, queries[]}
      │
  → atualiza status de auditoria || OK (DataTable: status_pesquisa.status = agent.output.status)
  → enquanto tentativa for diferente de 3 continua (If: tentativas <= 3?)
      ├─ TRUE (tentativas <= 3):
      │     → verifica que todos sejam ok (DataTable: passos by id_pesquisa)
      │     → passa apenas quando todos sejam auditar1 (Code: checks status === 'ok')
      │         ├─ Returns [] (empty): downstream stops → Filter → Wait → chama garimpeiro
      │         └─ Returns [{id_pesquisa}]: If2 evaluates
      │             ├─ TRUE: Update fase para redatar → chama redator
      │             └─ FALSE: Update fase para garimpar → (nothing?!)
      │     → Filter → Wait → chama garimpeiro (/webhook/Scrapper)
      │
      └─ FALSE (tentativas > 3):
            → atualiza status de auditoria1 (DataTable: status='ok')
            → Update fase para redatar1 → chama redator
```

**Agent Output:**
- Se aprovado (≥80% confiança): `{status: 'ok', informacao_consolidada, queries: []}`
- Se reprovado: `{status: 'pendente', queries: ['query1', 'query2']}`

---

## Data Tables Usadas

| Tabela | ID | Uso |
|--------|-----|-----|
| `status_pesquisa` | `arAS2xvz1XBxyA57` | Status de cada tópico (pendente, auditar, auditando, ok) |
| `passos` | `arAS2xvz1XBxyA57` (mesmo ID?) | Controle de tentativas (global por id_pesquisa) |
| `Pesquisa` | ? | Controle de fase (planejar, garimpar, auditar, redatar) |
| `resultado_scrapper_passos` | ? | Dados garimpados pelo Scrapper |

---

## 🔴 BUGS CRÍTICOS

### BUG 1 — **Dois Modelos Conectados ao Agent (Comportamento Indefinido)**
**Severidade: ALTA** ⚠️

O agente `auditor1` tem **dois modelos conectados**:
1. `OpenRouter Chat Model6` → `nvidia/nemotron-3-ultra-550b-a55b:free`
2. `OpenAI Chat Model3` → `gemini-2.5-flash`

Ambos conectam via `ai_languageModel`. O n8n não define qual é usado.

**Correção:** Remover `OpenRouter Chat Model6`. Manter apenas `gemini-2.5-flash`.

---

### BUG 2 — **Nome do Code Node vs Lógica Interna (auditando vs ok)**
**Severidade: CRÍTICA** 🚨

O Code node tem:
- **Nome:** `passa apenas quando todos sejam auditar1`
- **Comentário:** "Verifica se 100% dos itens possuem exatamente o status 'auditar'"
- **Código real:** `item.json.status === 'ok'`

```
O código verifica se status === 'ok', NÃO 'auditar'!
```

Isso é um **BUG DE LÓGICA** grave. O nome sugere que testa por `'auditar'`, mas o código testa por `'ok'`.

**Consequência:** O node só passa quando TODOS os itens já estão como `'ok'`, ou seja, quando a auditoria já terminou! Isso inverte a lógica e impede a transição correta.

**Correção:** Trocar para `item.json.status === 'auditar'` ou `['auditar', 'ok'].includes(item.json.status)`.

---

### BUG 3 — **If2: TRUE branch para Redator, FALSE branch Incompleto**
**Severidade: ALTA**

O `If2` avalia se `id_pesquisa exists`. Dois branches:
- **TRUE (index 0):** `Update fase para redatar` → `chama redator`
- **FALSE (index 1):** `Update fase para garimpar` → **não chama nada depois!**

O FALSE branch atualiza a fase para `'garimpar'` mas não dispara o Scrapper!

**Correção:** Adicionar `chama garimpeiro` após `Update fase para garimpar`, ou documentar se é intencional (o Filter+Wait+chama garimpeiro faria isso).

---

## 🟠 BUGS SÉRIOS

### BUG 4 — **tentativas Não é Incrementado no Auditor**
**Severidade: MÉDIA-ALTA**

O `If` `enquanto tentativa for diferente de 3 continua` checa `tentativas <= 3?`, mas o Auditor **não incrementa** `tentativas`. O valor vem da tabela `passos` definida pelo Scrapper.

**Problema:** Se `tentativas` vier como 0 ou 1, o loop pode executar a mesma condição várias vezes sem mudança.

**Sorte:** O Scrapper incrementa `tentativas` a cada chamada, então quando o Auditor chama `garimpeiro`, o valor incrementa. Na volta, `tentativas` terá aumentado.

Mas isso depende IMPLICITAMENTE de coordenção entre workflows — **frágil**.

---

### BUG 5 — **Modelo `nemotron-3-ultra-550b-a55b:free` Instável**
**Severidade: MÉDIA**

O `nvidia/nemotron-3-ultra-550b-a55b:free` via OpenRouter:
- É um alias para "melhor modelo gratuito"
- Muda com o tempo
- Pode ter rate limit severo (50 req/dia free tier)

Aliado ao Bug 1 (2 modelos), cria comportamento imprevisível.

---

### BUG 6 — **Update fase para garimpar sem Call para Scrapper**
**Severidade: MÉDIA-ALTA**

Quando `If2` retorna FALSE (não tem `id_pesquisa`), o fluxo faz:
```
Update fase para garimpar → [FIM - sem chamar Scrapper!]
```

Mas o nome sugere que deveria voltar para o Scrapper:

```
Update fase para garimpar → chama garimpeiro
```

O branch paralelo via Filter+Wait chama `garimpeiro`, mas o FALSE branch do If2 apenas atualiza a fase e para.

---

## 🟡 PROBLEMAS DE QUALIDADE

### BUG 7 — **System Prompt Bem Estruturado**
**Severidade: N/A (positivo)**

O system prompt:
- Define regra de 80% de confiança
- Explica variações naturais aceitáveis
- Define output JSON obrigatório
- Sem changelog debris ✅

---

### BUG 8 — **Staggering com Math.random()**
**Severidade: BAIXA**

`Staggering1: Math.random() * 5` — mesmo padrão dos outros workflows. Não determinístico.

---

### BUG 9 — **Dois Nós "chama redator" Duplicados**
**Severidade: BAIXA-MÉDIA**

Há dois HTTP requests para o mesmo endpoint:
- `chama redator`: `/webhook/gera-relatorio-final`
- `chama redator1`: `/webhook/gera-relatorio-final` (mesmo!)

Um é chamado no branch TRUE do If2, outro no branch FALSE do `enquanto tentativa...`.

**Correção:** Consolidar em um único nó (DRY).

---

## 📋 PLANO DE IMPLEMENTAÇÃO

### Fase 1 — Crítico (corrigir lógica)
> **Prioridade:** IMEDIATA

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 1.1 | **CORRIR BUG DE LÓGICA:** Trocar `status === 'ok'` para `status === 'auditar'` ou `['auditar', 'ok'].includes(status)` | `passa apenas quando todos sejam auditar1` | Baixa |
| 1.2 | Remover `OpenRouter Chat Model6` (manter apenas gemini-2.5-flash) | Conexão do modelo | Baixa |
| 1.3 | Adicionar `chama garimpeiro` após `Update fase para garimpar` no If2 FALSE branch | If2 structure | Baixa |

### Fase 2 — Resiliência
> **Prioridade:** MÉDIA

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 2.1 | Incrementar `tentativas` no Auditor ANTES de chamar garimpeiro | Novo DataTable update | Média |
| 2.2 | Consolidar os dois "chama redator" em um único nó | Estrutura | Baixa |
| 2.3 | Adicionar log/métrica de auditoria aprovada vs reprovada | Nodes novos | Média |

### Fase 3 — Qualidade
> **Prioridade:** BAIXA

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 3.1 | Renomear o Code node para refletir o que realmente faz | `passa apenas quando todos sejam auditar1` | Baixa |
| 3.2 | Adicionar comentário explicando coordenação de `tentativas` com Scrapper | Documentation | Baixa |

---

## Estimativa de Esforço

| Fase | Itens | Esforço |
|------|-------|--------|
| Fase 1 (Crítico) | 3 itens | ~15 min |
| Fase 2 (Resiliência) | 3 itens | ~30 min |
| Fase 3 (Qualidade) | 2 itens | ~10 min |
| **Total** | **8 itens** | **~55 min** |

---

## Nota: O Loop Auditor → Scrapper

```
Auditor → chama garimpeiro → Scrapper
  ↓
Scrapper incrementa tentativas → processa → chama auditor
  ↓
Auditor → If (tentativas <= 3?) → continua loop
```

O loop é limitado por `tentativas <= 3`. Se o Agente rejeitar o tópico 3 vezes, o Auditor desiste e chama o Redator.

Mas a coordenação é IMPLÍCITA — ambos workflows acessam a tabela `passos`. Se alguém mudar o esquema, quebra.
