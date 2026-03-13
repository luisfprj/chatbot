# Plano de Desenvolvimento — starter-kit

> **Projeto:** starter-kit
> **Data de criação:** 07/03/2026
> **Última atualização:** 08/03/2026
> **Status geral:** Fase 0 — Planejamento revisado

---

## Documentos Relacionados

| Documento | Descrição |
| --------- | --------- |
| [arquitetura.md](arquitetura.md) | Arquitetura do sistema, modelo de dados, contratos da API, especificação de telas |
| [infraestrutura.md](infraestrutura.md) | Docker, Nginx, profiles Spring, portas, comandos úteis |
| [decisoes-compartilhadas.md](decisoes-compartilhadas.md) | Decisões transversais reaproveitadas do projeto de referência |
| [regras-de-desenvolvimento.md](../regras/regras-de-desenvolvimento.md) | Padrões de código, segurança, nomenclatura, checklist |
| [projeto.txt](../projeto.txt) | Decisões e definições originais do projeto |
| [.github/](../.github/) | Instruções e prompts para o agente Copilot (instructions, prompts) |

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Stack Tecnológica](#2-stack-tecnológica)
3. [Estrutura do Repositório](#3-estrutura-do-repositório)
4. [Fases de Desenvolvimento](#4-fases-de-desenvolvimento)
5. [Evolução e Histórico](#5-evolução-e-histórico)

---

## 1. Visão Geral

O **starter-kit** é um projeto base completo, pronto para ser replicado como ponto de partida para novos projetos. Roda 100% local, sem dependências externas em runtime. Todas as dependências são resolvidas no build.

### Princípios

- **Local-first:** frontend, backend e banco operam na rede local, sem depender da internet em runtime.
- **Zero internet em runtime:** nenhum CDN, fonte externa ou API de terceiros.
- **Replicável:** clonar, renomear e começar um novo projeto.
- **Simples e direto:** evitar over-engineering e manter padrão profissional de mercado.

### Ajustes estratégicos adotados nesta revisão

- **"Offline first" foi reinterpretado como local-first:** a aplicação não precisa funcionar sem backend; ela precisa funcionar sem internet externa.
- **Login canônico por e-mail:** o identificador oficial de acesso passa a ser o e-mail. Usuário inicial: `admin@starterkit.com` / `senha123`.
- **Refresh token em cookie HttpOnly:** mantém a simplicidade do frontend e melhora a segurança em relação a `localStorage`.
- **Desenvolvimento e produção separados com clareza:** em dev sobe apenas o PostgreSQL via Docker; Nginx entra no fluxo de produção e homologação.
- **Banco real nos testes de integração:** backend usa Testcontainers com PostgreSQL, evitando divergências entre H2 e produção.
- **Observabilidade mínima obrigatória:** health check via Actuator, logs estruturados em produção e `traceId` por requisição.
- **Tempo padronizado:** timezone padrão `America/Sao_Paulo`, timestamps persistidos com timezone e backend preferindo `Instant`.
- **Concorrência otimista como padrão:** entidades relevantes devem ter `version` para prevenir sobrescritas silenciosas.
- **Backend como fonte real de permissão:** frontend não define autorização, apenas experiência e navegação.
- **Backup e restore como decisão arquitetural:** quando existir estado fora do banco, ele entra no perímetro operacional do sistema.

---

## 2. Stack Tecnológica

### Frontend

| Tecnologia | Versão/Detalhe |
| ---------- | -------------- |
| React | 18+ |
| Vite | 5+ |
| TypeScript | 5+ |
| Zustand | Gerenciamento de estado de sessão e UI |
| React Router | Roteamento SPA |
| MUI (Material UI) | Biblioteca de componentes |
| Axios | Cliente HTTP |
| React Hook Form + Zod | Formulários e validação client-side |
| Vitest | Testes unitários |
| Testing Library | Testes de componentes |

### Backend

| Tecnologia | Versão/Detalhe |
| ---------- | -------------- |
| Java | 21 |
| Spring Boot | 3.5.x |
| Gradle | Build tool |
| SpringDoc OpenAPI | Swagger / documentação da API |
| Spring Boot Actuator | Health checks e observabilidade |
| Lombok | Redução de boilerplate |
| Flyway | Migrations de banco |
| SLF4J + Logback | Logging |
| JJWT | Geração e validação de JWT |
| JUnit 5 + Mockito | Testes unitários |
| Testcontainers PostgreSQL | Testes de integração |

### Infraestrutura

| Tecnologia | Detalhe |
| ---------- | ------- |
| PostgreSQL | Banco de dados |
| Nginx | Proxy reverso + serve frontend |
| Docker | Containerização |
| Docker Compose | Orquestração local |
| Timezone operacional | `America/Sao_Paulo` |

### Autenticação

| Detalhe | Valor |
| ------- | ----- |
| Método | JWT + Refresh Token |
| Perfis | ADMIN, USER |
| Admin padrão | login: `admin@starterkit.com` / senha: `senha123` (via Flyway) |
| Access Token | Em memória no frontend |
| Refresh Token | Cookie HttpOnly com rotação |

---

## 3. Estrutura do Repositório

```text
starter-kit/
├── api/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/starterkit/
│   │   │   └── resources/
│   │   └── test/
│   ├── build.gradle
│   ├── Dockerfile
│   └── settings.gradle
├── frontend/
│   ├── src/
│   ├── public/
│   ├── index.html
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   └── Dockerfile
├── documentos/
├── regras/
├── nginx/
├── docker-compose.dev.yml
├── docker-compose.yml
├── projeto.txt
├── .gitignore
└── README.md
```

---

## 4. Fases de Desenvolvimento

### Fase 1 — Infraestrutura e Ambiente de Desenvolvimento

**Objetivo:** preparar o mono-repo, variáveis de ambiente e banco local para desenvolvimento.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 1.1 | Criar estrutura de diretórios | `/api`, `/frontend`, `/documentos`, `/regras`, `/nginx` | ⬜ |
| 1.2 | Criar `docker-compose.dev.yml` | PostgreSQL 16 Alpine, volume `postgres_data`, porta 5432 exposta, healthcheck. Em dev não sobe Nginx | ⬜ |
| 1.3 | Criar `.editorconfig` | Padronizar indentação, charset e final de linha | ⬜ |
| 1.4 | Criar `.gitignore` | Ignorar `.env`, `node_modules/`, `build/`, `.gradle/`, `dist/`, `postgres_data/`, IDEs | ✅ |
| 1.5 | Criar `.env.example` | Variáveis: `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD`, `JWT_SECRET`, `JWT_EXPIRATION`, `JWT_REFRESH_EXPIRATION`, `CORS_ORIGINS` | ⬜ |
| 1.6 | Definir convenção temporal | Registrar timezone padrão `America/Sao_Paulo`, uso de `TIMESTAMPTZ` e `Instant` na base | ⬜ |
| 1.7 | Criar `README.md` | Pré-requisitos, setup dev e prod, links para documentos | ⬜ |
| 1.8 | Inicializar Git | `main` + `develop` | ✅ |

**Critério de conclusão:** `docker compose -f docker-compose.dev.yml up` sobe o PostgreSQL na porta 5432 e responde a conexões.

### Fase 2 — Backend Base

**Objetivo:** projeto Spring Boot funcional com Flyway, Swagger, Actuator, tratamento de erros e logging.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 2.1 | Inicializar Spring Boot | Dependências: web, data-jpa, validation, security, actuator, postgresql, flyway-core, lombok, springdoc, jjwt | ⬜ |
| 2.2 | Configurar profiles | `application.yml`, `application-dev.yml`, `application-prod.yml` | ⬜ |
| 2.3 | Configurar SpringDoc OpenAPI | Swagger UI em `/api/swagger-ui/index.html` | ⬜ |
| 2.4 | Migration V1: tabela `users` | Usar `TIMESTAMPTZ`, unicidade por e-mail normalizado e coluna `version` para concorrência otimista | ⬜ |
| 2.5 | Migration V2: tabela `refresh_tokens` | Incluir `revoked_at` e índices em `user_id` e `expires_at` | ⬜ |
| 2.6 | Migration V3: inserir admin | `admin@starterkit.com` com senha BCrypt de `senha123` | ⬜ |
| 2.7 | Criar entidades JPA | `User`, `RefreshToken`, `Role`, sem Lombok `@Data` em entidade; `User` com `@Version` | ⬜ |
| 2.8 | Tratamento global de erros | `GlobalExceptionHandler` com formato padronizado e `traceId` | ⬜ |
| 2.9 | Logging estruturado | Dev legível, prod em JSON com `traceId` | ⬜ |
| 2.10 | DTOs de paginação | `PageResponse<T>` genérico | ⬜ |
| 2.11 | Expor health check | `/api/actuator/health` para Docker | ⬜ |
| 2.12 | Configurar convenção de tempo | Configurar timezone padrão da aplicação e evitar dependência de `systemDefault` | ⬜ |
| 2.13 | Dockerfile backend | Multi-stage com runtime não-root | ⬜ |

**Critério de conclusão:** backend sobe com `./gradlew bootRun`, Swagger acessível em `http://localhost:8080/api/swagger-ui/index.html`, health check responde em `http://localhost:8080/api/actuator/health`, banco com tabelas criadas e logs corretos.

### Fase 3 — Autenticação e Autorização

**Objetivo:** login funcional com JWT, refresh token seguro e proteção de rotas por perfil.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 3.1 | Spring Security config | Permitir `/api/auth/**`, `/api/swagger-ui/**`, `/api/actuator/health` | ⬜ |
| 3.2 | `JwtService` | Gerar e validar access token com TTL de 15 min | ⬜ |
| 3.3 | `JwtAuthenticationFilter` | Extrair Bearer token, validar e popular `SecurityContext` | ⬜ |
| 3.4 | `AuthController` | `login`, `refresh`, `logout`, `me` | ⬜ |
| 3.5 | `AuthService` | Login por e-mail, refresh por cookie, logout com revogação | ⬜ |
| 3.6 | Refresh token rotation | Novo token a cada uso, invalidação do anterior | ⬜ |
| 3.7 | Proteção por role | `@PreAuthorize("hasRole('ADMIN')")` no CRUD de usuários; backend como fonte de verdade de permissão | ⬜ |
| 3.8 | CORS | Dev: `http://localhost:5173`; produção: origem servida pelo Nginx; `allowCredentials=true` | ⬜ |

**Critério de conclusão:** login cria sessão com access token + cookie de refresh, `me` funciona com bearer token, `refresh` renova sessão, `logout` revoga sessão, requests sem token retornam 401.

### Fase 4 — Frontend Base

**Objetivo:** app React funcional com login, dashboard vazio, navegação e integração com API.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 4.1 | Inicializar Vite + React + TS | Projeto na pasta `/frontend` | ⬜ |
| 4.2 | Instalar dependências | MUI, Zustand, React Router, Axios, React Hook Form, Zod, `@fontsource/roboto` | ⬜ |
| 4.3 | Tema MUI | Tema local, sem CDN | ⬜ |
| 4.4 | Axios instance | `baseURL=/api`, `withCredentials=true`, interceptors de auth | ⬜ |
| 4.5 | `useAuthStore` | `user`, `accessToken`, `login`, `logout`, `refreshSession`, `fetchMe` | ⬜ |
| 4.6 | `AuthLayout` | Card centralizado | ⬜ |
| 4.7 | `MainLayout` | Sidebar + TopBar + conteúdo | ⬜ |
| 4.8 | Rotas protegidas | Redirecionamento por autenticação e role | ⬜ |
| 4.9 | Página Login | Formulário com React Hook Form + Zod | ⬜ |
| 4.10 | Página Dashboard | Placeholder para widgets futuros | ⬜ |
| 4.11 | Configurar rotas | `/login`, `/`, `/users`, `/users/new`, `/users/:id` | ⬜ |
| 4.12 | `vite.config.ts` | Proxy `/api` → `http://localhost:8080` | ⬜ |
| 4.13 | Dockerfile frontend | Multi-stage para build estático | ⬜ |

**Critério de conclusão:** `npm run dev` exibe login, autentica com `admin@starterkit.com` / `senha123`, mantém sessão via cookie HttpOnly e permite logout correto.

### Fase 5 — CRUD de Usuário

**Objetivo:** CRUD completo servindo como modelo replicável.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 5.1 | `UserRepository` | `findByEmail()`, `existsByEmail()`, `findAllActive(Pageable)` com e-mail normalizado | ⬜ |
| 5.2 | DTOs | Requests com Bean Validation; response com `createdAt` e `updatedAt` | ⬜ |
| 5.3 | `UserService` | Listar, buscar, criar, atualizar, deletar, impedir auto-exclusão | ⬜ |
| 5.4 | `UserController` | Endpoints REST paginados e protegidos | ⬜ |
| 5.5 | Types TypeScript | `UserResponse`, `CreateUserRequest`, `UpdateUserRequest` | ⬜ |
| 5.6 | `userService.ts` | `list`, `getById`, `create`, `update`, `remove` | ⬜ |
| 5.7 | `useUserStore` | Lista, loading, paginação e ações | ⬜ |
| 5.8 | Página listagem | Tabela server-side com loading, empty state e error state | ⬜ |
| 5.9 | Formulário criar/editar | MUI + React Hook Form + Zod | ⬜ |
| 5.10 | Confirmação de exclusão | Dialog MUI | ⬜ |
| 5.11 | Sidebar — item Usuários | Visível apenas para ADMIN | ⬜ |

**Critério de conclusão:** admin gerencia usuários com paginação, validações claras e fluxo completo de criação, edição e exclusão.

### Fase 6 — Qualidade e Testes

**Objetivo:** testes automatizados cobrindo cenários principais e servindo de referência.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 6.1 | Configurar testes backend | `spring-boot-starter-test`, `spring-security-test`, `testcontainers-postgresql`; sem H2 | ⬜ |
| 6.2 | Testes unitários — `UserService` | `create`, `update`, `delete`, `list` | ⬜ |
| 6.3 | Testes unitários — `JwtService` | gerar, validar, expirar, extrair claims | ⬜ |
| 6.4 | Teste integração — `AuthController` | login, refresh por cookie, logout, `me` | ⬜ |
| 6.5 | Teste integração — `UserController` | CRUD, autorização por role, e-mail duplicado | ⬜ |
| 6.6 | Configurar Vitest + Testing Library | `jsdom`, setup, aliases | ⬜ |
| 6.7 | Teste — `useAuthStore` | login, logout, bootstrap de sessão, `fetchMe` | ⬜ |
| 6.8 | Teste componente — LoginPage | renderização, validação, erro de credenciais | ⬜ |
| 6.9 | Testar concorrência e tempo | Cobrir serialização temporal, `@Version` e regras de período dependentes do timezone configurado | ⬜ |
| 6.10 | Documentar padrão de testes | Convenções, cobertura esperada e critérios mínimos | ⬜ |

**Critério de conclusão:** `./gradlew test` e `npm run test` passam com cobertura útil dos fluxos críticos.

### Fase 7 — Runtime Web, Docker Produção e Finalização

**Objetivo:** aplicação completa rodando em Docker com Nginx como entrada única, pronta para replicação.

| # | Tarefa | Detalhes | Status |
|---|--------|----------|--------|
| 7.1 | `docker-compose.yml` | Serviços: postgres, api, nginx; só porta 80 exposta | ⬜ |
| 7.2 | Variáveis via `.env` | `.env.example` documentado e `.env` ignorado | ⬜ |
| 7.3 | Health checks | Postgres com `pg_isready`; backend com `/api/actuator/health` | ⬜ |
| 7.4 | Nginx produção | Gzip, cache de assets, headers básicos de segurança e bloqueio de arquivos ocultos | ⬜ |
| 7.5 | Estratégia de backup e restore | Documentar o que deve entrar em backup quando houver arquivos operacionais fora do banco | ⬜ |
| 7.6 | `README.md` final | Sobre, setup, deploy, estrutura e variáveis | ⬜ |
| 7.7 | Atualizar documentos | Revisar plano, arquitetura, regras, infraestrutura e decisões compartilhadas | ⬜ |
| 7.8 | Teste final de replicação | `docker compose up --build` sobe o sistema completo e permite login + CRUD | ⬜ |

**Critério de conclusão:** qualquer pessoa com Docker instalado consegue subir a aplicação, acessar `http://localhost`, autenticar e gerenciar usuários.

---

## 5. Evolução e Histórico

| Data | Fase | Descrição |
| ---- | ---- | --------- |
| 07/03/2026 | Fase 0 | Planejamento inicial concluído |
| 08/03/2026 | Fase 0 | Planejamento revisado para alinhar segurança, autenticação, observabilidade e fluxo dev/prod |

---

> **Próximo passo:** iniciar **Fase 1 — Infraestrutura e Ambiente de Desenvolvimento**.