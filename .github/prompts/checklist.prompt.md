---
description: "Checklist pós-tarefa para validar que o código segue as regras do projeto. Cobre arquitetura, segurança, nomenclatura, testes e documentação."
agent: "agent"
---
# Checklist Pós-Tarefa

Execute esta verificação após concluir uma tarefa de implementação.

## Backend

- [ ] Controller não contém lógica de negócio
- [ ] `@Valid` presente em todo `@RequestBody`
- [ ] `@Transactional` presente em operações de escrita no Service
- [ ] Entity não é exposta como response HTTP — DTO criado
- [ ] Entity não usa `@Data` — usa `@Getter`, `@Setter`, `@Builder`, etc.
- [ ] Entidades de domínio possuem `createdAt`, `updatedAt` e `version`
- [ ] Queries parametrizadas — sem concatenação SQL
- [ ] E-mail normalizado para lowercase antes de persistir/consultar
- [ ] Senha nunca retornada em DTO, nunca logada
- [ ] Endpoints protegidos com `@PreAuthorize` onde necessário
- [ ] Paginação com `size` máximo 50 e `sort` whitelistado
- [ ] Timestamps usam `Instant` no Java e `TIMESTAMPTZ` no banco
- [ ] Nenhuma dependência de `ZoneId.systemDefault()`

## Frontend

- [ ] Chamadas HTTP feitas via `services/`, não em componentes
- [ ] Tipos definidos para props, state e responses
- [ ] Formulários usam `react-hook-form` + `zod`
- [ ] Access token em memória — nada em localStorage/sessionStorage
- [ ] Componentes no diretório correto (`pages/`, `components/`, etc.)
- [ ] Sem `dangerouslySetInnerHTML` sem sanitização
- [ ] Sem `any` desnecessário

## Migrations

- [ ] Naming correto: `V{n}__{descricao}.sql`
- [ ] Colunas temporais usam `TIMESTAMPTZ`
- [ ] Tabela de domínio tem `id`, `created_at`, `updated_at`, `version`
- [ ] Sem concatenação SQL — valores literais ou parametrizados
- [ ] Índices criados para colunas filtradas/ordenadas

## Geral

- [ ] Testes unitários/integração cobrem a funcionalidade principal
- [ ] Nenhum `TODO` ou `FIXME` sem justificativa ficou no código
- [ ] Plano de desenvolvimento atualizado com status da tarefa
- [ ] Documentos do agente (`.github/instructions/`, `.github/prompts/`) refletem novas decisões, se houver
