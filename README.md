# 🏍️ Moto Na Pratica - N8N Workflows

[![N8N](https://img.shields.io/badge/n8n-Automation-orange)](https://n8n.io)
[![MCP](https://img.shields.io/badge/MCP-Enabled-blue)](https://modelcontextprotocol.io)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

> Sistema de automação de conteúdo para o portal Moto Na Pratica - plataforma de notícias e conteúdo sobre o universo motociclístico.

## 🎯 Visão Geral

O Moto Na Pratica utiliza uma arquitetura de automação baseada em n8n com workflows especializados:

- 📊 Coletar automaticamente dados de eventos MotoGP
- 🎯 Selecionar pautas relevantes via Diretores SEO
- 🔬 Realizar pesquisa profunda (Deep Research em 4 sub-workflows)
- ✍️ Gerar artigos otimizados para SEO em 3 idiomas (PT/EN/ES)
- 🎨 Criar mídias (imagens via Vertex AI e áudio TTS)

## 🔄 Workflows

### Workflows Ativos (`workflows/active/`)

| # | Workflow | ID | Nodes | Arquivo |
|---|---------|-----|-------|---------|
| 1 | 🎯 **Diretores SEO** | `uaE0rGAbVVxkpD1o` | 26 | `diretores-seo.json` |
| 2 | 📊 **Events Scrapper & Results** | `WagAbrHMEF7s0RuH` | 13 | `events-scrapper.json` |
| 3 | 🔬 **Deep Research - Scrapper** | `cwLWc2wG6M8IlcqG` | 28 | `deep-research-scrapper.json` |
| 4 | 🔍 **Deep Research - Auditor** | `Vz7KzzYWavLw3meH` | 25 | `deep-research-auditor.json` |
| 5 | 📝 **Deep Research - Redator** | `WGxdU1MdmoMpdZa3` | 13 | `deep-research-redator.json` |
| 6 | ✍️ **Escritor** | `pdPyTCISLpV2aBcK` | 39 | `escritor.json` |
| 7 | 🎨 **Midia Generator** | `sgK1oLuR2qMVcvXI` | 22 | `midia-generator.json` |
| 8 | 🛠️ **Tools** | `kzsJmLkAWtfy7d6k` | 8 | `tools.json` |

### Workflows Inativos (`workflows/inactive/`)

| # | Workflow | ID | Nodes | Arquivo |
|---|---------|-----|-------|---------|
| 1 | 📋 **Deep Research - Planner** | `op3gvAdtkYjO9ydV` | 15 | `deep-research-planner.json` |

### Backup (`workflows/backup/`)

| # | Workflow | Nodes | Arquivo | Nota |
|---|---------|-------|---------|------|
| 1 | 🔬 **Deep Research (Monolítico)** | 92 | `deep-research-monolith-backup.json` | Versão original antes da refatoração |

**Total: 174 nodes** em 8 ativos + 1 inativo + 1 backup

## 📁 Estrutura

```
workflows/
├── active/          # 8 workflows em produção (JSON completo do n8n)
├── inactive/        # 1 workflow em desenvolvimento
└── backup/          # Backup do Deep Research monolítico (92 nodes)
```

## 📦 Importar Workflows

```bash
curl -X POST "https://seu-n8n/api/v1/workflows" \
  -H "X-N8N-API-KEY: sua-api-key" \
  -H "Content-Type: application/json" \
  -d @workflows/active/escritor.json
```

## 📚 Docs

- [Arquitetura](docs/architecture/README.md)
- [Deep Research Refactoring](docs/architecture/deep-research-refactoring.md)

---

> **Último backup**: 2026-09-23 — Exportação completa de todos os workflows reais do n8n
