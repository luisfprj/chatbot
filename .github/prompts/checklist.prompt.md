---
description: "Checklist pós-tarefa para validar que o código segue as regras do projeto. Cobre arquitetura, segurança, nomenclatura, testes e documentação."
agent: "agent"
---
# Checklist Pós-Tarefa

Execute esta verificação após concluir uma tarefa de implementação.

## Handlers (Webhook)

- [ ] Handler não contém lógica de negócio — apenas recebe e delega
- [ ] Assinatura `X-Hub-Signature-256` é verificada em todo POST do webhook
- [ ] Dados de entrada validados com Zod antes de processar
- [ ] Resposta 200 retornada imediatamente ao webhook (processamento assíncrono se necessário)

## Services

- [ ] Toda lógica de negócio está no service, não no handler
- [ ] Acesso a dados feito exclusivamente via Prisma Client
- [ ] Máquina de estados da conversa segue os estados documentados em `arquitetura.md`
- [ ] Transições de estado tratam entradas inválidas com mensagem amigável

## Segurança

- [ ] Queries parametrizadas via Prisma — sem concatenação SQL
- [ ] Tokens e secrets em variáveis de ambiente — nunca no código ou logs
- [ ] Apenas números cadastrados em `authorized_numbers` com `active = true` interagem
- [ ] Comando `/exportar` restrito a role `ADMIN`
- [ ] E-mail do colaborador normalizado para lowercase antes de persistir

## Prisma / Banco

- [ ] Modelos possuem `createdAt`, `updatedAt` com `TIMESTAMPTZ`
- [ ] Identificadores usam `BigInt` (`BIGSERIAL`)
- [ ] Schema atualizado via `prisma migrate dev` — nunca alteração manual
- [ ] Índices criados para campos usados em buscas frequentes

## Tipos e Validação

- [ ] Tipos TypeScript definidos para todas as estruturas de dados
- [ ] Sem `any` desnecessário
- [ ] Schemas Zod cobrem todas as entradas do usuário

## Geral

- [ ] Testes unitários cobrem a funcionalidade principal (Vitest)
- [ ] Nenhum `TODO` ou `FIXME` sem justificativa ficou no código
- [ ] Plano de desenvolvimento atualizado com status da tarefa
- [ ] Documentos do agente (`.github/instructions/`, `.github/prompts/`) refletem novas decisões, se houver
