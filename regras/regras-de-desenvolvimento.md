# Regras de Desenvolvimento — starter-kit

> **Projeto:** starter-kit
> **Data de criação:** 07/03/2026
> **Última atualização:** 08/03/2026

Documento complementar: [../documentos/decisoes-compartilhadas.md](../documentos/decisoes-compartilhadas.md)

---

## 1. Princípios Gerais

- **Simplicidade:** evitar over-engineering.
- **Consistência:** seguir os padrões definidos neste documento.
- **Segurança:** toda entrada do usuário deve ser validada.
- **Reprodutibilidade:** o projeto deve funcionar localmente de forma previsível.

---

## 2. Estrutura de Pastas — Obrigatória

### Backend (`/api`)

```text
src/main/java/com/starterkit/
├── config/
├── controller/
├── dto/
│   ├── request/
│   └── response/
├── entity/
├── enums/
├── exception/
├── filter/
├── repository/
├── service/
└── util/
```

**Regra:** cada classe deve ficar no pacote correto conforme sua responsabilidade. Controller não contém lógica de negócio. Service não acessa `HttpServletRequest`. Repository não contém lógica de negócio.

**Regra adicional:** entidades JPA não devem usar Lombok `@Data`. Preferir `@Getter`, `@Setter`, `@Builder`, `@NoArgsConstructor` e `@AllArgsConstructor`.

**Regra adicional:** entidades relevantes devem ter `createdAt`, `updatedAt` e `version`; usar `@Version` para concorrência otimista quando fizer sentido de negócio.

### Frontend (`/frontend`)

```text
src/
├── assets/
├── components/
│   ├── common/
│   └── layout/
├── layouts/
├── pages/
├── routes/
├── services/
├── stores/
├── types/
└── utils/
```

**Regra:** componentes de página ficam em `pages/`. Componentes reutilizáveis ficam em `components/`. Lógica de API fica em `services/`, nunca diretamente em componentes.

**Regra adicional:** formulários devem usar `react-hook-form` com `zod`.

---

## 3. Nomenclatura

### Backend (Java)

| Elemento | Convenção | Exemplo |
| -------- | --------- | ------- |
| Classe | PascalCase | `UserController` |
| Interface | PascalCase | `UserRepository` |
| Método | camelCase | `findByEmail()` |
| Variável | camelCase | `userName` |
| Constante | UPPER_SNAKE_CASE | `MAX_PAGE_SIZE` |
| Enum (valores) | UPPER_SNAKE_CASE | `EM_PREPARO`, `CANCELADO` |
| Pacote | lowercase | `com.starterkit.controller` |
| Tabela | snake_case, plural | `users`, `order_items` |
| Coluna | snake_case | `created_at`, `store_id` |
| DTO request | PascalCase + `Request` | `CreateUserRequest` |
| DTO response | PascalCase + `Response` | `UserResponse` |
| Endpoint REST | kebab-case, plural | `/api/order-items` |
| Migration | `V{n}__{descricao}.sql` | `V1__create_users_table.sql` |

### Frontend (TypeScript/React)

| Elemento | Convenção | Exemplo |
| -------- | --------- | ------- |
| Componente | PascalCase | `UserListPage.tsx` |
| Hook/Store | camelCase com prefixo `use` | `useAuthStore.ts` |
| Service | camelCase + `Service` | `authService.ts` |
| Interface | PascalCase | `UserResponse` |
| Variável/função | camelCase | `handleAcceptOrder` |
| Constante | UPPER_SNAKE_CASE | `API_BASE_URL` |

---

## 4. Segurança

### 4.1 SQL Injection

- Nunca concatenar strings para montar queries SQL.
- Sempre usar Spring Data query methods ou JPQL parametrizado.
- SQL puro apenas em migrations Flyway.

### 4.2 XSS

- Não usar `dangerouslySetInnerHTML` sem sanitização explícita.
- Não inserir dados do usuário diretamente no DOM.
- Não persistir refresh token em `localStorage` ou `sessionStorage`.

### 4.3 Senhas

- Armazenar sempre com BCrypt.
- Nunca logar senhas.
- Nunca retornar senha em DTO de resposta.
- Validação mínima: 8 caracteres.

### 4.4 JWT

- Secret via variável de ambiente.
- Access token com expiração curta.
- Refresh token com rotação a cada uso.
- Refresh token em cookie `HttpOnly`, `SameSite=Lax`.
- Tokens devem ser invalidados ao trocar senha ou fazer logout.

### 4.5 CORS

- Em desenvolvimento: permitir `http://localhost:5173`.
- Em produção: permitir apenas a origem servida pelo Nginx ou configurada por variável de ambiente.
- Quando houver refresh por cookie, `allowCredentials=true` é obrigatório.

### 4.6 Paginação e Ordenação

- `page` default = `0`.
- `size` default = `10`.
- `size` máximo = `50`.
- Campos de ordenação devem ser whitelistados no backend.
- Nunca confiar diretamente no campo de ordenação enviado pelo cliente.

### 4.7 Datas e Timezone

- O timezone padrão do projeto é `America/Sao_Paulo`.
- Colunas temporais no PostgreSQL devem usar `TIMESTAMPTZ`.
- No backend, preferir `Instant` para persistência e processamento interno.
- Conversões de fuso e formatação para tela ou relatório devem ocorrer na borda da aplicação.
- Evitar `ZoneId.systemDefault()` como fonte de verdade; usar configuração explícita da aplicação.

### 4.8 Estado Operacional Fora do Banco

- Sempre que houver arquivos operacionais, eles passam a fazer parte do perímetro de segurança e operação.
- Backup e restore devem considerar banco e arquivos operacionais como um conjunto coerente.
- Não assumir que `docker compose up` resolve toda a operação quando existirem diretórios externos ou volumes sensíveis.

---

## 5. Padrões de Código

### 5.1 Backend

#### Controller

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping
    public ResponseEntity<PageResponse<UserResponse>> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ResponseEntity.ok(userService.list(page, size));
    }
}
```

**Regras:**

- Usar `@RequiredArgsConstructor` para injeção via construtor.
- Usar `@Valid` em requests.
- Retornar `ResponseEntity` com status correto.
- Não conter lógica de negócio.
- Endpoints paginados devem impor limites de `size` e ordenação permitida.
- Autorização real deve estar no backend; ocultação no frontend nunca substitui regra de acesso.

#### Service

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public UserResponse create(CreateUserRequest request) {
        String normalizedEmail = request.getEmail().trim().toLowerCase();
        if (userRepository.existsByEmail(normalizedEmail)) {
            throw new BusinessException("E-mail já cadastrado");
        }
        return null;
    }
}
```

**Regras:**

- Usar `@Transactional` em operações de escrita.
- Lançar exceções de negócio específicas.
- Não capturar exceções genéricas sem necessidade.
- Converter Entity para DTO na service.
- Normalizar e-mail para lowercase antes de persistir ou consultar.
- Regras de período e corte temporal devem usar timezone configurado explicitamente.

#### Repository

```java
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

    boolean existsByEmail(String email);

    @Query("SELECT u FROM User u WHERE u.active = true")
    Page<User> findAllActive(Pageable pageable);
}
```

**Regras:**

- Repository deve ser interface que estende `JpaRepository`.
- Preferir query methods do Spring Data.
- Se usar `@Query`, parametrizar corretamente.
- Não usar concatenação em queries nativas.

### 5.2 Frontend

#### Service (API)

```tsx
import api from './api';

export const userService = {
  list: (page: number, size: number) =>
    api.get('/users', { params: { page, size } }),
};
```

**Regras:**

- Um service por domínio.
- Sempre tipar request e response.
- Usar a instância Axios configurada em `api.ts`.

#### Zustand Store

```tsx
interface AuthState {
  user: UserResponse | null;
  accessToken: string | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
}
```

**Regras:**

- Uma store por domínio.
- State e actions no mesmo `create`.
- Chamadas HTTP via services.
- Não persistir refresh token em storage do navegador.

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

### Regras para Dockerfiles

- Usar multi-stage build.
- Imagem final mínima.
- Não incluir ferramentas de build na imagem final.
- Não rodar container como root em produção.
- Health checks são obrigatórios para serviços principais.

### Variáveis de Ambiente

| Variável | Descrição | Exemplo |
| -------- | --------- | ------- |
| `DB_HOST` | Host do PostgreSQL | `postgres` |
| `DB_PORT` | Porta do PostgreSQL | `5432` |
| `DB_NAME` | Nome do banco | `starterkit` |
| `DB_USERNAME` | Usuário do banco | `starterkit` |
| `DB_PASSWORD` | Senha do banco | `secret` |
| `JWT_SECRET` | Chave secreta do JWT | valor forte |
| `JWT_EXPIRATION` | TTL do access token | `900000` |
| `JWT_REFRESH_EXPIRATION` | TTL do refresh token | `604800000` |
| `CORS_ORIGINS` | Origens permitidas | `http://localhost:80` |

---

## 8. Checklist para Novas Funcionalidades

- [ ] Entity criada com anotações JPA corretas.
- [ ] Migration Flyway criada para schema novo.
- [ ] DTOs de request e response separados.
- [ ] Repository com queries parametrizadas.
- [ ] Service com regras de negócio e validações.
- [ ] Controller com endpoints REST corretos e `@Valid`.
- [ ] Proteção por role aplicada quando necessário.
- [ ] Paginação e ordenação validadas no backend.
- [ ] Types TypeScript criados.
- [ ] Service frontend criado.
- [ ] Store criada quando fizer sentido.
- [ ] Formulários com `react-hook-form` + `zod` quando aplicável.
- [ ] Página com componentes MUI.
- [ ] Rota adicionada.
- [ ] Testes unitários adicionados.
- [ ] Testes de integração adicionados quando o fluxo for crítico.

---

## 9. Regras de Teste

- Backend: usar Testcontainers com PostgreSQL para testes de integração.
- Frontend: usar Vitest + Testing Library.
- Testar comportamento observável, não detalhes de implementação.
- Todo bug corrigido em auth, service ou segurança deve ganhar teste de regressão.
- Cobertura inicial deve priorizar autenticação, CRUD de usuários e tratamento global de erros.
- Fluxos com data, timezone e concorrência otimista devem ter testes dedicados quando forem relevantes.