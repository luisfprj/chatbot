---
applyTo: "api/src/**/*.java"
description: "Use when editing Java backend code. Covers layered architecture rules, entity conventions, service patterns, security, and common violations like logic in controllers or exposed entities."
---
# Regras para Código Java — Backend

## Arquitetura de Camadas

Respeitar rigorosamente: **Controller → Service → Repository → Entity**.

### Controller

- Usar `@RequiredArgsConstructor` para injeção via construtor.
- Usar `@Valid` em todo request body.
- Retornar `ResponseEntity` com status HTTP correto.
- **Nunca** conter lógica de negócio — apenas receber, validar e delegar.
- Endpoints paginados devem impor limites de `size` (máximo 50) e ordenação whitelistada.
- Autorização real via `@PreAuthorize` — frontend não define permissão.

### Service

- Usar `@Transactional` em operações de escrita.
- Lançar exceções de negócio específicas (`BusinessException`, `ResourceNotFoundException`).
- Converter Entity → DTO aqui, nunca expor Entity na resposta HTTP.
- Normalizar e-mail para lowercase antes de persistir ou consultar.
- Regras temporais devem usar timezone configurado (`America/Sao_Paulo`), nunca `ZoneId.systemDefault()`.

### Repository

- Interface que estende `JpaRepository`.
- Preferir query methods do Spring Data.
- Se usar `@Query`, parametrizar corretamente (`:param` ou `?1`).
- **Nunca** concatenar strings em queries nativas.

### Entity

- **Nunca** usar Lombok `@Data` em entidades JPA.
- Usar `@Getter`, `@Setter`, `@Builder`, `@NoArgsConstructor`, `@AllArgsConstructor`.
- Entidades relevantes devem ter `createdAt`, `updatedAt` e `version`.
- Usar `@Version` para concorrência otimista.
- Preferir `Instant` para campos temporais.
- Identificador padrão: `BIGSERIAL` (Long no Java).

## Violações Comuns a Evitar

| Violação | Correção |
|----------|----------|
| Lógica de negócio no Controller | Mover para Service |
| Entity exposta como response | Criar DTO response |
| Falta `@Transactional` em escrita | Adicionar no Service |
| Falta `@Valid` no request body | Adicionar no Controller |
| Concatenação SQL | Usar parâmetros |
| `@Data` em Entity | Substituir por `@Getter`/`@Setter` |
| Senha retornada em DTO | Excluir do response |

## Pacotes

Cada classe deve ficar no pacote correto conforme sua responsabilidade:

```
com.starterkit.config/
com.starterkit.controller/
com.starterkit.dto.request/
com.starterkit.dto.response/
com.starterkit.entity/
com.starterkit.enums/
com.starterkit.exception/
com.starterkit.filter/
com.starterkit.repository/
com.starterkit.service/
com.starterkit.util/
```

## Nomenclatura

- Classe: `PascalCase` (ex: `UserController`)
- Método: `camelCase` (ex: `findByStoreIdAndStatus()`)
- Constante: `UPPER_SNAKE_CASE` (ex: `MAX_PAGE_SIZE`)
- Enum (valores): `UPPER_SNAKE_CASE` (ex: `EM_PREPARO`, `CANCELADO`)
- DTO request: `PascalCase` + `Request` (ex: `CreateUserRequest`)
- DTO response: `PascalCase` + `Response` (ex: `UserResponse`)
- Endpoint REST: kebab-case, plural (ex: `/api/order-items`)
