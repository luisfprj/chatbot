---
applyTo: "api/src/**/*.ts"
description: "Use when editing Node.js/TypeScript backend code. Covers layered architecture rules, service patterns, validation, security, and common violations like logic in handlers or missing validation."
---
# Regras para Código TypeScript — Backend

## Arquitetura de Camadas

Respeitar rigorosamente: **Webhook Handler → Service → Prisma → Validator (Zod)**.

### Handler (Webhook Routes)

- Apenas recebe, extrai dados do payload e delega para Services.
- **Nunca** conter lógica de negócio.
- Sempre responder `200` ao webhook da Meta rapidamente.
- Verificação de assinatura deve ocorrer como middleware, antes do handler.

### Service

- Um service por domínio: `conversation`, `atestado`, `export`, `whatsapp`, `auth`.
- Contém toda a lógica de negócio e orquestração.
- Acesso a dados exclusivamente via Prisma Client.
- Validação de entrada via schemas Zod.
- E-mail normalizado para lowercase antes de persistir ou consultar.
- Regras temporais devem usar timezone configurado (`America/Sao_Paulo`), nunca default do SO.

### Validator (Zod)

- Schemas Zod são puros — sem acesso a banco ou serviços.
- Mensagens de erro em português.
- Um arquivo de validator por domínio ou grupo de validações.

### Types

- Interfaces e tipos compartilhados ficam em `types/`.
- Evitar `any` — usar tipos específicos ou `unknown`.

## Violações Comuns a Evitar

| Violação | Correção |
|----------|----------|
| Lógica de negócio no Handler | Mover para Service |
| Falta de validação Zod | Adicionar schema antes de processar |
| Concatenação SQL manual | Usar Prisma Client |
| `any` desnecessário | Definir tipo correto |
| Token/secret hardcoded | Usar variável de ambiente via config |
| Webhook sem verificação de assinatura | Adicionar middleware de verificação |
| Número não verificado | Checar `authorized_numbers` antes de processar |

## Estrutura de Diretórios

Cada arquivo deve ficar no diretório correto conforme responsabilidade:

```
api/src/
├── config/          → env, constantes, database client
├── handlers/        → rotas Express, middleware de webhook
├── services/        → lógica de negócio (um por domínio)
├── validators/      → schemas Zod
├── types/           → interfaces e tipos
├── utils/           → funções utilitárias
└── app.ts           → setup Express
```

## Nomenclatura

- Arquivo: `camelCase` + sufixo descritivo (ex: `conversation.service.ts`, `webhook.handler.ts`)
- Interface/Type: `PascalCase` (ex: `AtestadoData`, `ConversationStep`)
- Variável/função: `camelCase` (ex: `handleMessage`, `sendTextMessage`)
- Constante: `UPPER_SNAKE_CASE` (ex: `MAX_CONVERSATION_TIMEOUT`)
- Enum valores: `UPPER_SNAKE_CASE` (ex: `UNIDADE_TRABALHO`, `CONFIRMACAO`)
