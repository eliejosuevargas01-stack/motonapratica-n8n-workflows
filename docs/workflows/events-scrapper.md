# 📊 Moto Na Pratica - Events Scrapper & Results

> Coleta diária de resultados de eventos MotoGP

## Visão Geral

Este workflow realiza a coleta automatizada de calendário de eventos e resultados das corridas de MotoGP, salvando os dados no banco para uso em artigos.

## 📋 Informações

| Campo | Valor |
|-------|-------|
| **ID** | `WagAbrHMEF7s0RuH` |
| **Nome** | Moto Na Pratica - Events Scrapper & Results |
| **Status** | ✅ Ativo |
| **Triggers** | 1 (Agendado) |
| **Tag** | `Moto-Na-Pratica` |

## 🎯 Função

- Extrair calendário de eventos do MotoGP
- Coletar resultados de corridas finalizadas
- Atualizar banco de dados com novos dados

## 🔄 Processo

1. **Jina AI** → Extrai markdown do site da MotoGP
2. **Parser** → Extrai ID da temporada mais recente
3. **HTTP Request** → Busca calendário completo
4. **Supabase** → Salva no banco de dados

## 📅 Agendamento

Executa diariamente para capturar novos resultados.

---

**Tag:** `Moto-Na-Pratica` | **MCP Available:** ✅ Sim
