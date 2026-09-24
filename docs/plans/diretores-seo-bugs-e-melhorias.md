# 🔍 Análise de Bugs & Plano de Melhorias — Diretores SEO

> **Workflow:** `Moto Na Pratica - Diretores SEO` (`uaE0rGAbVVxkpD1o`)  
> **Data da análise:** 2026-09-23  
> **Status:** 🔴 Pipeline parcialmente quebrado (endpoint Deep Research morto)

---

## Estrutura do Fluxo

O workflow contém **2 pipelines paralelos** dentro do mesmo fluxo:

| Pipeline | Trigger | Fontes de Dados | Modelo |
|----------|---------|-----------------|--------|
| **Diretor 1** (Notícias Gerais) | `noticias` — 06:10 | Google News + Motorsport RSS + Google Trends → Merge → Agent | `poolside/laguna-xs-2.1:free` (via OpenRouter) |
| **Diretor 2** (MotoGP Events) | `noticias1` — 06:10 | Motorsport RSS apenas → Agent | `vertex_ai/gemini-3.6-flash` |

```
Pipeline 1 (Diretor Notícias Gerais):
  noticias (06:10)
    ├─→ scrapping de motosport (RSS) → parser motosport ──────→ Merge[1]
    ├─→ queries para google news → Google_news search → parser → Merge[0]
    └─→ Google_trends_trending_now → parser google trends ───→ Merge[2]
                                                                  │
                                                          cria um id unico
                                                                  │
                                                    Diretor SEO Agent (3 tools)
                                                      ├─ Google Trends (tool)
                                                      ├─ Posts Ja Escritos (tool)
                                                      └─ Deep Research (tool) ← BROKEN

Pipeline 2 (Diretor MotoGP Events):
  noticias1 (06:10)
    └─→ scrapping de motosport1 → parser motosport1 → cria id unico2
                                                           │
                                                 Diretor SEO Agent 2 (3 tools)
                                                   ├─ Google Trends (tool)
                                                   ├─ Posts Ja Escritos (tool)
                                                   └─ Deep Research (tool) ← BROKEN
```

---

## 🔴 BUGS CRÍTICOS (pipeline quebrado)

### BUG 1 — Endpoint Deep Research MORTO
**Severidade: BLOQUEANTE** 🚨

Ambos os diretores chamam via tool call:
```
POST https://myn8n.seommerce.shop/webhook/deep-research
```

Mas **ninguém está escutando nesse endpoint**:
- O monolito `Deep Research - Moto Na Pratica` (`QfkpbzVHz9xpeY9k`) foi **apagado**
- O novo `Deep Research - Planner` (`op3gvAdtkYjO9ydV`) que tem webhook em `/deep-research` está **INATIVO**
- Os outros novos workflows escutam em webhooks diferentes:
  - Scrapper → `/Scrapper`
  - Auditor → `/auditor`
  - Redator → `/gera-relatorio-final`

**Resultado:** Toda vez que o agente chama a tool `Moto na Pratica - Deep Research`, o POST vai para o vazio. O agente recebe erro ou timeout, e nenhuma pesquisa acontece. A pipeline inteira está morta a partir deste ponto.

**Correção:** Ativar o Deep Research Planner (`op3gvAdtkYjO9ydV`) OU reimportar o monolito do backup (`workflows/backup/deep-research-monolith-backup.json`).

---

### BUG 2 — Campo Fantasma `pesquisa_profunda_google`
**Severidade: ALTA** ⚠️

O user prompt do Diretor 1 referencia:
```
Google Search & PAA: {{ JSON.stringify($json.pesquisa_profunda_google) }}
```

Mas o nó `cria um id unico` **nunca define** esse campo. Os campos definidos são:
- `manchetes_google_news` ✅
- `noticias_motogp` ✅
- `tendencias_brasil` ✅
- `id_pesquisa` ✅
- `pesquisa_profunda_google` ❌ NÃO EXISTE

**Resultado:** O agente recebe `Google Search & PAA: undefined` no prompt — desperdiçando tokens e confundindo o modelo sobre dados que deveria ter recebido mas não existem.

**Correção:** Remover a linha `Google Search & PAA: {{ JSON.stringify($json.pesquisa_profunda_google) }} use the google trends news to know` do user prompt do Diretor 1.

---

## 🟠 BUGS SÉRIOS (falha silenciosa / dados incorretos)

### BUG 3 — Data Hardcoded no Google News
**Severidade: MÉDIA**

O nó `Google_news search` tem a query:
```
={{ $json.query }} after:2026-07-01
```

A data `2026-07-01` é **fixa**. Conforme o tempo passa:
- O filtro fica cada vez mais largo (meses de notícias velhas misturadas)
- Quando trocar de ano, vai buscar notícias de mais de 1 ano atrás
- Notícias obsoletas poluem o contexto do agente

**Correção:** Substituir por data dinâmica (últimos 7 dias):
```
={{ $json.query }} after:{{ $now.minus({days: 7}).toFormat('yyyy-MM-dd') }}
```

---

### BUG 4 — Merge Bloqueante sem Fallback
**Severidade: MÉDIA-ALTA**

O `Merge` usa modo `combineByPosition` com 3 inputs obrigatórios:
- Input 0: Google News (SerpAPI)
- Input 1: Motorsport RSS (HTTP)
- Input 2: Google Trends (SerpAPI)

Se **qualquer um** falhar (Motorsport.com fora do ar, SerpAPI com rate limit, timeout), o Merge **nunca dispara** → o Diretor 1 nunca executa → **falha silenciosa total** sem log nem alerta.

**Correção:** Adicionar `onError: continueRegularOutput` em cada nó HTTP source, retornando array vazio como fallback. Ou trocar para `combineAll` / `append` mode para que dados parciais ainda cheguem ao agente.

---

### BUG 5 — Ambos Triggers às 06:10 (Race Condition)
**Severidade: MÉDIA**

Os dois triggers (`noticias` e `noticias1`) disparam no **mesmo minuto**: 06:10.

O Diretor 2 termina muito mais rápido (1 fonte vs 3 fontes + Merge). Se ambos chamam o Deep Research quase simultaneamente:
- Dois dossiês em paralelo no pipeline downstream
- Dois artigos gerados ao mesmo tempo
- Potencial conflito no Escritor/Midia Generator

**Correção:** Separar os horários. Diretor 1 às **06:10**, Diretor 2 às **07:10** (1h de gap).

---

### BUG 6 — `id_pesquisa` com Risco de Colisão
**Severidade: BAIXA-MÉDIA**

```javascript
Math.floor(Math.random() * 1000000)  // range 0–999.999
```

Como ambos os diretores rodam no mesmo segundo, há risco (baixo ~1/1M) de colisão de ID. Para um ID que rastreia toda a pipeline (Deep Research → Escritor → Mídia), deveria ser mais robusto.

**Correção:** Trocar por:
```javascript
Date.now().toString(36) + '-' + Math.random().toString(36).substring(2, 8)
```
Ou usar `crypto.randomUUID()` se disponível no n8n.

---

## 🟡 PROBLEMAS DE QUALIDADE

### BUG 7 — Changelog de Dev no System Prompt
**Severidade: BAIXA**

Os system prompts de **ambos** os agentes terminam com ~987 chars de notas internas:
```
O que mudou e por que vai funcionar melhor:
Step 1: Agora exige que a IA levante pelo menos 3 tópicos candidatos...
Step 2: Foi renomeado para ANTI-DUPLICATION PROTOCOL...
```

Isso consome tokens de contexto sem utilidade para o modelo e pode até confundir a execução.

**Correção:** Remover tudo a partir de "O que mudou e por que vai funcionar melhor".

---

### BUG 8 — Diretor 2 recebe System Prompt Genérico
**Severidade: BAIXA-MÉDIA**

O Diretor 2 (MotoGP Events) usa o **exatamente mesmo system prompt** do Diretor 1 — incluindo referências a "Google News", "general trends" e instruções para analisar "JSON with news, MotoGP, and general trends".

Mas o Diretor 2 só recebe `noticias_motogp` no input. O prompt manda ele analisar dados que não existem na sua execução.

**Correção:** Criar system prompt **dedicado** para o Diretor 2, focado em:
- Análise de resultados de corrida MotoGP
- Transferências de pilotos
- Classificações e polêmicas
- Sem referência a Google News ou Trends gerais (ele não recebe esses dados)

---

### BUG 9 — Modelo Free para Tarefa Complexa
**Severidade: BAIXA-MÉDIA**

O Diretor 1 usa `poolside/laguna-xs-2.1:free` — modelo gratuito via OpenRouter com:
- Limite de 50 req/dia no OpenRouter free
- Qualidade inferior para raciocínio multi-step (3 tool calls sequenciais + análise + decisão)

O Diretor 2 já usa `gemini-3.6-flash` que é significativamente mais capaz.

**Correção:** Equalizar ambos para `gemini-3.6-flash` ou pelo menos `gemini-2.5-flash`. A tarefa de seleção de pauta é crítica (decide o que será publicado) e merece um modelo confiável.

---

### BUG 10 — Secret Exposto no Repositório
**Severidade: MÉDIA**

O header `x-apify-secret` está hardcoded em plaintext nos JSONs dos workflows:
```
bowT7Vwgclr4bUcJd80xZXgyVmI2HzjQdCRYNKX8rjiq3TorPWyw3...
```

Agora está no GitHub. Se o repo for público (ou se tornar), o secret está comprometido.

**Correção:** Mover para Credentials do n8n (não hardcoded no JSON). Para o repo, adicionar nota no README ou usar `.gitattributes` para sanitizar exports.

---

## 📋 PLANO DE IMPLEMENTAÇÃO

### Fase 1 — Crítico (desbloquear pipeline)
> **Prioridade:** IMEDIATA — a pipeline está morta sem isso

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 1.1 | Ativar Deep Research Planner OU reimportar monolito do backup | Externo ao workflow | Baixa |
| 1.2 | Remover `$json.pesquisa_profunda_google` do prompt do Diretor 1 | `Diretor SEO Escolhe a pauta a ser produzida` | Baixa |

### Fase 2 — Resiliência (evitar falhas silenciosas)
> **Prioridade:** ALTA — falhas silenciosas são piores que erros explícitos

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 2.1 | Adicionar `onError: continueRegularOutput` nas 3 fontes HTTP + fallback vazio | `scrapping de motosport`, `Google_news search`, `Google_trends_trending_now` | Baixa |
| 2.2 | Separar horários: Diretor 1 às 06:10, Diretor 2 às 07:10 | `noticias1` | Baixa |
| 2.3 | Trocar `Math.random()` por ID robusto (timestamp + random) | `cria um id unico`, `cria um id unico2` | Baixa |
| 2.4 | Data dinâmica no filtro do Google News (últimos 7 dias) | `Google_news search` | Baixa |

### Fase 3 — Qualidade (melhorar decisões dos agentes)
> **Prioridade:** MÉDIA — melhora a qualidade dos artigos produzidos

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 3.1 | Remover changelog de dev (~987 chars) dos system prompts | Ambos agents | Baixa |
| 3.2 | Criar system prompt dedicado para Diretor 2 (MotoGP Events) | `Diretor SEO Escolhe a pauta a ser produzida2` | Média |
| 3.3 | Equalizar modelos para `gemini-3.6-flash` ou `gemini-2.5-flash` | `OpenAI Chat Model` | Baixa |
| 3.4 | Mover `x-apify-secret` para Credentials do n8n | `Call 'Moto na Pratica'`, `Call 'Moto na Pratica'2` | Baixa |

### Fase 4 — Observabilidade
> **Prioridade:** BAIXA — nice-to-have para monitoramento

| # | Ação | Nós afetados | Complexidade |
|---|------|-------------|-------------|
| 4.1 | Adicionar nó de notificação quando o Merge falhar ou agente retornar erro | Novo nó após agents | Média |
| 4.2 | Salvar decisão do agente em DataTable antes de chamar Deep Research | Novo nó após output parser | Média |

---

## Estimativa de Esforço

| Fase | Itens | Esforço estimado |
|------|-------|-----------------|
| Fase 1 (Crítico) | 2 itens | ~15 min |
| Fase 2 (Resiliência) | 4 itens | ~30 min |
| Fase 3 (Qualidade) | 4 itens | ~45 min |
| Fase 4 (Observabilidade) | 2 itens | ~30 min |
| **Total** | **12 itens** | **~2h** |
