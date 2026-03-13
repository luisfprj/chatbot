# Infraestrutura — Chatbot WhatsApp Atestados

> **Projeto:** chatbot-atestado
> **Data de criação:** 13/03/2026
> **Última atualização:** 13/03/2026

Documento complementar: [decisoes-compartilhadas.md](decisoes-compartilhadas.md)

---

## 1. Visão Geral dos Containers

```text
┌────────────────────────────────────────────────────────┐
│                 Docker Compose Network                  │
│                 (chatbot-network)                       │
│                                                        │
│  ┌──────────────────┐     ┌──────────────────────┐     │
│  │      api         │     │      postgres         │     │
│  │    :3000         │────▶│       :5432           │     │
│  │                  │     │                       │     │
│  │  Node.js 20      │     │  PostgreSQL 16        │     │
│  │  TypeScript       │     │  Alpine              │     │
│  └────────┬─────────┘     └───────────────────────┘     │
│           │                                             │
└───────────┼─────────────────────────────────────────────┘
            │ webhooks (HTTPS via ngrok em dev)
            ▼
   ┌─────────────────────┐
   │  Meta Cloud API     │
   │  (WhatsApp Business)│
   └─────────────────────┘
```

**Portas expostas ao host:**

- `3000` para a API Node.js (webhook + health check).
- `5432` para PostgreSQL apenas em desenvolvimento local.

**Convenções operacionais:**

- Timezone padrão da aplicação: `America/Sao_Paulo`.
- Datas persistidas no banco com timezone (`TIMESTAMPTZ`).
- HTTPS obrigatório para receber webhooks da Meta — usar ngrok em desenvolvimento.

---

## 2. Docker Compose — Desenvolvimento

### `docker-compose.dev.yml`

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: chatbot-postgres
    environment:
      POSTGRES_DB: chatbot
      POSTGRES_USER: chatbot
      POSTGRES_PASSWORD: chatbot_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - chatbot-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U chatbot"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:

networks:
  chatbot-network:
    driver: bridge
```

**Em desenvolvimento:**

- A API Node.js roda localmente via `npm run dev`.
- Apenas o PostgreSQL roda no Docker.
- ngrok expõe a porta 3000 via HTTPS para os webhooks da Meta.
- A porta `5432` fica exposta para ferramentas como DBeaver.

---

## 3. Docker Compose — Produção

### `docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: chatbot-postgres
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - chatbot-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME}"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    container_name: chatbot-api
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: ${DB_NAME}
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      WHATSAPP_TOKEN: ${WHATSAPP_TOKEN}
      WHATSAPP_PHONE_NUMBER_ID: ${WHATSAPP_PHONE_NUMBER_ID}
      WHATSAPP_VERIFY_TOKEN: ${WHATSAPP_VERIFY_TOKEN}
      WHATSAPP_APP_SECRET: ${WHATSAPP_APP_SECRET}
      APP_TIMEZONE: ${APP_TIMEZONE:-America/Sao_Paulo}
      NODE_ENV: production
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - chatbot-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3000/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s

volumes:
  postgres_data:

networks:
  chatbot-network:
    driver: bridge
```

### `.env` (exemplo)

```env
# Banco de Dados
DB_NAME=chatbot
DB_USERNAME=chatbot
DB_PASSWORD=mudar_em_producao

# WhatsApp Business API (Meta Cloud API)
WHATSAPP_TOKEN=token-de-acesso-permanente-da-meta
WHATSAPP_PHONE_NUMBER_ID=id-do-numero-de-telefone
WHATSAPP_VERIFY_TOKEN=token-customizado-para-verificacao-webhook
WHATSAPP_APP_SECRET=app-secret-para-validacao-de-assinatura

# Aplicação
APP_TIMEZONE=America/Sao_Paulo
NODE_ENV=production
PORT=3000
```

> **Importante:** `.env` não deve ser versionado. Fornecer apenas `.env.example`.

---

## 4. Dockerfile

### Backend — `/api/Dockerfile`

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npx prisma generate
RUN npm run build

FROM node:20-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/package.json ./
COPY --from=build /app/prisma ./prisma
USER appuser
EXPOSE 3000
CMD ["node", "dist/app.js"]
```

---

## 5. Portas e Serviços

### Produção

| Serviço | Porta interna | Porta exposta | Acesso externo |
| ------- | ------------- | ------------- | -------------- |
| API Node.js | 3000 | 3000 | ✅ (recebe webhooks) |
| PostgreSQL | 5432 | — | ❌ |

### Desenvolvimento

| Serviço | Porta | Acesso |
| ------- | ----- | ------ |
| API (dev) | 3000 | `http://localhost:3000` |
| PostgreSQL | 5432 | `localhost:5432` |
| ngrok | 443 | HTTPS → localhost:3000 (para webhooks da Meta) |

---

## 6. Configuração do WhatsApp (pré-requisito externo)

### Setup inicial obrigatório

1. Criar conta no [Meta for Developers](https://developers.facebook.com/).
2. Criar um aplicativo do tipo Business.
3. Adicionar o produto WhatsApp ao app.
4. Configurar um número de telefone de teste.
5. Gerar um token de acesso permanente (System User Token).
6. Configurar a URL do webhook:
   - **Desenvolvimento:** URL do ngrok (ex: `https://abc123.ngrok.io/webhook`)
   - **Produção:** URL pública do servidor (ex: `https://meuservidor.com/webhook`)
7. Definir os campos de assinatura do webhook: `messages`.

### ngrok em desenvolvimento

```bash
# Instalar ngrok
npm install -g ngrok

# Expor porta 3000
ngrok http 3000
```

Copiar a URL HTTPS gerada e registrar no painel da Meta como URL do webhook.

---

## 7. Volumes e Persistência

| Volume | Container | Path | Descrição |
| ------ | --------- | ---- | --------- |
| `postgres_data` | postgres | `/var/lib/postgresql/data` | Dados do banco |

---

## 8. Comandos Úteis

### Desenvolvimento

```bash
# Subir banco
docker compose -f docker-compose.dev.yml up -d

# Instalar dependências
cd api && npm install

# Rodar migrations
npx prisma migrate dev

# Seed de dados iniciais
npx prisma db seed

# Iniciar em modo dev
npm run dev

# Expor para WhatsApp (em outro terminal)
ngrok http 3000

# Parar banco
docker compose -f docker-compose.dev.yml down
```

### Produção

```bash
docker compose up -d --build
docker compose logs -f
docker compose logs -f api
docker compose down
docker compose down -v
```

### Prisma

```bash
# Criar migration
npx prisma migrate dev --name descricao_da_migration

# Aplicar migrations em produção
npx prisma migrate deploy

# Abrir Prisma Studio (visualizar dados)
npx prisma studio

# Gerar client
npx prisma generate
```

### Troubleshooting

Em incidentes, validar nesta ordem:

1. Health check do PostgreSQL (`pg_isready`).
2. Health check da API (`GET /health`).
3. Variáveis de ambiente obrigatórias (tokens WhatsApp, DB).
4. ngrok ativo e URL registrada na Meta (em dev).
5. Assinatura do webhook válida (verificar `WHATSAPP_APP_SECRET`).

---

## 9. Git e GitHub

### Repositório

| Dado | Valor |
| ---- | ----- |
| Plataforma | GitHub |
| Repositório | `git@github.com:luisfprj/chatbot.git` |
| URL | https://github.com/luisfprj/chatbot |
| Tipo | Mono-repo |

### Branches

| Branch | Uso | Protegida |
| ------ | --- | --------- |
| `main` | Código estável | Sim |
| `develop` | Desenvolvimento ativo | Não |

---

## 10. Variáveis de Ambiente — Referência Completa

| Variável | Obrigatória | Descrição | Exemplo |
| -------- | ----------- | --------- | ------- |
| `DB_HOST` | Sim (prod) | Host do PostgreSQL | `postgres` |
| `DB_PORT` | Sim (prod) | Porta do PostgreSQL | `5432` |
| `DB_NAME` | Sim | Nome do banco | `chatbot` |
| `DB_USERNAME` | Sim | Usuário do banco | `chatbot` |
| `DB_PASSWORD` | Sim | Senha do banco | `secret` |
| `WHATSAPP_TOKEN` | Sim | Token de acesso da Meta API | — |
| `WHATSAPP_PHONE_NUMBER_ID` | Sim | ID do número WhatsApp Business | — |
| `WHATSAPP_VERIFY_TOKEN` | Sim | Token para verificação do webhook | — |
| `WHATSAPP_APP_SECRET` | Sim | App Secret para validar assinatura | — |
| `APP_TIMEZONE` | Não | Timezone (default: `America/Sao_Paulo`) | `America/Sao_Paulo` |
| `PORT` | Não | Porta da API (default: `3000`) | `3000` |
| `NODE_ENV` | Não | Ambiente (default: `development`) | `production` |