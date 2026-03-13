# Decisões Compartilhadas — Chatbot WhatsApp Atestados

> **Projeto:** chatbot-atestado
> **Origem de referência:** starter-kit genérico
> **Data:** 13/03/2026
>
> Este documento registra decisões transversais do projeto. Serve como referência para manter consistência ao longo do desenvolvimento.

---

## 1. Decisões Adotadas

| Decisão | Aplicação no projeto |
| ------- | -------------------- |
| Timezone padrão `America/Sao_Paulo` | Convenção operacional — datas formatadas neste fuso |
| PostgreSQL com `TIMESTAMPTZ` | Colunas temporais persistidas com timezone |
| Prisma como ORM e ferramenta de migrations | Schema controlado por `prisma migrate` |
| Node.js + TypeScript como backend | Runtime do chatbot e servidor de webhooks |
| Zod para validação server-side | Validação de e-mail, datas, horas e entradas do usuário |
| WhatsApp Business API (Meta Cloud API) | Canal único de interação com o usuário |
| Autorização por número de telefone | Tabela `authorized_numbers` com role `USER` / `ADMIN` |
| ExcelJS para geração de planilhas | Exportação sob demanda por administradores |
| Docker + Docker Compose | Containerização e orquestração local |
| `BIGSERIAL` como identificador padrão | Menor complexidade; UUID não é necessário neste projeto |
| Vitest como framework de testes | Testes unitários e de integração |
| Queries sempre parametrizadas via Prisma | Prevenção de SQL injection |
| Variáveis sensíveis via `.env` | Tokens, secrets e credenciais nunca no código |
| Verificação de assinatura de webhook | Segurança na recepção de mensagens da Meta API |

---

## 2. Decisões do Starter-Kit Não Adotadas

| Decisão do starter-kit | Motivo da remoção |
| ---------------------- | ----------------- |
| Frontend React + Vite + MUI | Sem interface web — toda interação é via WhatsApp |
| Nginx como proxy reverso | Sem frontend web para servir |
| JWT + Refresh Token (autenticação web) | Autenticação é pelo número de telefone do WhatsApp |
| CRUD de usuários com UI | Substituído por tabela `authorized_numbers` com seed |
| Spring Boot + Java + Gradle | Substituído por Node.js + TypeScript — melhor ecossistema para bots |
| Flyway + Hibernate | Substituído por Prisma (migrations + ORM) |
| Swagger / OpenAPI | Sem API REST pública — o backend é um webhook receiver |
| Concorrência otimista (`@Version`) | Desnecessária — operações são lineares por conversa |
| Lombok | Tecnologia Java, não aplicável |
| Testcontainers | Substituído por testes Node.js com banco real local |
| Actuator / health checks Spring | Substituído por endpoint `/health` simples em Express |
| CORS configurado | Sem frontend web; webhook não depende de CORS |

---

## 3. Decisões de Domínio do Chatbot

| Decisão | Detalhe |
| ------- | ------- |
| Fluxo de conversa como máquina de estados | Cada conversa tem `current_step` e acumula dados em JSONB |
| Timeout de 30 minutos | Conversas inativas são expiradas automaticamente |
| Confirmação antes de salvar | Usuário vê resumo e confirma antes do registro ser persistido |
| Campos condicionais (horas vs. dias) | Fluxo bifurca conforme resposta de afastamento |
| Exportação exclusiva para admin | Comando `/exportar` só funciona para números com role `ADMIN` |
| Planilha com uma linha por atendimento | Formato compatível com a planilha original do Google Forms |
| Nome em caixa alta | O chatbot orienta o usuário a escrever em caixa alta |
| E-mail normalizado para lowercase | Antes de validar e persistir |

---

## 4. Diretriz de Uso

Ao evoluir este projeto:

1. Manter as decisões adotadas como padrão, salvo motivo técnico forte para mudança.
2. Registrar qualquer mudança em segurança, persistência ou integração como decisão explícita neste documento.
3. Sempre verificar se as instruções do agente (`.github/instructions/`) refletem as decisões mais recentes.
