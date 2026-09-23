# 🔄 Deep Research Refactoring - 3 Fluxos

> Divisão do fluxo monolítico em 3 workflows especializados

## 📊 Visão Geral

O Deep Research atual (88 nodes) será dividido em 3 workflows menores e mais gerenciáveis:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     DEEP RESEARCH SISTEMA                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐
│   1. PLANNER    │───▶│  2. SCRAPPER    │───▶│  3. AUDITOR & WRITER    │
│                 │    │                 │    │                         │
│ - Recebe tema │    │ - Executa buscas│    │ - Audita qualidade      │
│ - Cria plano  │    │ - Faz scraping  │    │ - Escreve relatório     │
│ - Dispara     │    │ - Salva dados   │    │ - Entrega final         │
└─────────────────┘    └─────────────────┘    └─────────────────────────┘
     15-20 nodes            30-35 nodes            35-40 nodes
```

---

## 1️⃣ Moto Na Pratica - Deep Research Planner

### Propósito
Recebe o tema do Diretor SEO, analisa com IA e cria um plano de pesquisa estruturado.

### Input
```json
{
  "tema": "Nova Honda CB500X 2026",
  "request_id": "uuid-gerado",
  "prioridade": "alta"
}
```

### Output
```json
{
  "success": true,
  "request_id": "uuid-gerado",
  "plano": {
    "titulo": "Pesquisa: Nova Honda CB500X 2026",
    "topicos": [
      {
        "id": "t1",
        "nome": "Especificações Técnicas",
        "queries": ["Honda CB500X 2026 especificações", "CB500X 2026 ficha técnica"]
      },
      {
        "id": "t2",
        "nome": "Preço e Lançamento",
        "queries": ["Honda CB500X 2026 preço Brasil", "CB500X 2026 lançamento"]
      }
    ]
  },
  "status": "planejado"
}
```

### Nodes Principais
1. **Webhook** - Recebe tema via POST
2. **Schema Validator** - Valida input JSON
3. **monta o plano** (OpenRouter Chat Model) - IA planeja estrutura
4. **Parser** - Extrai JSON do output da IA
5. **Insert Plano** - Salva no Supabase
6. **chama Scrapper** - Tool Call para workflow 2
7. **Respond to Webhook** - Confirmação

### Banco de Dados
**Tabela: `deep_research_planos`**
```sql
CREATE TABLE deep_research_planos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  request_id TEXT UNIQUE NOT NULL,
  tema TEXT NOT NULL,
  plano_json JSONB,
  status TEXT DEFAULT 'planejado',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

---

## 2️⃣ Moto Na Pratica - Deep Research Scrapper

### Propósito
Executa as buscas baseadas no plano, coleta dados via Google Search, Apify, Jina AI e salva no banco.

### Input
```json
{
  "request_id": "uuid-gerado",
  "plano": { /* objeto do plano */ }
}
```

### Output
```json
{
  "success": true,
  "request_id": "uuid-gerado",
  "topicos_processados": 5,
  "resultados_coletados": 25,
  "status": "coletado"
}
```

### Nodes Principais
1. **Webhook** - Recebe plano via Tool Call
2. **Loop Tópicos** - Processa cada tópico do plano
3. **Get Tópico Pendente** - Busca próximo da fila
4. **Google Search Scrapper** - Busca no Google
5. **Apify Scrapper** - Scraping profundo de sites
6. **Jina AI Extract** - Extrai conteúdo limpo
7. **Embeddings Google Gemini** - Gera embeddings
8. **Postgres PGVector Store** - Salva no vector store
9. **Insert Resultado** - Salva metadados no Supabase
10. **Update Status** - Marca como coletado
11. **Callback Auditor** - Tool Call para workflow 3

### Banco de Dados
**Tabela: `deep_research_resultados`**
```sql
CREATE TABLE deep_research_resultados (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  request_id TEXT NOT NULL,
  topico_id TEXT NOT NULL,
  query TEXT,
  fonte TEXT,
  titulo TEXT,
  url TEXT,
  conteudo TEXT,
  embedding TEXT, -- JSON array
  status TEXT DEFAULT 'coletado',
  nota_auditoria INTEGER,
  motivo_auditoria TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_request_id ON deep_research_resultados(request_id);
CREATE INDEX idx_status ON deep_research_resultados(status);
```

### Serviços Externos
- **Google Search** - SerpAPI
- **Apify** - Atores de scraping
- **Jina AI** - Extração de conteúdo
- **Postgres PGVector** - Vector store

---

## 3️⃣ Moto Na Pratica - Deep Research Auditor & Writer

### Propósito
Audita a qualidade dos dados coletados, verifica relevância e fontes, escreve o relatório final.

### Input
```json
{
  "request_id": "uuid-gerado",
  "tema": "Nova Honda CB500X 2026"
}
```

### Output
```json
{
  "success": true,
  "request_id": "uuid-gerado",
  "status": "relatorio_pronto",
  "relatorio_id": "uuid-relatorio",
  "resumo": "A nova Honda CB500X 2026 traz..."
}
```

### Nodes Principais

#### Fase 1: Auditoria
1. **Webhook** - Recebe notificação de coleta completa
2. **Obtem Pesquisas Realizadas** - Busca todos resultados
3. **Loop Auditoria** - Itera por cada resultado
4. **busca_postgres_pgvector** - Contexto relevante
5. **auditor1** (OpenRouter Chat Model) - IA avalia qualidade
6. **Parser Auditoria** - Extrai decisão (aprovado/rejeitado)
7. **atualiza status de auditoria** - Marca no banco
8. **passa quando todos** - Verifica se terminou

#### Fase 2: Redação
9. **se todos prontos** - Condição para prosseguir
10. **redator final** (OpenRouter Chat Model) - Escreve relatório
11. **obtem relatorio anterior** - Verifica existência
12. **Delete anterior** - Remove versão antiga
13. **Insert relatório** - Salva nova versão
14. **atualiza status geral** - "relatorio_pronto"
15. **Guardrails** - Verificação de segurança
16. **Respond to Webhook** - Entrega callback

### Banco de Dados
**Tabela: `deep_research_relatorios`**
```sql
CREATE TABLE deep_research_relatorios (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  request_id TEXT UNIQUE NOT NULL,
  tema TEXT NOT NULL,
  relatorio TEXT NOT NULL, -- Markdown completo
  fontes JSONB, -- Array de URLs e títulos
  status TEXT DEFAULT 'rascunho',
  nota_media INTEGER,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

---

## 🔄 Fluxo de Comunicação

```
Diretores SEO
      │
      │ POST /deep-research/plan
      │ { tema: "..." }
      ▼
┌─────────────────┐
│ 1. PLANNER      │
│ - Cria plano    │
│ - Salva no BD   │
└────────┬────────┘
         │
         │ Tool Call
         │ POST /deep-research/scrap
         │ { request_id, plano }
         ▼
┌─────────────────┐
│ 2. SCRAPPER     │
│ - Coleta dados  │
│ - Salva vectors │
└────────┬────────┘
         │
         │ Tool Call
         │ POST /deep-research/audit
         │ { request_id }
         ▼
┌─────────────────┐
│ 3. AUDITOR      │
│ - Audita info   │
│ - Escreve doc   │
└────────┬────────┘
         │
         │ Responde ao caller
         │ { success, relatorio }
         ▼
    Escritor Principal
```

---

## 🗄️ Estrutura de Dados (Supabase)

### Relacionamentos
```
deep_research_planos (1)
    │
    ├──► deep_research_topicos (N)
    │       │
    │       └──► deep_research_resultados (N)
    │               │
    │               └──► deep_research_relatorios (1)
    │
    └──► deep_research_relatorios (1)
```

---

## ✅ Checklist de Implementação

### Banco de Dados
- [ ] Criar tabela `deep_research_planos`
- [ ] Criar tabela `deep_research_topicos`
- [ ] Criar tabela `deep_research_resultados`
- [ ] Criar tabela `deep_research_relatorios`
- [ ] Configurar PGVector (se não existir)
- [ ] Criar índices

### Workflows
- [ ] Criar "Moto Na Pratica - Deep Research Planner"
- [ ] Criar "Moto Na Pratica - Deep Research Scrapper"
- [ ] Criar "Moto Na Pratica - Deep Research Auditor"
- [ ] Configurar webhooks
- [ ] Configurar Tool Calls entre workflows
- [ ] Testar fluxo end-to-end

### Credenciais
- [ ] OpenRouter API Key
- [ ] Google Service Account (Vertex AI + Embeddings)
- [ ] SerpAPI Key (Google Search)
- [ ] Apify API Key
- [ ] Jina AI API Key
- [ ] Supabase JWT

### Migração
- [ ] Arquivar workflow antigo (renomear para _legacy)
- [ ] Exportar dados importantes do anterior
- [ ] Testar novo fluxo
- [ ] Monitorar logs

---

## 📊 Comparação: Antes vs Depois

| Aspecto | Fluxo Monolítico | 3 Fluxos Separados |
|---------|-----------------|-------------------|
| **Nodes** | 88 | ~20 + ~35 + ~35 = 90 |
| **Manutenção** | Difícil | Fácil |
| **Escalabilidade** | Tudo junto | Independente |
| **Monitoramento** | Confuso | Granular |
| **Falhas** | Afeta tudo | Isolada |
| **Reusabilidade** | Nenhuma | Scrapper reutilizável |
| **Testes** | Complexos | Modulares |

---

## 🚀 Próximos Passos

1. **Aprovar estrutura** (com usuário)
2. **Criar tabelas no Supabase**
3. **Criar workflow Planner**
4. **Criar workflow Scrapper**
5. **Criar workflow Auditor**
6. **Testar integração**
7. **Migrar do fluxo antigo**
8. **Documentar e treinar**

---

[← Voltar para Arquitetura](/docs/architecture/README.md)
