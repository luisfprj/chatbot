# Plano de Desenvolvimento — Chatbot WhatsApp Atestados

> **Projeto:** chatbot-atestado
> **Data de criação:** 13/03/2026
> **Última atualização:** 13/03/2026
> **Status geral:** Fase 0 — Planejamento concluído

---

## Documentos Relacionados

| Documento | Descrição |
| --------- | --------- |
| [arquitetura.md](arquitetura.md) | Arquitetura do sistema, modelo de dados, fluxo da conversa, integração WhatsApp |
| [infraestrutura.md](infraestrutura.md) | Docker, portas, variáveis de ambiente, comandos úteis |
| [decisoes-compartilhadas.md](decisoes-compartilhadas.md) | Decisões transversais do projeto |
| [regras-de-desenvolvimento.md](../regras/regras-de-desenvolvimento.md) | Padrões de código, segurança, nomenclatura, checklist |
| [projeto.txt](../projeto.txt) | Definições e decisões originais do projeto |
| [.github/](../.github/) | Instruções e prompts para o agente Copilot |

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Stack Tecnológica](#2-stack-tecnológica)
3. [Estrutura do Repositório](#3-estrutura-do-repositório)
4. [Fases de Desenvolvimento](#4-fases-de-desenvolvimento)
5. [Evolução e Histórico](#5-evolução-e-histórico)

---

## 1. Visão Geral

O **chatbot-atestado** é um chatbot WhatsApp para coleta de atestados médicos. Substitui um formulário manual por uma conversa guiada no WhatsApp, persiste os dados em PostgreSQL e permite exportação para Excel por administradores.

### Princípios

- **Simples e direto:** um chatbot com fluxo linear, sem over-engineering.
- **WhatsApp como interface única:** sem frontend web, toda interação via WhatsApp Business API (Meta Cloud API).
- **Dados no banco:** registros persistidos em PostgreSQL, Excel gerado sob demanda.
- **Segurança por número:** apenas números autorizados podem interagir com o bot. Admins têm comandos especiais.

### Decisões de projeto

- **Stack:** Node.js + TypeScript + Prisma + PostgreSQL.
- **Integração:** Meta Cloud API (WhatsApp Business) via webhooks.
- **Autorização:** por número de telefone cadastrado no banco com role (`USER` / `ADMIN`).
- **Exportação:** admin envia `/exportar` → bot gera `.xlsx` e envia como documento no WhatsApp.
- **Timezone:** `America/Sao_Paulo` como fuso operacional.

---

## 2. Stack Tecnológica

### Backend

| Tecnologia | Descrição |
| ---------- | --------- |
| Node.js | 20+ |
| TypeScript | 5+ |
| Express | Servidor HTTP + webhook routes |
| Prisma | ORM e migrations |
| Zod | Validação de dados (server-side) |
| ExcelJS | Geração de planilhas .xlsx |
| Vitest | Testes unitários e integração |

### Infraestrutura

| Tecnologia | Descrição |
| ---------- | --------- |
| PostgreSQL | 16, banco de dados |
| Docker | Containerização |
| Docker Compose | Orquestração local |

### Integração

| Tecnologia | Descrição |
| ---------- | --------- |
| Meta Cloud API | WhatsApp Business API (webhooks + send messages) |
| ngrok | Túnel HTTPS para desenvolvimento local |

---

## 3. Estrutura do Repositório

```text
chatbot-atestado/
├── api/
│   ├── src/
│   │   ├── config/
│   │   ├── handlers/
│   │   ├── services/
│   │   ├── validators/
│   │   ├── types/
│   │   ├── utils/
│   │   └── app.ts
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── migrations/
│   ├── package.json
│   ├── tsconfig.json
│   └── Dockerfile
├── documentos/
├── regras/
├── docker-compose.dev.yml
├── docker-compose.yml
├── projeto.txt
├── .gitignore
└── README.md
```

---

## 4. Fases de Desenvolvimento

### Fase 1 — Infraestrutura e Ambiente de Desenvolvimento

**Objetivo:** preparar o repositório, ambiente de desenvolvimento e banco local.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 1.1 | Criar estrutura de diretórios | `/api/src`, `/api/prisma`, `/documentos`, `/regras` | ⬜ |
| 1.2 | Inicializar projeto Node.js + TypeScript | `package.json`, `tsconfig.json`, dependências base | ⬜ |
| 1.3 | Configurar Prisma | `schema.prisma` com provider PostgreSQL, config de datasource | ⬜ |
| 1.4 | Criar `docker-compose.dev.yml` | PostgreSQL 16 Alpine, volume, porta 5432, healthcheck | ⬜ |
| 1.5 | Criar `.env.example` | Variáveis de banco, WhatsApp API, timezone | ⬜ |
| 1.6 | Configurar scripts | `dev`, `build`, `start`, `db:migrate`, `db:seed` | ⬜ |
| 1.7 | Criar `.editorconfig` | Padronizar indentação e charset | ⬜ |
| 1.8 | Criar `README.md` | Setup, pré-requisitos, variáveis, comandos úteis | ⬜ |
| 1.9 | Atualizar `.gitignore` | Node.js, Prisma, .env, dist | ✅ |

**Critério de conclusão:** `docker compose -f docker-compose.dev.yml up` sobe PostgreSQL; `npm run dev` inicia o servidor na porta 3000.

### Fase 2 — Integração WhatsApp Business API

**Objetivo:** webhook funcional recebendo e respondendo mensagens no WhatsApp.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 2.1 | Criar servidor Express | `app.ts` com rota de webhook (`GET` para verificação, `POST` para mensagens) | ⬜ |
| 2.2 | Implementar verificação de webhook | Resposta ao challenge da Meta com `WHATSAPP_VERIFY_TOKEN` | ⬜ |
| 2.3 | Implementar verificação de assinatura | Validar `X-Hub-Signature-256` com `WHATSAPP_APP_SECRET` | ⬜ |
| 2.4 | Implementar `whatsapp.service.ts` | Enviar mensagem de texto via Meta API | ⬜ |
| 2.5 | Implementar envio de documento | Upload de media + envio de documento via Meta API | ⬜ |
| 2.6 | Testar com ngrok | Configurar túnel HTTPS, registrar webhook na Meta | ⬜ |
| 2.7 | Echo bot | Bot responde com eco da mensagem recebida (teste de integração) | ⬜ |

**Critério de conclusão:** enviar mensagem no WhatsApp → bot responde com eco. Documentos podem ser enviados via API.

### Fase 3 — Autorização e Controle de Acesso

**Objetivo:** apenas números autorizados interagem com o bot.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 3.1 | Migration: `authorized_numbers` | Tabela com `phone_number`, `name`, `role`, `active`, timestamps | ⬜ |
| 3.2 | Seed: números iniciais | Inserir pelo menos um admin e um usuário de teste | ⬜ |
| 3.3 | Implementar `auth.service.ts` | Verificar se número está autorizado e ativo, retornar role | ⬜ |
| 3.4 | Rejeitar não autorizados | Mensagem educada para números não cadastrados | ⬜ |
| 3.5 | Identificar admin | Permitir comandos admin (`/exportar`) apenas para role `ADMIN` | ⬜ |

**Critério de conclusão:** número não cadastrado recebe mensagem de rejeição. Número cadastrado recebe resposta do bot. Admin é identificado corretamente.

### Fase 4 — Fluxo de Conversa (Máquina de Estados)

**Objetivo:** conversa guiada coleta todos os dados do atestado.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 4.1 | Migration: `conversations` | Tabela com `phone_number`, `current_step`, `data` (JSONB), timestamps | ⬜ |
| 4.2 | Implementar `conversation.service.ts` | Máquina de estados: criar, avançar, recuperar conversa ativa | ⬜ |
| 4.3 | Etapa: dados gerais | Unidade, matrícula, nome, e-mail | ⬜ |
| 4.4 | Etapa: tipo e local | Tipo de atendimento, local, clínica/hospital, médico/dentista | ⬜ |
| 4.5 | Etapa: decisão afastamento | Pergunta SIM/NÃO, condicional para fluxo horas ou dias | ⬜ |
| 4.6 | Fluxo: atendimento por horas | Data, hora início, hora término | ⬜ |
| 4.7 | Fluxo: afastamento por dias | Data inicial, hora início, hora término, duração, data retorno, CID | ⬜ |
| 4.8 | Etapa: confirmação | Resumo dos dados, confirmação ou cancelamento | ⬜ |
| 4.9 | Validadores Zod | E-mail, datas (DD/MM/AAAA), horas (HH:MM), número inteiro positivo | ⬜ |
| 4.10 | Timeout de conversa | Expirar conversas sem interação por 30 minutos | ⬜ |

**Critério de conclusão:** conversa completa no WhatsApp coleta todos os dados, valida entradas e exibe resumo para confirmação.

### Fase 5 — Persistência dos Atestados

**Objetivo:** atestados confirmados são salvos no banco.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 5.1 | Migration: `atestados` | Tabela com todos os campos do objeto de atendimento | ⬜ |
| 5.2 | Implementar `atestado.service.ts` | Salvar atestado completo a partir dos dados da conversa | ⬜ |
| 5.3 | Confirmação ao usuário | Mensagem de sucesso após registro | ⬜ |
| 5.4 | Iniciar nova conversa | Após finalizar, permitir novo registro na próxima mensagem | ⬜ |

**Critério de conclusão:** atestado confirmado é salvo no PostgreSQL. Registro é consultável via SQL.

### Fase 6 — Exportação Excel

**Objetivo:** administradores exportam atestados para Excel via comando no WhatsApp.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 6.1 | Implementar `export.service.ts` | Gerar `.xlsx` com ExcelJS conforme mapeamento de colunas | ⬜ |
| 6.2 | Mapear colunas | Headers da planilha conforme especificação (seção 5 da arquitetura) | ⬜ |
| 6.3 | Comando `/exportar` | Reconhecer comando, verificar role admin, gerar e enviar planilha | ⬜ |
| 6.4 | Filtros opcionais | `/exportar` sem filtro = todos; `/exportar 03/2026` = mês; etc. | ⬜ |
| 6.5 | Envio do arquivo | Upload de media + envio como documento via WhatsApp | ⬜ |

**Critério de conclusão:** admin envia `/exportar` → recebe arquivo `.xlsx` com os atestados registrados, colunas corretas e uma linha por atendimento.

### Fase 7 — Testes e Finalização

**Objetivo:** testes automatizados, documentação final e estabilidade.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 7.1 | Configurar Vitest | Setup do test runner | ⬜ |
| 7.2 | Testes unitários — validadores | Testar schemas Zod (e-mail, datas, horas, SIM/NÃO) | ⬜ |
| 7.3 | Testes unitários — máquina de estados | Testar transições, dados parciais, timeout | ⬜ |
| 7.4 | Testes unitários — geração Excel | Testar colunas, dados condicionais, formatação | ⬜ |
| 7.5 | Teste integração — fluxo completo | Simular conversa completa → salvar → exportar | ⬜ |
| 7.6 | Dockerfile | Multi-stage para produção, runtime não-root | ⬜ |
| 7.7 | `docker-compose.yml` | Serviços: PostgreSQL + API, variáveis via `.env` | ⬜ |
| 7.8 | Health check da API | Endpoint `/health` para Docker | ⬜ |
| 7.9 | README.md final | Setup completo, pré-requisitos, comandos, variáveis | ⬜ |
| 7.10 | Revisar documentos | Atualizar plano, arquitetura, regras, infraestrutura | ⬜ |

**Critério de conclusão:** `npm test` passa. `docker compose up --build` sobe o sistema completo. Bot funcional no WhatsApp.

---

## 5. Evolução e Histórico

| Data | Fase | Descrição |
| ---- | ---- | --------- |
| 13/03/2026 | Fase 0 | Planejamento concluído — projeto adaptado do starter-kit genérico para chatbot WhatsApp de atestados |

---

> **Próximo passo:** iniciar **Fase 1 — Infraestrutura e Ambiente de Desenvolvimento**.