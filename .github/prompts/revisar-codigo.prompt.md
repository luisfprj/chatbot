---
description: "Revisar código contra as regras de desenvolvimento do projeto. Verifica violações de arquitetura, segurança, nomenclatura e padrões obrigatórios."
agent: "agent"
---
# Revisão de Código

Revise o código atual (ou os arquivos alterados recentemente) contra as regras do projeto.

## O que verificar

### Arquitetura

1. Cada classe está no pacote/diretório correto conforme a estrutura definida?
2. Controllers delegam para Services sem lógica de negócio?
3. Services contêm `@Transactional` em escrita e convertem Entity→DTO?
4. Repositories usam query methods ou JPQL parametrizado?
5. Entidades JPA seguem as convenções (sem `@Data`, com `@Version`, com `createdAt`/`updatedAt`)?

### Segurança

6. Alguma query SQL usa concatenação de string em vez de parâmetros?
7. Senhas estão sendo logadas ou retornadas em algum DTO?
8. Endpoints sensíveis estão protegidos com `@PreAuthorize`?
9. `@Valid` está presente em todos os `@RequestBody`?
10. CORS está configurado com `allowCredentials=true`?
11. Refresh token está apenas em cookie HttpOnly (não em localStorage)?
12. Há uso de `dangerouslySetInnerHTML` sem sanitização?

### Nomenclatura e Convenções

13. Classes, métodos e variáveis seguem a convenção de naming documentada?
14. DTOs seguem o padrão `*Request` / `*Response`?
15. Migrations usam `V{n}__{descricao}.sql`?
16. Frontend: chamadas HTTP passam por `services/`, não por componentes?
17. Frontend: formulários usam `react-hook-form` + `zod`?

### Temporal

18. Colunas de data no banco usam `TIMESTAMPTZ`?
19. Backend usa `Instant` para campos temporais?
20. Alguma lógica depende de `ZoneId.systemDefault()`?

## Formato da Revisão

Para cada violação encontrada, reporte:

- **Arquivo e linha** onde a violação ocorre
- **Regra violada** (referência à regra específica)
- **Impacto** (segurança, consistência, manutenibilidade)
- **Correção sugerida** com exemplo de código quando aplicável

Se nenhuma violação for encontrada, confirme que o código está em conformidade.
