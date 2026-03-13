---
applyTo: "api/src/**/controller/**"
description: "Use when creating or editing REST controllers/endpoints. Covers HTTP verbs, validation, role protection, response patterns, pagination, and common violations like missing @Valid or wrong HTTP status."
---
# Regras para Contratos de API — Controllers

## Base URL

Todos os endpoints ficam sob `/api`. Exemplo: `/api/users`, `/api/auth/login`.

## Naming de Endpoints

- Usar **kebab-case** e **plural**: `/api/order-items`, `/api/users`.
- Nunca camelCase ou snake_case em URLs: ~~`/api/orderItems`~~, ~~`/api/order_items`~~.

## Verbos HTTP

| Operação | Verbo | Status sucesso |
|----------|-------|----------------|
| Listar (paginado) | `GET` | `200 OK` |
| Buscar por ID | `GET` | `200 OK` |
| Criar | `POST` | `201 Created` |
| Atualizar completo | `PUT` | `200 OK` |
| Excluir | `DELETE` | `204 No Content` |

**Nunca** usar `POST` para busca ou `GET` para criar/alterar.

## Validação de Entrada

- **Sempre** usar `@Valid` em `@RequestBody`.
- Validações via Bean Validation (`@NotBlank`, `@Email`, `@Size`, etc.) nos DTOs de request.
- E-mail deve ser normalizado para lowercase no service antes de persistir.

## Proteção por Role

- Endpoints de gestão (CRUD de usuários) exigem `@PreAuthorize("hasRole('ADMIN')")`.
- Endpoints de autenticação são públicos: `/api/auth/**`.
- Endpoints de health check são públicos: `/api/actuator/health`.
- Swagger é público em dev: `/api/swagger-ui/**`, `/api/v3/api-docs/**`.
- **Backend é a fonte de verdade de autorização.** Não depender de ocultação no frontend.

## Paginação

Endpoints de listagem devem seguir este padrão:

```java
@GetMapping
public ResponseEntity<PageResponse<UserResponse>> list(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {
    // size máximo 50, validar no service
}
```

- `page` default: `0`
- `size` default: `10`, máximo: `50`
- Campos de ordenação devem ser **whitelistados** no backend.
- Nunca confiar diretamente no campo `sort` enviado pelo cliente.

## Padrão de Response

### Sucesso paginado
```json
{
  "content": [...],
  "page": 0,
  "size": 10,
  "totalElements": 42,
  "totalPages": 5,
  "last": false
}
```

### Erro global
```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "Descrição do erro",
  "timestamp": "2026-03-07T12:00:00Z",
  "traceId": "4f0f8d7d4c3f4d3f",
  "fields": [
    { "field": "email", "message": "E-mail é obrigatório" }
  ]
}
```

O campo `fields` aparece apenas em erros de validação.

## Violações Comuns a Evitar

| Violação | Correção |
|----------|----------|
| Falta `@Valid` no request body | Adicionar `@Valid` em todo `@RequestBody` |
| Endpoint sem proteção de role | Adicionar `@PreAuthorize` onde necessário |
| Verb errado (POST para busca) | Usar `GET` para leitura, `POST` para criação |
| Status HTTP errado | `201` para criação, `204` para delete, `200` para leitura |
| Entity retornada direto | Converter para DTO response no service |
| Paginação sem limite | Impor `size` máximo 50 e whitelist de sort |
| Lógica de negócio no controller | Mover para service |

## Convenções

- Um controller por domínio: `AuthController`, `UserController`.
- Usar `@RequiredArgsConstructor` para injeção.
- Retornar `ResponseEntity<T>` sempre.
- Documentar com SpringDoc OpenAPI quando relevante.
