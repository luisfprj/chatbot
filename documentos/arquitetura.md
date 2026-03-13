# Documento de Arquitetura — starter-kit

> **Projeto:** starter-kit
> **Data de criação:** 07/03/2026
> **Última atualização:** 08/03/2026

Documento complementar: [decisoes-compartilhadas.md](decisoes-compartilhadas.md)

---

## 1. Visão Geral da Arquitetura

O starter-kit segue uma arquitetura **monolítica modular** distribuída em containers Docker. Todos os serviços rodam na mesma rede local, sem dependências externas em runtime.

### Definições operacionais

- **Local-first:** o sistema não depende da internet para funcionar, mas continua dependendo do backend e do banco locais.
- **Ponto único de entrada:** Nginx é o único serviço exposto externamente em produção.
- **Identificador de login:** o e-mail do usuário é o login oficial do sistema.
- **Sessão híbrida:** access token curto em memória + refresh token persistido em cookie HttpOnly com rotação.
- **Timezone padrão do projeto:** `America/Sao_Paulo` como fuso operacional padrão.
- **Estratégia temporal:** persistir timestamps com timezone no banco e trabalhar com `Instant` no backend sempre que possível.

```text
┌─────────────────────────────────────────────────────────────┐
│                    Rede Local (Docker)                      │
│                                                             │
│  ┌──────────┐     ┌──────────────┐     ┌────────────────┐   │
│  │ Browser  │────▶│    Nginx     │────▶│   PostgreSQL   │   │
│  │          │     │     :80      │     │     :5432      │   │
│  └──────────┘     └──────┬───────┘     └───────▲────────┘   │
│                          │                     │             │
│                    ┌─────▼─────┐               │             │
│                    │ Backend   │───────────────┘             │
│                    │   :8080   │                             │
│                    │ Spring    │                             │
│                    └───────────┘                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Fluxo de Requisições

1. Browser acessa `http://localhost`.
2. Nginx recebe todas as requisições.
3. Rotas `/api/*` são encaminhadas ao backend.
4. Rotas `/*` servem os arquivos estáticos do frontend.
5. Backend se comunica com PostgreSQL na rede interna.
6. Em desenvolvimento, apenas o PostgreSQL roda via Docker; frontend e backend rodam localmente.

---

## 2. Arquitetura do Backend

### Padrão: camadas

```text
┌─────────────────────────────────────────┐
│ Controller                             │  ← HTTP, validação de entrada
├─────────────────────────────────────────┤
│ Service                                │  ← regras de negócio
├─────────────────────────────────────────┤
│ Repository                             │  ← acesso a dados
├─────────────────────────────────────────┤
│ Entity / Enum                          │  ← modelo persistido
└─────────────────────────────────────────┘
```

### Regras de dependência

- Controller depende de Service e DTO.
- Service depende de Repository e Entity.
- Repository depende apenas de Entity.
- Entity não depende de camadas superiores.

### Convenções transversais adotadas

#### Data e hora

- O banco deve usar `TIMESTAMPTZ` para colunas de data e hora.
- O backend deve preferir `Instant` para persistência, comparação e eventos de sistema.
- Conversões para apresentação e regras de calendário devem ocorrer na borda da aplicação usando o timezone configurado.
- O timezone padrão do starter é `America/Sao_Paulo` e não deve depender de `ZoneId.systemDefault()`.

#### Persistência estrutural

- O schema é controlado exclusivamente por migrations Flyway.
- O Hibernate opera com `ddl-auto: validate`.
- Entidades de domínio devem padronizar `createdAt`, `updatedAt` e `version`.
- Concorrência otimista via `@Version` é a abordagem padrão para entidades relevantes.
- O starter mantém `BIGSERIAL` como identidade padrão por simplicidade local. UUID fica como decisão opcional por projeto, não como padrão da base.

#### Operação e segurança

- Backend é a fonte de verdade de autorização; frontend só organiza navegação e restrição visual.
- Health checks e observabilidade fazem parte da arquitetura base.
- Backup e restore são preocupações arquiteturais do sistema quando houver estado fora do banco.
- Arquivos operacionais, quando existirem, entram no perímetro de segurança, backup e restore.

### Pacotes Java

```text
com.starterkit
├── config/
│   ├── SecurityConfig
│   ├── SwaggerConfig
│   └── WebConfig
├── controller/
│   ├── AuthController
│   └── UserController
├── dto/
│   ├── request/
│   └── response/
├── entity/
│   ├── User
│   └── RefreshToken
├── enums/
│   └── Role
├── exception/
│   ├── GlobalExceptionHandler
│   ├── BusinessException
│   ├── ResourceNotFoundException
│   └── UnauthorizedException
├── filter/
│   └── JwtAuthenticationFilter
├── repository/
│   ├── UserRepository
│   └── RefreshTokenRepository
├── service/
│   ├── AuthService
│   ├── UserService
│   └── JwtService
└── util/
    └── PasswordUtil
```

### Modelo de Dados

#### Tabela: `users`

| Coluna | Tipo | Restrições | Descrição |
| ------ | ---- | ---------- | --------- |
| id | BIGSERIAL | PK, auto-increment | Identificador único |
| name | VARCHAR(255) | NOT NULL | Nome completo |
| email | VARCHAR(255) | NOT NULL | E-mail de login |
| password | VARCHAR(255) | NOT NULL | Hash BCrypt |
| role | VARCHAR(20) | NOT NULL, DEFAULT 'USER' | ADMIN ou USER |
| active | BOOLEAN | NOT NULL, DEFAULT true | Usuário ativo/inativo |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Data de criação |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Data de atualização |
| version | BIGINT | NOT NULL, DEFAULT 0 | Controle de concorrência otimista |

**Regra adicional:** aplicar unicidade por e-mail normalizado em lowercase.

#### Tabela: `refresh_tokens`

| Coluna | Tipo | Restrições | Descrição |
| ------ | ---- | ---------- | --------- |
| id | BIGSERIAL | PK, auto-increment | Identificador único |
| token | VARCHAR(512) | NOT NULL, UNIQUE | Token opaco |
| user_id | BIGINT | FK → users.id, NOT NULL | Usuário dono do token |
| expires_at | TIMESTAMPTZ | NOT NULL | Expiração |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Data de criação |
| revoked_at | TIMESTAMPTZ | NULL | Momento da revogação |

#### Índices recomendados

- `ux_users_email_lower` em `lower(email)`.
- `idx_refresh_tokens_user_id` em `user_id`.
- `idx_refresh_tokens_expires_at` em `expires_at`.

### Flyway Migrations

| Arquivo | Descrição |
| ------- | --------- |
| `V1__create_users_table.sql` | Cria `users` com índice único em e-mail normalizado, timestamps com timezone e coluna `version` |
| `V2__create_refresh_tokens.sql` | Cria `refresh_tokens` com `revoked_at` e índices |
| `V3__insert_admin_user.sql` | Insere admin padrão |

### Decisões explicitamente não adotadas por padrão

- PWA não faz parte do starter base neste momento.
- API administrativa de backup e restore não é requisito inicial da base.
- Contexto organizacional como `storeId` não é obrigatório no starter; pode ser introduzido por projeto quando houver multiunidade.

---

## 3. Contratos da API REST

### Base URL: `/api`

### Autenticação

#### `POST /api/auth/login`

**Request Body:**

```json
{
  "email": "admin@starterkit.com",
  "password": "senha123"
}
```

**Response 200:**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

**Cabeçalho adicional:** `Set-Cookie: refreshToken=<opaque>; HttpOnly; SameSite=Lax; Path=/api/auth`

#### `POST /api/auth/refresh`

**Request:** sem body; usa o cookie `refreshToken`

**Response 200:** mesmo formato do login, com rotação do cookie de refresh

#### `POST /api/auth/logout`

**Request:** sem body; usa o cookie `refreshToken`

**Response 204:** token revogado e cookie expirado

#### `GET /api/auth/me`

**Headers:** `Authorization: Bearer <accessToken>`

**Response 200:**

```json
{
  "id": 1,
  "name": "Administrador",
  "email": "admin@starterkit.com",
  "role": "ADMIN",
  "active": true
}
```

### Usuários (requer ADMIN)

#### `GET /api/users?page=0&size=10&sort=name,asc`

**Headers:** `Authorization: Bearer <accessToken>`

**Response 200:**

```json
{
  "content": [
    {
      "id": 1,
      "name": "Administrador",
      "email": "admin@starterkit.com",
      "role": "ADMIN",
      "active": true,
      "createdAt": "2026-03-07T12:00:00Z"
    }
  ],
  "page": 0,
  "size": 10,
  "totalElements": 1,
  "totalPages": 1,
  "last": true
}
```

#### `GET /api/users/{id}`

**Response 200:**

```json
{
  "id": 1,
  "name": "Administrador",
  "email": "admin@starterkit.com",
  "role": "ADMIN",
  "active": true,
  "createdAt": "2026-03-07T12:00:00Z",
  "updatedAt": "2026-03-07T12:00:00Z"
}
```

#### `POST /api/users`

**Request Body:**

```json
{
  "name": "Novo Usuário",
  "email": "novo@starterkit.com",
  "password": "minhasenha123",
  "role": "USER"
}
```

#### `PUT /api/users/{id}`

**Request Body:**

```json
{
  "name": "Nome Atualizado",
  "email": "atualizado@starterkit.com",
  "role": "ADMIN",
  "active": true
}
```

#### `DELETE /api/users/{id}`

**Response 204:** sem corpo

### Padrão de Erro Global

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "Descrição do erro",
  "timestamp": "2026-03-07T12:00:00Z",
  "traceId": "4f0f8d7d4c3f4d3f",
  "fields": [
    {
      "field": "email",
      "message": "E-mail é obrigatório"
    }
  ]
}
```

O campo `fields` aparece apenas em erros de validação.

---

## 4. Arquitetura do Frontend

### Estrutura de Pastas

```text
frontend/src/
├── assets/
├── components/
│   ├── common/
│   └── layout/
├── layouts/
├── pages/
│   ├── auth/
│   ├── dashboard/
│   └── users/
├── routes/
├── services/
│   ├── api.ts
│   ├── authService.ts
│   └── userService.ts
├── stores/
│   ├── useAuthStore.ts
│   └── useUserStore.ts
├── types/
├── utils/
├── App.tsx
├── main.tsx
└── theme.ts
```

### Fluxo de Autenticação (Frontend)

```text
LoginPage
  └── authService.login()
        ├── recebe access token
        ├── navegador recebe cookie HttpOnly de refresh
        └── store mantém access token em memória

Requests autenticadas
  └── Axios interceptor adiciona Authorization: Bearer <token>

Ao receber 401
  └── authService.refresh()
        ├── usa cookie refreshToken
        ├── recebe novo access token
        └── reexecuta request original
```

### Armazenamento de Tokens

- **Access Token:** mantido em memória na store de autenticação.
- **Refresh Token:** mantido em cookie `HttpOnly`, `SameSite=Lax`, inacessível ao JavaScript.
- Na inicialização da app, o frontend tenta renovar o access token automaticamente usando o cookie de refresh.

### Rotas

| Rota | Página | Acesso | Layout |
| ---- | ------ | ------ | ------ |
| `/login` | LoginPage | Público | AuthLayout |
| `/` | DashboardPage | Autenticado | MainLayout |
| `/users` | UserListPage | ADMIN | MainLayout |
| `/users/new` | UserFormPage | ADMIN | MainLayout |
| `/users/:id` | UserFormPage | ADMIN | MainLayout |

---

## 5. Fluxo JWT — Detalhamento Técnico

### Configuração dos Tokens

| Parâmetro | Valor |
| --------- | ----- |
| Algoritmo | HS256 |
| Access Token TTL | 15 minutos |
| Refresh Token TTL | 7 dias |
| JWT Secret | Variável de ambiente (`JWT_SECRET`) |
| Refresh Token formato | UUID v4 opaco |
| Transporte do refresh | Cookie HttpOnly |

### Fluxo Completo

```text
1. Login
   Cliente → POST /api/auth/login {email, password}
   Servidor → valida credenciais
            → gera access token
            → gera refresh token
            → salva refresh token no banco
            → retorna access token
            → envia cookie HttpOnly

2. Requisição autenticada
   Cliente → GET /api/users com Authorization: Bearer <accessToken>
   Servidor → valida JWT
            → carrega usuário
            → autoriza acesso

3. Access token expirado
   Cliente → recebe 401
   Cliente → POST /api/auth/refresh com cookie refreshToken
   Servidor → valida refresh token
            → verifica expiração e revogação
            → gera novo access token
            → gera novo refresh token
            → revoga token anterior
            → retorna novo access token
            → regrava cookie

4. Logout
   Cliente → POST /api/auth/logout
   Servidor → revoga refresh token atual
            → expira cookie
   Cliente → limpa access token da memória
```

### Proteção de Endpoints por Role

| Endpoint | ADMIN | USER | Público |
| -------- | ----- | ---- | ------- |
| `POST /api/auth/login` | - | - | ✅ |
| `POST /api/auth/refresh` | - | - | ✅ |
| `POST /api/auth/logout` | ✅ | ✅ | ❌ |
| `GET /api/auth/me` | ✅ | ✅ | ❌ |
| `GET /api/users` | ✅ | ❌ | ❌ |
| `GET /api/users/{id}` | ✅ | ❌ | ❌ |
| `POST /api/users` | ✅ | ❌ | ❌ |
| `PUT /api/users/{id}` | ✅ | ❌ | ❌ |
| `DELETE /api/users/{id}` | ✅ | ❌ | ❌ |
| `GET /api/swagger-ui/**` | - | - | ✅ em dev |

---

## 6. Telas — Especificação

### Tela de Login

- Card centralizado.
- Campos: e-mail e senha.
- Botão `Entrar`.
- Erro inline para credenciais inválidas.
- Redireciona para `/` após login.

### Layout Principal

- Sidebar com links de navegação.
- TopBar com nome do usuário e ação de logout.
- Conteúdo central renderizado pela rota ativa.
- Item `Usuários` visível apenas para ADMIN.

### Dashboard

- Título `Dashboard`.
- Texto `Bem-vindo ao starter-kit`.
- Espaço reservado para widgets futuros.

### Listagem de Usuários

- Tabela server-side com paginação.
- Ações por linha: editar e excluir.
- Botão `+ Novo`.
- Estados explícitos de loading, vazio e erro.

### Formulário de Usuário

- Campos: nome, e-mail, senha na criação, perfil, ativo.
- Validação client-side com Zod.
- Erros do backend exibidos inline quando possível.

---

## 7. Decisões Técnicas

| Decisão | Justificativa |
| ------- | ------------- |
| Access token em memória | Evita persistência desnecessária no navegador |
| Refresh token em cookie HttpOnly | Reduz risco de XSS com baixa complexidade operacional |
| Refresh token rotation | Cada uso gera novo token e invalida o anterior |
| BCrypt para senhas | Padrão de mercado |
| Flyway para migrations | Versionamento e reprodutibilidade do schema |
| Nginx como entry point único | Simplifica proxy, CORS e futura camada SSL |
| MUI | Biblioteca madura e amplamente adotada |
| Zustand | Simples e suficiente para escopo médio |
| Axios | Interceptors e tratamento de erro mais ergonômicos |
| React Hook Form + Zod | Validação forte sem excesso de boilerplate |
| Multi-stage Docker build | Imagens menores em produção |
| Testcontainers em vez de H2 | Evita divergência entre teste e produção |