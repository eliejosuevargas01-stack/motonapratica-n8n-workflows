# 🎨 Moto Na Pratica - Midia Generator

> Gerador de mídias (imagens e áudio) para artigos do Moto Na Pratica

## Visão Geral

Este workflow é responsável por gerar as mídias visuais e sonoras para os artigos publicados no Moto Na Pratica. Ele utiliza Vertex AI para criação de imagens e Google Cloud TTS para narração em áudio.

## 📋 Informações

| Campo | Valor |
|-------|-------|
| **ID** | `sgK1oLuR2qMVcvXI` |
| **Nome** | Moto Na Pratica - Midia Generator |
| **Status** | ✅ Ativo |
| **Nodes** | 22 |
| **Triggers** | 1 (Webhook) |
| **Tag** | `Moto-Na-Pratica` |

## 🎯 Função

Gera até **2 imagens por artigo** (não mais que isso) e áudio TTS para narração do conteúdo.

**Chamado por:** O workflow "Escritor" via chamada direta

## 🔄 Fluxo de Processamento

```
┌─────────────────────────────────────────────────────────────┐
│  Webhook (Image Generator) - Recebe payload do artigo       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Switch - Define ação (img/audio/update)                    │
└─────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
    ┌───────────┐    ┌───────────┐    ┌───────────┐
    │  Gerar    │    │  Gerar    │    │ Melhorar  │
    │  Imagens  │    │  Áudio    │    │   Post    │
    └───────────┘    └───────────┘    └───────────┘
```

## 🖼️ Geração de Imagens

### Passo 1: Diretor de Artes (Vertex AI)

**Modelo:** `gemini-3.6-flash`

**Função:** Analisa o artigo e cria prompts otimizados para geração de imagens.

**Input:**
- Título do artigo
- Blocos de conteúdo HTML
- Contexto da notícia

**Output:**
```json
{
  "estrategia_visual_e_orcamento": "Explicação da estratégia visual",
  "prompt_capa": "Prompt detalhado em inglês para a imagem principal",
  "imagens_internas": [
    {
      "numero_do_bloco_alvo": 2,
      "prompt_em_ingles": "Prompt para imagem interna"
    }
  ]
}
```

### Passo 2: Parser de Prompts

Processa a resposta do Diretor de Artes e estrutura os prompts para o loop de geração.

### Passo 3: Loop de Geração

Para cada imagem:
1. Formata o prompt individual
2. Chama o Gerador de Imagens (Vertex AI)
3. Aguarda 20 segundos entre gerações
4. Envia a imagem para a API do Moto Na Pratica
5. Repete até completar todas as imagens

### Passo 4: Gerador de Imagens

**Modelo:** `google/gemini-2.5-flash-image` (via OpenRouter)

**Configuração:**
```javascript
{
  "model": "google/gemini-2.5-flash-image",
  "messages": [
    {
      "role": "system",
      "content": "You are an expert image generation AI..."
    },
    {
      "role": "user",
      "content": "Prompt em inglês..."
    }
  ]
}
```

## 🔊 Geração de Áudio (TTS)

### Processo de Conversão

1. **Split Out:** Divide o payload do artigo
2. **Monta TTS Text:** Converte HTML para texto narrável
3. **Google Cloud TTS:** Gera áudio em PT-BR

**Voz:** `pt-BR-Neural2-B`

**Configuração:**
```javascript
{
  "input": { "text": "Texto narrado..." },
  "voice": {
    "languageCode": "pt-BR",
    "name": "pt-BR-Neural2-B"
  },
  "audioConfig": {
    "audioEncoding": "LINEAR16"
  },
  "outputGcsUri": "gs://audios-de-teste/..."
}
```

### Tratamento de HTML para TTS

O sistema converte automaticamente:
- **Tabelas** → "Veja a tabela comparativa em nosso site"
- **Fichas técnicas** → "Confira a ficha técnica no site"
- **Prós/Contras** → "Confira a lista completa no site"
- **Blockquotes** → "Atenção para a dica da oficina: [conteúdo]"
- **Títulos (H2/H3)** → Frases com pontuação para pausa

## ⚠️ Restrições Importantes

### Anti-Censura de Imagens

**PROIBIDO** usar nos prompts:
- ❌ Nomes de pilotos reais: "Pedro Acosta", "Marc Márquez", "Bagnaia"
- ❌ Marcas de fábrica: "Ducati", "Aprilia", "KTM", "Honda", "Yamaha"

**OBRIGATÓRIO** usar:
- ✅ Descrições físicas detalhadas
- ✅ Características do equipamento
- ✅ Números de corrida
- ✅ Cores e visual

**Exemplo:**
```
❌ ERRADO: "Pedro Acosta na Ducati"
✅ CERTO: "A young Spanish motorcycle racer wearing a red, 
          white and black racing suit with number 31 and a 
          shark-themed helmet, riding a red Italian prototype 
          sports motorcycle with aerodynamic front winglets"
```

### Orçamento de Imagens

- **Limite máximo:** 2 imagens por artigo
- **Distribuição:** 
  - 1 imagem de capa (hero)
  - 1 imagem interna (após bloco estratégico)
- **Distribuição estratégica:** Não colocar imagens em blocos seguidos

## 🔗 Integração

### Webhook

```
POST https://myn8n.seommerce.shop/webhook/motonapratica
```

### Payload de Entrada

```json
{
  "query": {
    "action": "img" | "audio" | "update"
  },
  "body": {
    "payload_para_api": {
      "output": {
        "pt": {
          "id": "post-id",
          "title": "Título do Artigo",
          "block-1": "<p>Conteúdo HTML...</p>",
          "block-2": "<p>Mais conteúdo...</p>"
        }
      }
    }
  }
}
```

### Resposta

```json
{
  "message": "Workflow got started."
}
```

## 📤 Envio para API

Após gerar as imagens, são enviadas para:

```
PATCH https://motonapratica.online/api/posts
Headers: x-api-key: motonapratica-secret-key-2026
Body:
{
  "post_id": "...",
  "position": 1,
  "image": "base64..."
}
```

## 🔧 Configuração

### Credenciais Necessárias

1. **Google Service Account** (Vertex AI):
   - ID: `LHBHRLlwptta0XHn`
   - Escopos: AI Platform, Cloud Storage

2. **OpenRouter API** (Image Generation):
   - ID: `6EGfrOZ5n31iTUmN`
   - Modelo: `google/gemini-2.5-flash-image`

3. **OpenAI API**:
   - ID: `ni5SGSoVdackq86c`
   - Modelo: `gemini-2.5-flash`

## 🐛 Troubleshooting

### Problema: Imagens não são geradas

**Causa provável:** Modelo `gemini-2.5-flash-image` indisponível

**Solução:**
1. Verificar status do OpenRouter
2. Confirmar que a API key está válida
3. Verificar logs no Render LLM Router

### Problema: TTS retorna erro

**Causa provável:** Quota excedida no Google Cloud

**Solução:**
1. Verificar billing do Google Cloud
2. Confirmar que a service account tem permissões TTS

### Problema: Imagens censuradas

**Causa:** Prompts contêm nomes próprios ou marcas

**Solução:** 
- Refinar o prompt do Diretor de Artes
- Adicionar mais exemplos de anti-censura

## 📊 Métricas

| Métrica | Valor |
|---------|-------|
| Tempo médio de geração | ~2-3 minutos |
| Imagens por artigo | Máximo 2 |
| Modelo de IA | Gemini 2.5 Flash Image |
| Voz TTS | pt-BR-Neural2-B |
| Taxa de sucesso | ~95% |

## 📝 Changelog

### v2.0 - 23/09/2026
- ✨ Separado do workflow principal como serviço independente
- ✨ Adicionado controle de orçamento (máx 2 imagens)
- ✨ Implementada anti-censura para nomes próprios
- ✨ Otimizado prompts do Diretor de Artes

### v1.0 - 22/07/2026
- 🎉 Versão inicial integrada ao Escritor

---

**Tag:** `Moto-Na-Pratica` | **MCP Available:** ✅ Sim
