# Instruções Gerais do Projeto — starter-kit

## Contexto do Projeto

Este é um **starter-kit** monolítico modular para projetos locais. Roda com Docker, sem dependências externas em runtime. A stack é:

- **Backend:** Java 21 + Spring Boot 3.5.x + Gradle + Flyway + PostgreSQL
- **Frontend:** React 18 + Vite + TypeScript + Zustand + MUI + React Hook Form + Zod
- **Infra:** Docker Compose + Nginx (produção) + PostgreSQL 16

## Documentação Obrigatória

Antes de gerar código, consulte os documentos do projeto:

- [Arquitetura](../documentos/arquitetura.md) — modelo de dados, contratos REST, estrutura de pacotes
- [Regras de Desenvolvimento](../regras/regras-de-desenvolvimento.md) — padrões de código, segurança, nomenclatura
- [Decisões Compartilhadas](../documentos/decisoes-compartilhadas.md) — decisões transversais adotadas
- [Infraestrutura](../documentos/infraestrutura.md) — Docker, Nginx, profiles, portas
- [Plano de Desenvolvimento](../documentos/plano-de-desenvolvimento.md) — fases, tarefas e status

## Convenções Críticas

- **Timezone:** `America/Sao_Paulo` como fuso operacional. Persistir com `TIMESTAMPTZ`. Backend usa `Instant`. Nunca depender de `ZoneId.systemDefault()`.
- **Login por e-mail:** normalizar para lowercase antes de persistir ou consultar.
- **Sessão híbrida:** access token curto em memória + refresh token em cookie HttpOnly com rotação.
- **Backend é fonte de verdade de autorização.** Frontend apenas organiza navegação.
- **Schema controlado por Flyway.** Hibernate com `ddl-auto: validate`.
- **Entidades relevantes** devem ter `createdAt`, `updatedAt` e `version` (`@Version`).
- **Identificador padrão:** `BIGSERIAL`. UUID é decisão consciente por projeto.

## Arquitetura de Camadas (Backend)

```
Controller → Service → Repository → Entity
```

- Controller: HTTP e validação de entrada. Sem lógica de negócio.
- Service: regras de negócio, `@Transactional` em escrita, conversão Entity→DTO.
- Repository: interface JpaRepository, queries parametrizadas, sem concatenação SQL.
- Entity: sem Lombok `@Data`. Usar `@Getter`, `@Setter`, `@Builder`, `@NoArgsConstructor`, `@AllArgsConstructor`.

## Estrutura de Pastas

- Backend: `/api/src/main/java/com/starterkit/` — pacotes: config, controller, dto/request, dto/response, entity, enums, exception, filter, repository, service, util
- Frontend: `/frontend/src/` — pastas: assets, components/common, components/layout, layouts, pages, routes, services, stores, types, utils
- Documentos: `/documentos/`
- Regras: `/regras/`

## Segurança (Resumo)

- SQL: nunca concatenar strings. Usar query methods ou JPQL parametrizado.
- XSS: nunca usar `dangerouslySetInnerHTML` sem sanitização.
- Senhas: BCrypt, nunca logar, nunca retornar em DTO.
- JWT secret via variável de ambiente. Access token curto. Refresh com rotação.
- CORS explícito com `allowCredentials=true`.
- Paginação: `size` máximo 50, campos de ordenação whitelistados.

## Git

- Branches: `develop` (desenvolvimento ativo) e `main` (código estável).
- Desenvolver em `develop`, merge para `main` quando estável.

## Ao Planejar Próximas Etapas

Sempre considere: as instruções do agente (`.github/instructions/` e `.github/prompts/`) estão atualizadas com as decisões mais recentes? Se alguma regra, convenção ou padrão mudou, atualize os arquivos correspondentes para que o agente continue alinhado ao projeto.
