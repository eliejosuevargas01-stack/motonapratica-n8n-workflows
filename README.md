# 🏍️ Moto Na Pratica - N8N Workflows

[![N8N](https://img.shields.io/badge/n8n-Automation-orange)](https://n8n.io)
[![MCP](https://img.shields.io/badge/MCP-Enabled-blue)](https://modelcontextprotocol.io)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

> Sistema de automação de conteúdo para o portal Moto Na Pratica - plataforma de notícias e conteúdo sobre o universo motociclístico.

## 📋 Índice

- [Visão Geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [Workflows](#workflows)
- [Instalação](#instalação)
- [Documentação](#documentação)
- [Contribuição](#contribuição)

## 🎯 Visão Geral

O Moto Na Pratica utiliza uma arquitetura de automação baseada em n8n com 6 workflows especializados que trabalham em conjunto para:

- 📊 Coletar automaticamente dados de eventos MotoGP
- 🎯 Selecionar pautas relevantes via Diretores SEO
- 🔬 Realizar pesquisa profunda sobre temas escolhidos
- ✍️ Gerar artigos otimizados para SEO
- 🎨 Criar mídias (imagens e áudio) para os artigos

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                    Moto Na Prática                         │
│                 Sistema de Automação                       │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  📊 Events   │    │   📝 News    │    │   🛠️ Tools   │
│  Scrapper    │    │   Director   │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
        │                     │
        └─────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │   🔬 Deep Research    │
        │   (Pesquisa Completa) │
        └───────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │   ✍️ Escritor         │
        │   (Gera Artigo)       │
        └───────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │   🎨 Midia Generator  │
        │   (Imagens + Áudio)   │
        └───────────────────────┘
```

## 🔄 Workflows

### 1. 📊 Moto Na Pratica - Events Scrapper & Results
**ID:** `WagAbrHMEF7s0RuH` | **Status:** ✅ Ativo

**Função:** Busca diária dos resultados dos eventos de MotoGP.

- **Trigger:** Agendado (diário)
- **Fonte:** MotoGP.com via Jina AI
- **Saída:** Eventos e resultados salvos no Supabase

[Ver detalhes →](docs/workflows/events-scrapper.md)

---

### 2. 🎯 Moto Na Pratica - Diretores SEO
**ID:** `uaE0rGAbVVxkpD1o` | **Status:** ✅ Ativo

**Função:** Decidem qual pauta será publicada.

- **Dois diretores especializados:**
  - Diretor de **Eventos Específicos**
  - Diretor de **Notícias Gerais**
- **Chamada:** Aciona o Deep Research via Tool Call
- **Objetivo:** Encontrar a melhor notícia do momento

[Ver detalhes →](docs/workflows/diretores-seo.md)

---

### 3. 🔬 Deep Research - Moto Na Pratica
**ID:** `QfkpbzVHz9xpeY9k` | **Status:** ✅ Ativo

**Função:** Pesquisa profunda e completa sobre o tema.

- **Input:** Tema escolhido pelo Diretor SEO
- **Processo:** Pesquisa web, análise de concorrentes, PAA
- **Output:** Dossiê completo para o escritor

[Ver detalhes →](docs/workflows/deep-research.md)

---

### 4. ✍️ Moto Na Pratica - Escritor
**ID:** `pdPyTCISLpV2aBcK` | **Status:** ✅ Ativo

**Função:** Escreve o artigo final a partir do dossiê.

- **Input:** Dossiê do Deep Research
- **Output:** Artigo completo em PT/EN/ES
- **Tag:** `escritor por scrapper`

[Ver detalhes →](docs/workflows/escritor.md)

---

### 5. 🎨 Moto Na Pratica - Midia Generator
**ID:** `sgK1oLuR2qMVcvXI` | **Status:** ✅ Ativo

**Função:** Gera imagens e áudio para os artigos.

- **Limite:** Máximo 2 imagens por artigo
- **AI:** Vertex AI (Diretor de Artes + Gerador de Imagens)
- **TTS:** Google Cloud Text-to-Speech
- **Restrições:**
  - Não usar nomes próprios (anti-censura)
  - Descrições físicas detalhadas
  - Orçamento rigoroso (4 blocos internos máx.)

[Ver detalhes →](docs/workflows/midia-generator.md)

---

### 6. 🔧 Moto Na Pratica - Tools
**ID:** `kzsJmLkAWtfy7d6k` | **Status:** ✅ Ativo

**Função:** Ferramentas compartilhadas entre workflows.

- **Google Trends** (SerpAPI)
- **Apify** para scraping
- **Processamento de dados**

**Chamado via:** Tool Call por outros workflows

[Ver detalhes →](docs/workflows/tools.md)

## 📦 Instalação

### Pré-requisitos

- [n8n](https://n8n.io) v2.18.4+
- Credenciais configuradas:
  - Google Service Account (Vertex AI)
  - SerpAPI (Google Trends)
  - Supabase
  - OpenRouter/OpenAI

### Importar Workflows

1. Acesse seu painel n8n
2. Clique em **"Import from File"**
3. Selecione os arquivos JSON em `/workflows/active/`

Ou use a API:

```bash
curl -X POST "https://seu-n8n/api/v1/workflows" \
  -H "X-N8N-API-KEY: sua-api-key" \
  -H "Content-Type: application/json" \
  -d @workflows/active/escritor.json
```

## 📚 Documentação

- [Arquitetura Completa](docs/architecture/README.md)
- [API de Integração](docs/api/README.md)
- [Troubleshooting](docs/troubleshooting/README.md)
- [MCP Tools](docs/mcp-tools.md)

## 🤝 Contribuição

1. Fork o projeto
2. Crie sua branch (`git checkout -b feature/nova-funcionalidade`)
3. Commit suas mudanças (`git commit -m 'Adiciona nova funcionalidade'`)
4. Push para a branch (`git push origin feature/nova-funcionalidade`)
5. Abra um Pull Request

## 📝 Licença

Este projeto está licenciado sob a licença MIT - veja o arquivo [LICENSE](LICENSE) para detalhes.

---

<p align="center">
  <strong>Moto Na Pratica</strong> - Automatizando o mundo das duas rodas 🏍️
</p>
