---
applyTo: "api/prisma/**"
description: "Use when creating or editing Prisma schema and migrations. Covers model conventions, required fields, temporal types, index patterns, and safety rules."
---
# Regras para Prisma Schema e Migrations

## Schema (`schema.prisma`)

### Convenções de Modelo

Cada tabela de domínio deve ter:

| Campo | Tipo Prisma | Tipo DB | Regra |
|-------|-------------|---------|-------|
| `id` | `BigInt @id @default(autoincrement())` | `BIGSERIAL` | Obrigatório |
| `created_at` | `DateTime @default(now())` | `TIMESTAMPTZ` | Obrigatório |
| `updated_at` | `DateTime @updatedAt` | `TIMESTAMPTZ` | Obrigatório |

### Convenções Temporais

- **Sempre** usar `DateTime` para timestamps — mapea para `TIMESTAMPTZ` no PostgreSQL.
- Colunas de data pura: usar tipo nativo `@db.Date`.
- Colunas de hora pura: usar tipo nativo `@db.Time()`.
- O timezone operacional do projeto é `America/Sao_Paulo`.

### Naming no Schema

- Nome do modelo: `PascalCase` (ex: `AuthorizedNumber`, `Atestado`, `Conversation`)
- Nome do campo: `camelCase` (ex: `phoneNumber`, `createdAt`)
- Usar `@@map("nome_tabela")` para mapear para snake_case no banco.
- Usar `@map("nome_coluna")` para campos individuais.

Exemplo:
```prisma
model AuthorizedNumber {
  id          BigInt   @id @default(autoincrement())
  phoneNumber String   @unique @map("phone_number") @db.VarChar(20)
  name        String   @db.VarChar(255)
  role        String   @default("USER") @db.VarChar(10)
  active      Boolean  @default(true)
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  @@map("authorized_numbers")
}
```

## Migrations

### Criação

```bash
npx prisma migrate dev --name descricao_em_snake_case
```

- Descrição em `snake_case`, descritiva da operação.
- Exemplos: `create_authorized_numbers`, `create_conversations`, `create_atestados`.

### Regras Gerais

- Cada migration deve ter uma responsabilidade clara.
- **Nunca** alterar uma migration já executada. Criar uma nova.
- O Prisma controla o schema — nunca alterar o banco manualmente.

## Segurança SQL

- **Nunca** concatenar strings em SQL.
- Acessar dados sempre via Prisma Client (queries parametrizadas automaticamente).
- SQL puro somente em migrations `prisma/migrations/`, com valores literais.
- Seeds devem usar Prisma Client, não SQL direto.

## Índices

- Definir índices no `schema.prisma` usando `@@index` ou `@@unique`.
- Naming: Prisma gera nomes automaticamente, mas pode-se customizar.
- Criar índices para campos usados em buscas frequentes.
- Exemplo:
  ```prisma
  @@index([phoneNumber], map: "idx_atestados_phone")
  @@index([timestamp], map: "idx_atestados_timestamp")
  ```

## Seed (`prisma/seed.ts`)

- Seed script em TypeScript.
- Configurar no `package.json`:
  ```json
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
  ```
- Usar `upsert` para idempotência.
- Inserir dados iniciais: números admin e usuários de teste.
