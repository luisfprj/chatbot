# Instruções Gerais do Projeto — Chatbot WhatsApp Atestados

## Contexto do Projeto

Este é um **chatbot WhatsApp** para coleta de atestados médicos. Substitui um formulário manual por uma conversa guiada no WhatsApp. Sem interface web — toda interação via WhatsApp Business API (Meta Cloud API).

- **Backend:** Node.js 20 + TypeScript + Express + Prisma + PostgreSQL
- **Integração:** Meta Cloud API (WhatsApp Business) via webhooks
- **Infra:** Docker Compose + PostgreSQL 16

## Documentação Obrigatória

Antes de gerar código, consulte os documentos do projeto:

- [Arquitetura](../documentos/arquitetura.md) — modelo de dados, fluxo da conversa, integração WhatsApp, máquina de estados
- [Regras de Desenvolvimento](../regras/regras-de-desenvolvimento.md) — padrões de código, segurança, nomenclatura
- [Decisões Compartilhadas](../documentos/decisoes-compartilhadas.md) — decisões transversais adotadas
- [Infraestrutura](../documentos/infraestrutura.md) — Docker, portas, variáveis de ambiente
- [Plano de Desenvolvimento](../documentos/plano-de-desenvolvimento.md) — fases, tarefas e status

## Convenções Críticas

- **Timezone:** `America/Sao_Paulo` como fuso operacional. Persistir com `TIMESTAMPTZ`. Nunca depender do timezone do SO.
- **Autorização por número de telefone:** tabela `authorized_numbers` com role `USER` / `ADMIN`.
- **Webhook seguro:** verificar `X-Hub-Signature-256` em todo POST recebido.
- **Schema controlado por Prisma Migrations.** Nunca alterar banco manualmente.
- **Validação com Zod:** toda entrada do usuário validada antes de processar.
- **Identificador padrão:** `BIGSERIAL`.
- **E-mail normalizado:** lowercase antes de persistir.

## Arquitetura de Camadas (Backend)

```
Webhook Handler → Service → Prisma (Repository) → Validators (Zod)
```

- Handler: recebe webhook, roteia e delega. Sem lógica de negócio.
- Service: regras de negócio, orquestração da conversa, acesso a dados via Prisma.
- Validator: schemas Zod puros, sem dependência de banco.

## Estrutura de Pastas

- Backend: `/api/src/` — pastas: config, handlers, services, validators, types, utils
- Prisma: `/api/prisma/` — schema.prisma, migrations/, seed.ts
- Documentos: `/documentos/`
- Regras: `/regras/`

## Segurança (Resumo)

- SQL: nunca concatenar strings. Usar Prisma Client (parametrizado).
- Webhook: verificar assinatura `X-Hub-Signature-256` com `WHATSAPP_APP_SECRET`.
- Tokens/secrets: variáveis de ambiente (`.env`), nunca no código, nunca logados.
- Autorização: apenas números cadastrados com `active = true` interagem.
- Admin: comandos como `/exportar` restritos a role `ADMIN`.

## Git

- Branches: `develop` (desenvolvimento ativo) e `main` (código estável).
- Desenvolver em `develop`, merge para `main` quando estável.

## Ao Planejar Próximas Etapas

Sempre considere: as instruções do agente (`.github/instructions/` e `.github/prompts/`) estão atualizadas com as decisões mais recentes? Se alguma regra, convenção ou padrão mudou, atualize os arquivos correspondentes para que o agente continue alinhado ao projeto.
