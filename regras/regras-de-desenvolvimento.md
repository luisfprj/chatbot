# Regras de Desenvolvimento — Chatbot WhatsApp Atestados

> **Projeto:** chatbot-atestado
> **Data de criação:** 13/03/2026
> **Última atualização:** 13/03/2026

Documento complementar: [../documentos/decisoes-compartilhadas.md](../documentos/decisoes-compartilhadas.md)

---

## 1. Princípios Gerais

- **Simplicidade:** evitar over-engineering. O chatbot é um fluxo linear.
- **Consistência:** seguir os padrões definidos neste documento.
- **Segurança:** validar toda entrada do usuário, verificar assinatura de webhooks.
- **Reprodutibilidade:** o projeto deve funcionar localmente de forma previsível com Docker.

---

## 2. Estrutura de Pastas — Obrigatória

### Backend (`/api`)

```text
src/
├── config/          → variáveis de ambiente, constantes
├── handlers/        → rotas Express e handlers de webhook
├── services/        → lógica de negócio (um service por domínio)
├── validators/      → schemas Zod de validação
├── types/           → interfaces e tipos TypeScript
├── utils/           → funções utilitárias
└── app.ts           → setup do servidor Express
```

**Regra:** cada módulo deve ficar no diretório correto conforme sua responsabilidade:
- Handlers não contêm lógica de negócio — apenas roteiam e delegam.
- Services contêm toda a lógica de negócio e orquestração.
- Validators são puros — sem acesso a banco ou serviços.

### Prisma (`/api/prisma`)

```text
prisma/
├── schema.prisma
├── migrations/
└── seed.ts
```

---

## 3. Nomenclatura

### TypeScript

| Elemento | Convenção | Exemplo |
| -------- | --------- | ------- |
| Arquivo de service | camelCase + `.service.ts` | `conversation.service.ts` |
| Arquivo de handler | camelCase + `.handler.ts` | `webhook.handler.ts` |
| Arquivo de validator | camelCase + `.validator.ts` | `atestado.validator.ts` |
| Arquivo de tipo | camelCase + `.types.ts` | `whatsapp.types.ts` |
| Interface/Type | PascalCase | `AtestadoData`, `ConversationStep` |
| Variável/função | camelCase | `handleMessage`, `sendTextMessage` |
| Constante | UPPER_SNAKE_CASE | `MAX_CONVERSATION_TIMEOUT` |
| Enum (valores) | UPPER_SNAKE_CASE | `UNIDADE_TRABALHO`, `CONFIRMACAO` |

### Banco de dados

| Elemento | Convenção | Exemplo |
| -------- | --------- | ------- |
| Tabela | snake_case, plural | `authorized_numbers`, `atestados` |
| Coluna | snake_case | `phone_number`, `created_at` |
| Índice | `idx_{tabela}_{coluna}` | `idx_atestados_phone` |
| Índice unique | `ux_{tabela}_{coluna}` | `ux_authorized_numbers_phone` |
| Migration | descrição em snake_case | `create_authorized_numbers` |

---

## 4. Segurança

### 4.1 SQL Injection

- Nunca concatenar strings para montar queries.
- Sempre usar Prisma Client (queries parametrizadas automaticamente).
- SQL puro apenas em migrations Prisma, com valores literais.

### 4.2 Webhook

- Verificar `X-Hub-Signature-256` em toda requisição `POST` do webhook.
- Usar `WHATSAPP_APP_SECRET` para computar o HMAC.
- Rejeitar requests com assinatura inválida ou ausente.

### 4.3 Autorização

- Apenas números cadastrados na tabela `authorized_numbers` com `active = true` podem interagir.
- Comandos admin (`/exportar`) só funcionam para números com role `ADMIN`.
- Números desconhecidos recebem mensagem de rejeição educada.

### 4.4 Variáveis Sensíveis

- Tokens da Meta API via variáveis de ambiente (`.env`).
- `.env` nunca versionado — apenas `.env.example`.
- Nunca logar tokens ou secrets.

### 4.5 Validação de Entrada

- Toda entrada do usuário via WhatsApp deve ser validada com Zod antes de processar.
- E-mail normalizado para lowercase antes de persistir.
- Datas validadas com formato `DD/MM/AAAA`.
- Horas validadas com formato `HH:MM`.
- Valores SIM/NÃO aceitar variações: `sim`, `s`, `não`, `nao`, `n`.

### 4.6 Datas e Timezone

- O timezone padrão do projeto é `America/Sao_Paulo`.
- Colunas temporais no PostgreSQL devem usar `TIMESTAMPTZ`.
- Colunas de data pura usam `DATE`; horas puras usam `TIME`.
- Nunca depender do timezone do sistema operacional.

---

## 5. Padrões de Código

### 5.1 Handler (Webhook)

```typescript
// handlers/webhook.handler.ts
import { Router, Request, Response } from 'express';
import { conversationService } from '../services/conversation.service';
import { authService } from '../services/auth.service';

const router = Router();

router.post('/webhook', async (req: Request, res: Response) => {
  // 1. Extrair mensagem do payload
  // 2. Verificar autorização do número
  // 3. Delegar para o service de conversa
  // 4. Responder 200 ao webhook
  res.sendStatus(200);
});

export default router;
```

**Regras:**
- Handlers apenas roteiam e delegam — sem lógica de negócio.
- Sempre responder `200` ao webhook da Meta rapidamente.
- Processar mensagens de forma assíncrona quando necessário.

### 5.2 Service

```typescript
// services/conversation.service.ts
import { prisma } from '../config/database';

export const conversationService = {
  async handleMessage(phoneNumber: string, message: string) {
    // 1. Buscar conversa ativa
    // 2. Determinar próximo passo
    // 3. Validar entrada
    // 4. Atualizar dados da conversa
    // 5. Enviar próxima pergunta
  },
};
```

**Regras:**
- Um service por domínio (`conversation`, `atestado`, `export`, `whatsapp`, `auth`).
- Services contêm toda lógica de negócio.
- Acesso a banco via Prisma Client.
- Validação de entrada via schemas Zod.

### 5.3 Validator (Zod)

```typescript
// validators/atestado.validator.ts
import { z } from 'zod';

export const emailSchema = z.string().email('E-mail inválido').transform(v => v.toLowerCase());

export const dateSchema = z.string().regex(
  /^\d{2}\/\d{2}\/\d{4}$/,
  'Formato esperado: DD/MM/AAAA'
);

export const timeSchema = z.string().regex(
  /^\d{2}:\d{2}$/,
  'Formato esperado: HH:MM'
);
```

**Regras:**
- Schemas Zod são puros — sem dependência de banco ou serviços.
- Um arquivo de validator por domínio ou grupo de validações.
- Mensagens de erro em português.

---

## 6. Git

| Branch | Uso |
| ------ | --- |
| `main` | Código estável |
| `develop` | Desenvolvimento ativo |

Fluxo padrão:

1. Desenvolver em `develop`.
2. Validar manualmente e por testes.
3. Fazer merge para `main` quando estiver estável.

---

## 7. Docker

### Regras para Dockerfile

- Usar multi-stage build.
- Imagem final mínima (Alpine).
- Não incluir devDependencies na imagem final.
- Não rodar container como root em produção.
- Health check obrigatório para a API.

### Variáveis de Ambiente

| Variável | Descrição | Exemplo |
| -------- | --------- | ------- |
| `DB_HOST` | Host do PostgreSQL | `postgres` |
| `DB_PORT` | Porta do PostgreSQL | `5432` |
| `DB_NAME` | Nome do banco | `chatbot` |
| `DB_USERNAME` | Usuário do banco | `chatbot` |
| `DB_PASSWORD` | Senha do banco | `secret` |
| `WHATSAPP_TOKEN` | Token da Meta API | — |
| `WHATSAPP_PHONE_NUMBER_ID` | ID do número WhatsApp | — |
| `WHATSAPP_VERIFY_TOKEN` | Token de verificação webhook | — |
| `WHATSAPP_APP_SECRET` | App Secret para assinatura | — |
| `APP_TIMEZONE` | Timezone (default: `America/Sao_Paulo`) | `America/Sao_Paulo` |
| `PORT` | Porta da API (default: `3000`) | `3000` |

---

## 8. Checklist para Novas Funcionalidades

- [ ] Arquivo no diretório correto (`handlers/`, `services/`, `validators/`, `types/`).
- [ ] Tipagem TypeScript completa (sem `any` desnecessário).
- [ ] Validação de entrada com Zod.
- [ ] E-mail normalizado para lowercase.
- [ ] Queries via Prisma (nunca concatenação SQL).
- [ ] Migration Prisma criada quando há mudança de schema.
- [ ] Colunas temporais usam `TIMESTAMPTZ`, `DATE` ou `TIME`.
- [ ] Verificação de autorização do número.
- [ ] Comandos admin protegidos por role.
- [ ] Testes unitários adicionados para lógica nova.
- [ ] Variáveis sensíveis via `.env`, nunca hardcoded.
- [ ] Plano de desenvolvimento atualizado com status da tarefa.

---

## 9. Regras de Teste

- Usar Vitest como framework de testes.
- Testar comportamento observável, não detalhes de implementação.
- Priorizar testes para: validadores, máquina de estados, geração de Excel.
- Todo bug corrigido deve ganhar teste de regressão.
- Dados de teste nunca devem depender do banco de produção.