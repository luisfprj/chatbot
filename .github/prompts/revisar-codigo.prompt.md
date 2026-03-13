---
description: "Revisar código contra as regras de desenvolvimento do projeto. Verifica violações de arquitetura, segurança, nomenclatura e padrões obrigatórios."
agent: "agent"
---
# Revisão de Código

Revise o código atual (ou os arquivos alterados recentemente) contra as regras do projeto.

## O que verificar

### Arquitetura

1. Cada arquivo está no diretório correto conforme a estrutura definida (`handlers/`, `services/`, `validators/`, `types/`, `utils/`)?
2. Handlers delegam para services sem lógica de negócio?
3. Services contêm a lógica de negócio e acessam dados via Prisma Client?
4. Validators são schemas Zod puros, sem dependência de banco?
5. Tipos estão definidos em `types/` e são importados onde necessário?

### Segurança

6. Alguma operação de banco usa concatenação de string em vez de Prisma Client?
7. Tokens/secrets estão hardcoded ou sendo logados?
8. Webhook verifica `X-Hub-Signature-256` em todo POST recebido?
9. Apenas números com `active = true` em `authorized_numbers` podem interagir?
10. Comando `/exportar` é restrito à role `ADMIN`?
11. Dados de entrada são validados com Zod antes de processar?

### Nomenclatura e Convenções

12. Funções e variáveis usam `camelCase`?
13. Tipos e interfaces usam `PascalCase`?
14. Arquivos usam `kebab-case.ts`?
15. Modelos Prisma usam `PascalCase` com `@@map` para snake_case?
16. E-mail normalizado para lowercase antes de persistir?

### Temporal

17. Colunas de data no Prisma usam `DateTime` (mapeia para `TIMESTAMPTZ`)?
18. Timezone operacional `America/Sao_Paulo` respeitado?
19. Nenhuma dependência do timezone do sistema operacional?

### Máquina de Estados

20. Transições de estado seguem o fluxo documentado em `arquitetura.md`?
21. Entradas inválidas resultam em mensagem amigável, não em erro silencioso?
22. Estado da conversa é persistido corretamente em `conversations`?

## Formato da Revisão

Para cada violação encontrada, reporte:

- **Arquivo e linha** onde a violação ocorre
- **Regra violada** (referência à regra específica)
- **Impacto** (segurança, consistência, manutenibilidade)
- **Correção sugerida** com exemplo de código quando aplicável

Se nenhuma violação for encontrada, confirme que o código está em conformidade.
