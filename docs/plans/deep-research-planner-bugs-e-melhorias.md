# 🔍 Análise de Bugs & Plano de Melhorias — Deep Research Planner

> **Workflow:** `Deep Research - Planner` (`op3gvAdtkYjO9ydV`)  
> **Data da análise:** 2026-09-23  
> **Status:** 🔴 Pipeline QUEBRADA — workflow está INATIVO

---

## Estrutura do Fluxo

```
POST /webhook/deep-research
  │
  ├─ query.id (int) — obrigatório
  ├─ body.tema (string) — tema da pesquisa
  ├─ body.webhook_callback (url) — callback URL (não usado!)
  ├─ body.language (string) — idioma
  └─ headers[x-apify-secret] — autenticação

Flow:
  Webhook → schema (If: valida 7 condições)
    ├─ FALSE → Respond to Webhook (202 Accepted) → FIM (mas não retorna erro real!)
    └─ TRUE → Code in JavaScript (validação Custom)
           ├─ FALSE → Respond to Webhook1 (400 error)
           └─ TRUE → Guardrails (sanitiza tema)
                  → pega a pesquisa (DataTable upsert: cria/atualiza Pesquisa)
                  → monta o plano (Agent: STORM methodology)
                  → Code in JavaScript1 (parse JSON array)
                  → Insert row (DataTable: insere topicos)
                  → muda fase da pesquisa (DataTable: update)
                  → Wait2 (staggering 0-3s)
                  → chama garimpeiro2 (HTTP: POST /webhook/Scrapper)
```

**DataTable usada:** `Pesquisa` (`9ltZLFGUXe11LFQk`)

**Output esperado:** 
- Cria 1 registro em `Pesquisa` (fase="planejar")
- Cria 10-18 registros de tópicos (steps) via agent
- Chama o Scrapper (`chama garimpeiro2`) para iniciar a execução

---

## 🔴 BUGS CRÍTICOS (pipeline quebrada)

### BUG 1 — Workflow INATIVO
**Severidade: BLOQUEANTE** 🚨

O workflow tem `active: false`. Isso significa que **ninguém está escutando em `/webhook/deep-research`**.

**Resultado:** Toda vez que os Diretores chamam o tool `Moto na Pratica - Deep Research`:
```
POST https://myn8n.seommerce.shop/webhook/deep-research
```
O n8n retorna **404 Not Found**. A pipeline inteira morre aqui.

**Correção:** Ativar o workflow no n8n.

---

### BUG 2 — Segredo DIFERENTE do Diretores
**Severidade: CRÍTICA** 🔑

O Planner valida o secret:
```javascript
const EXPECTED_SECRET = "bowT7Vwgclr4bUcJd80xZXgyVml2HzjQdCRYNKX8rjiq3TorPWyw3EiHezl";
```

Mas o Diretores envia:
```
x-apify-secret: bowT7Vwgclr4bUcJd80xZXgyVmI2HzjQdCRYNKX8rjiq3TorPWyw3EiHezLK7hygLlsah6ZLvZmuZf2XIjFmwk6UrtWT26a8OnKGwwje9aDEGLiHMU9N6FYhcO04326B
```

**O secret do Planner está TRUNCADO** (termina em `...Hezl` ao invés de `...hO04326B`).

**Resultado:** Mesmo se o workflow estivesse ativo, a validação do secret **FALHARIA** para todas as chamadas vindas dos Diretores.

**Correção:** Atualizar o secret no Planner para o valor correto completo.

---

## 🟠 BUGS SÉRIOS (comportamento incorreto)

### BUG 3 — Dois Modelos Conectados ao Agent (Ambiguidade)
**Severidade: ALTA** ⚠️

O agente `monta o plano` tem **dois modelos conectados**:
1. `OpenRouter Chat Model` → `nvidia/nemotron-3-ultra-550b-a55b:free`
2. `OpenAI Chat Model` → `gemini-3.5-flash`

Ambos conectam via `ai_languageModel`. O n8n não define claramente qual modelo é usado quando há múltiplas conexões do mesmo tipo.

**Resultado:** Comportamento indefinido — pode usar qualquer um dos dois ou falhar silenciosamente.

**Correção:** Remover uma das conexões. Recomendado: usar apenas `gemini-3.5-flash` (mais confiável e estável).

---

### BUG 4 — Campo `webhook_callback` Validado mas NÃO USADO
**Severidade: MÉDIA-ALTA**

O Code node valida:
```javascript
const webhookCallback = item.body?.webhook_callback;
if (webhookCallback === undefined || webhookCallback === null || String(webhookCallback).trim() === '') {
  errors.push("Missing or empty required parameter: 'body.webhook_callback'.");
}
```

Mas o campo **nunca é passado adiante**. O DataTable upsert grava:
- `asssunto` (tema)
- `webhook_callback` ← USA `body.callbackUrl` (nome diferente!)
- `id_pesquisa`
- `Fase`
- `lang`

**Bug:** O webhook espera `body.webhook_callback`, mas o DataTable lê de `body.callbackUrl`. Nome incompatível!

**Resultado:** O campo salvo no DataTable será `undefined`.

**Correção:** Usar `body.webhook_callback` consistentemente.

---

### BUG 5 — If Schema com 7 Condições mas Sem Mensagem de Erro Clara
**Severidade: MÉDIA**

O If node `schema` valida 7 condições:
1. `query.id` exists
2. `body.tema` exists
3. `query.id` notEmpty
4. `body.tema` notEmpty
5. `x-apify-secret` exists
6. `x-apify-secret` notEmpty
7. `x-apify-secret` equals (valor hardcoded)

Se **qualquer uma** falhar, o fluxo vai para FALSE → `Respond to Webhook` (202 Accepted).

**Problema:** Retorna 202 Accepted mesmo quando a validação falha! O cliente (Diretores) acha que recebeu com sucesso, mas nenhum processamento acontece.

**Correção:** O ramo FALSE deveria retornar 400 Bad Request com lista de erros.

---

### BUG 6 — Double Validation (If + Code)
**Severidade: BAIXA-MÉDIA**

Há **duas validações duplicadas**:
1. If `schema` → valida campos + secret
2. Code `JavaScript` → valida os mesmos campos + secret novamente

Isso é redundante e desperdiça execução. Uma das duas deveria ser removida.

---

## 🟡 PROBLEMAS DE QUALIDADE

### BUG 7 — System Prompt com Duplo "Exemplos corretos"
**Severidade: BAIXA**

O system prompt tem a instrução `Exemplos corretos de passo_id:` repetida DUAS vezes:
```
Exemplos corretos de passo_id:
1a
1b
1c
etc...
PASSO ID DEVE SER ESTRICTAMENTE [NUMERO DO PASSO][LETRA DO SUBTOPICO]

🏆 GOLDEN RULES FOR CONTENT (STORM METHOD):
...

Exemplos corretos de passo_id:
1a
1b
1c
etc...
PASSO ID DEVE SER ESTRICTAMENTE [NUMERO DO PASSO][LETRA DO SUBTOPICO]
```

Desperdiça tokens e pode confundir o modelo.

---

### BUG 8 — Modelo `nemotron-3-ultra-550b-a55b:free` (OpenRouter Free)
**Severidade: BAIXA-MÉDIA**

O modelo `nvidia/nemotron-3-ultra-550b-a55b:free` é:
- Gratuito via OpenRouter
- Pode ter rate limits severos
- Qualidade incerta para tarefas de planejamento complexo

Se isso for o modelo atualmente em uso (deptro do comportamento indefinido do Bug 3), a qualidade dos planos pode ser baixa.

---

### BUG 9 — Secret Hardcoded em DOIS Lugares
**Severidade: MÉDIA**

O secret aparece em:
1. If node `schema` → condição `equals`
2. Code node `JavaScript` → `EXPECTED_SECRET`

Se mudar o secret, tem que atualizar dois lugares. Propenso a erro.

---

### BUG 10 — Campo `asssunto` com Typo
**Severidade: BAIXA**

O DataTable salva o tema em um campo chamado `asssunto` (3 S's). Typo.

---

## 📋 PLANO DE IMPLEMENTAÇÃO

### Fase 1 — Crítico (desbloquear pipeline)
> **Prioridade:** IMEDIATA

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 1.1 | **ATIVAR** o workflow no n8n | Config global | 1 clique |
| 1.2 | Corrigir o secret truncado no Code node | `Code in JavaScript` | Baixa |
| 1.3 | Corrigir o secret truncado no If node | `schema` | Baixa |

### Fase 2 — Corrigir Comportamento
> **Prioridade:** ALTA

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 2.1 | Remover um dos dois modelos (manter gemini-3.5-flash) | Conexão `OpenRouter Chat Model` | Baixa |
| 2.2 | Corrigir `webhook_callback` vs `callbackUrl` inconsistency | `pega a pesquisa` DataTable | Baixa |
| 2.3 | Fazer o If FALSE retornar 400 com lista de erros | `Respond to Webhook` | Média |
| 2.4 | Remover validação duplicada (manter apenas Code node) | `schema` If node | Baixa |

### Fase 3 — Qualidade
> **Prioridade:** MÉDIA

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 3.1 | Remover duplicação do "Exemplos corretos de passo_id" | System prompt do agent | Baixa |
| 3.2 | Consolidar secret em um único local (Code node) | Remover do If node | Baixa |
| 3.3 | Corrigir typo `asssunto` → `assunto` | DataTable columns | Baixa |

---

## Estimativa de Esforço

| Fase | Itens | Esforço |
|------|-------|--------|
| Fase 1 (Crítico) | 3 itens | ~5 min |
| Fase 2 (Corrigir) | 4 itens | ~20 min |
| Fase 3 (Qualidade) | 3 itens | ~10 min |
| **Total** | **10 itens** | **~35 min** |

---

## Nota: Integração com Diretores

O fluxo correto seria:

```
Diretores (Agent escolhe pauta)
  ↓ chama tool "Moto na Pratica - Deep Research"
POST /webhook/deep-research?id_pesquisa=XXX
  body: { tema: "...", webhook_callback: "...", language: "pt" }
  headers: { x-apify-secret: "..." }
  ↓
Planner (STORM methodology)
  ↓
Scrapper (web scraping)
  ↓
Auditor (validação)
  ↓
Redator (dossier)
  ↓
callback /webhook/deep-research-callback (Escritor)
```

Mas hoje **toda a pipeline está morta** porque o Planner está INATIVO.
