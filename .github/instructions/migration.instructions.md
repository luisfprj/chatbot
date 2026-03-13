---
applyTo: "api/src/**/db/migration/**"
description: "Use when creating or editing Flyway SQL migrations. Covers naming conventions, required columns, temporal types, index patterns, and safety rules against SQL concatenation."
---
# Regras para Migrations Flyway

## Naming Convention

Padrão obrigatório: `V{n}__{descricao}.sql`

- `V` maiúsculo seguido de número sequencial.
- Dois underscores `__` separando versão e descrição.
- Descrição em `snake_case`, descritiva da operação.

Exemplos corretos:
```
V1__create_users_table.sql
V2__create_refresh_tokens.sql
V3__insert_admin_user.sql
V4__add_phone_to_users.sql
```

Exemplos **incorretos**:
```
v1_create_users.sql        → v minúsculo, um underscore
V1-create-users.sql        → hífen em vez de underscore
create_users.sql           → sem prefixo de versão
```

## Colunas Obrigatórias

Toda tabela de domínio deve ter:

| Coluna | Tipo | Restrição |
|--------|------|-----------|
| `id` | `BIGSERIAL` | `PRIMARY KEY` |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT CURRENT_TIMESTAMP` |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT CURRENT_TIMESTAMP` |
| `version` | `BIGINT` | `NOT NULL DEFAULT 0` |

## Convenções Temporais

- **Sempre** usar `TIMESTAMPTZ` para colunas de data e hora. Nunca `TIMESTAMP` sem timezone.
- O timezone operacional do projeto é `America/Sao_Paulo`.

## Contexto Organizacional (quando aplicável)

Se o projeto derivado usar multi-tenant ou contexto organizacional:
- Avaliar se a tabela precisa de `store_id` ou equivalente.
- No starter-kit base, `store_id` **não** é obrigatório por padrão.

## Segurança SQL

- **Nunca** concatenar strings em SQL. Usar parâmetros (`$1`, `$2` ou nomes).
- Valores dinâmicos em `INSERT` de seed devem ser constantes literais.
- Senhas em seeds devem ser hash BCrypt pré-computados.

## Índices

- Criar índices para colunas usadas em `WHERE`, `JOIN` e foreign keys.
- Naming de índice: `idx_{tabela}_{coluna}` ou `ux_{tabela}_{coluna}` para unique.
- Exemplos:
  ```sql
  CREATE UNIQUE INDEX ux_users_email_lower ON users (lower(email));
  CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens (user_id);
  ```

## Foreign Keys

- Nomear explicitamente: `fk_{tabela_origem}_{tabela_destino}`.
- Exemplo:
  ```sql
  CONSTRAINT fk_refresh_tokens_users FOREIGN KEY (user_id) REFERENCES users(id)
  ```

## Regras Gerais

- Cada migration deve ser atômica — uma responsabilidade por arquivo.
- Nunca alterar uma migration já executada. Criar uma nova.
- O Hibernate está configurado com `ddl-auto: validate` — toda mudança de schema passa por migration.
