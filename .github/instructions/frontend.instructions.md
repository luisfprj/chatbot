---
applyTo: "frontend/src/**/*.{ts,tsx}"
description: "Use when editing React/TypeScript frontend code. Covers component structure, service layer, store patterns, form validation, and common violations like HTTP calls in components or missing types."
---
# Regras para Código Frontend — React + TypeScript

## Estrutura de Pastas

```
frontend/src/
├── assets/           → Imagens, fontes, ícones
├── components/
│   ├── common/       → Componentes reutilizáveis genéricos
│   └── layout/       → Header, Sidebar, Footer
├── layouts/          → Layout wrappers para rotas
├── pages/            → Componentes de página (uma por rota)
├── routes/           → Configuração de rotas
├── services/         → Chamadas HTTP (um service por domínio)
├── stores/           → Zustand stores (uma por domínio)
├── types/            → Interfaces e tipos compartilhados
└── utils/            → Funções utilitárias
```

## Regras Fundamentais

### Componentes

- Componentes de página ficam em `pages/`.
- Componentes reutilizáveis ficam em `components/`.
- Usar MUI (Material UI) como biblioteca de componentes.
- **Nunca** fazer chamadas HTTP diretamente em componentes — usar `services/`.

### Services (API)

- Um service por domínio (ex: `authService.ts`, `userService.ts`).
- Sempre tipar request e response com interfaces de `types/`.
- Usar a instância Axios configurada em `api.ts`.
- **Nunca** usar `fetch` diretamente ou criar instâncias Axios avulsas.

### Stores (Zustand)

- Uma store por domínio.
- State e actions no mesmo `create`.
- Chamadas HTTP via services, nunca direto na store.
- **Nunca** persistir refresh token em `localStorage` ou `sessionStorage`.
- Access token fica apenas em memória (variável da store).

### Formulários

- Usar `react-hook-form` com `zod` para validação.
- Schemas Zod em arquivo separado ou junto ao formulário.
- Mensagens de validação em português quando aplicável.

### Tipagem

- Sempre tipar props, state e retornos de funções.
- Evitar `any` — usar tipos específicos ou `unknown` quando necessário.
- Interfaces de API devem ficar em `types/`.

## Violações Comuns a Evitar

| Violação | Correção |
|----------|----------|
| HTTP direto em componente | Mover para `services/` |
| Falta tipagem | Tipar props, state e returns |
| Layout ad hoc sem componente | Usar componentes de `layouts/` |
| Token em localStorage | Manter access token em memória |
| `any` desnecessário | Definir tipo correto |
| Axios sem instância central | Usar `api.ts` configurado |
| Formulário sem validação | Usar react-hook-form + zod |

## Nomenclatura

| Elemento | Convenção | Exemplo |
|----------|-----------|---------|
| Componente (arquivo) | PascalCase | `UserListPage.tsx` |
| Hook/Store | camelCase + `use` | `useAuthStore.ts` |
| Service | camelCase + `Service` | `authService.ts` |
| Interface | PascalCase | `UserResponse` |
| Variável/função | camelCase | `handleSubmit` |
| Constante | UPPER_SNAKE_CASE | `API_BASE_URL` |

## Segurança no Frontend

- **Nunca** usar `dangerouslySetInnerHTML` sem sanitização explícita.
- Frontend **não define** autorização — apenas oculta elementos para UX. A permissão real vem do backend.
- Não expor secrets ou chaves em código frontend.
