---
description: "Template para iniciar uma nova fase do plano de desenvolvimento. Organiza contexto, tarefas e verificação de pré-requisitos antes de começar a implementação."
agent: "agent"
---
# Iniciar Nova Fase de Desenvolvimento

Antes de começar, execute os seguintes passos:

## 1. Contexto

- Leia o [plano de desenvolvimento](../documentos/plano-de-desenvolvimento.md) e identifique a próxima fase a iniciar.
- Leia a [arquitetura](../documentos/arquitetura.md) para entender o modelo de dados, estados da conversa e integração WhatsApp.
- Leia as [regras de desenvolvimento](../regras/regras-de-desenvolvimento.md) para seguir os padrões obrigatórios.
- Leia as [decisões compartilhadas](../documentos/decisoes-compartilhadas.md) para respeitar decisões já tomadas.

## 2. Pré-requisitos

- Verifique se as fases anteriores estão concluídas (status `✅`).
- Verifique se o banco está acessível e as migrations existentes estão aplicadas (`npx prisma migrate status`).
- Verifique se as variáveis de ambiente do WhatsApp estão configuradas (`.env`).
- Verifique se há tarefas da fase com dependências externas ou bloqueios.

## 3. Planejamento

- Liste todas as tarefas da fase identificada.
- Organize a implementação na ordem definida no plano.
- Para cada tarefa, identifique: arquivos a criar/modificar, alterações no schema Prisma, testes esperados.

## 4. Execução

- Implemente tarefa por tarefa, seguindo a ordem do plano.
- Após cada tarefa concluída, atualize o status no plano de desenvolvimento.
- Siga as regras de nomenclatura, segurança e arquitetura documentadas.

## 5. Validação Final

- Todos os testes passam (`npm test`).
- Webhook responde corretamente ao GET de verificação e POST de mensagens.
- Nenhum `TODO` ou `FIXME` ficou pendente.

## 6. Atualização dos Documentos do Agente

- As instruções em `.github/instructions/` e `.github/prompts/` estão atualizadas com eventuais novas decisões?
- Se alguma regra, convenção ou padrão mudou durante a fase, atualize os arquivos correspondentes.
