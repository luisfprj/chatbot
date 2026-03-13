# Documento de Arquitetura — Chatbot WhatsApp Atestados

> **Projeto:** chatbot-atestado
> **Data de criação:** 13/03/2026
> **Última atualização:** 13/03/2026

Documento complementar: [decisoes-compartilhadas.md](decisoes-compartilhadas.md)

---

## 1. Visão Geral da Arquitetura

O sistema é um **chatbot WhatsApp** para coleta de atestados médicos. Não possui interface web — toda a interação acontece pelo WhatsApp via Meta Cloud API (WhatsApp Business). O backend recebe webhooks, conduz a conversa, persiste os dados em PostgreSQL e gera planilhas Excel sob demanda.

### Definições operacionais

- **Sem frontend web:** toda a interface é o WhatsApp.
- **Webhook-driven:** o backend reage a mensagens recebidas via webhook da Meta Cloud API.
- **Autorização por número:** somente números cadastrados na tabela `authorized_numbers` podem usar o bot.
- **Papel admin:** números com role `ADMIN` podem executar comandos administrativos como `/exportar`.
- **Timezone padrão do projeto:** `America/Sao_Paulo` como fuso operacional padrão.
- **Estratégia temporal:** persistir timestamps com timezone no banco (`TIMESTAMPTZ`) e trabalhar com UTC/ISO no backend.

```text
┌─────────────────────────────────────────────────────────────┐
│                    Docker Compose Network                    │
│                                                             │
│  ┌──────────────────┐          ┌────────────────────┐       │
│  │   Node.js App    │          │   PostgreSQL 16     │       │
│  │   (TypeScript)   │─────────▶│      :5432          │       │
│  │    :3000         │          │                     │       │
│  └────────┬─────────┘          └─────────────────────┘       │
│           │                                                  │
└───────────┼──────────────────────────────────────────────────┘
            │ webhooks (HTTPS)
            ▼
   ┌─────────────────────┐
   │  Meta Cloud API     │
   │  (WhatsApp Business)│
   └─────────────────────┘
```

### Fluxo Principal

1. Usuário envia mensagem no WhatsApp para o número do bot.
2. Meta Cloud API envia webhook `POST` para o backend.
3. Backend verifica assinatura do webhook (segurança).
4. Backend identifica o número e verifica se está autorizado.
5. Serviço de conversa conduz o fluxo (máquina de estados).
6. Dados validados são persistidos no PostgreSQL.
7. Admin envia `/exportar` → backend gera Excel → envia via WhatsApp como documento.

---

## 2. Arquitetura do Backend

### Padrão: camadas

```text
┌─────────────────────────────────────────┐
│ Webhook Handler (routes)               │  ← recebe e roteia webhooks
├─────────────────────────────────────────┤
│ Services                               │  ← lógica de negócio e orquestração
├─────────────────────────────────────────┤
│ Prisma (Repository)                    │  ← acesso a dados
├─────────────────────────────────────────┤
│ Validators (Zod)                       │  ← validação de entrada
└─────────────────────────────────────────┘
```

### Regras de dependência

- Webhook handlers dependem de Services.
- Services dependem de Prisma Client e Validators.
- Validators são puros (sem dependência de banco ou serviços).

### Convenções transversais adotadas

#### Data e hora

- O banco deve usar `TIMESTAMPTZ` para colunas de timestamp.
- Colunas de data pura (sem hora) usam `DATE`; colunas de hora pura usam `TIME`.
- O backend trabalha com `Date` (ISO 8601) e converte para exibição usando o timezone `America/Sao_Paulo`.
- Nunca depender do timezone do sistema operacional.

#### Persistência estrutural

- O schema é controlado exclusivamente por Prisma Migrations.
- Tabelas de domínio devem padronizar `created_at` e `updated_at`.
- O identificador padrão é `BIGSERIAL` (mapeado como `BigInt` ou `Int` no Prisma).

#### Segurança

- Verificação de assinatura do webhook (X-Hub-Signature-256) é obrigatória.
- Apenas números cadastrados no banco podem interagir com o bot.
- Queries sempre parametrizadas via Prisma — nunca concatenar SQL.
- Variáveis sensíveis (tokens, secrets) via variáveis de ambiente.

### Estrutura de Pastas

```text
api/
├── src/
│   ├── config/              → variáveis de ambiente, constantes
│   ├── handlers/            → rotas e handlers de webhook
│   ├── services/
│   │   ├── whatsapp.service.ts      → enviar mensagens e documentos via Meta API
│   │   ├── conversation.service.ts  → máquina de estados da conversa
│   │   ├── atestado.service.ts      → persistência de atestados
│   │   ├── export.service.ts        → geração de Excel
│   │   └── auth.service.ts          → verificação de números autorizados
│   ├── validators/          → schemas Zod (email, datas, horas, etc.)
│   ├── types/               → interfaces e tipos TypeScript
│   ├── utils/               → utilitários (formatação, etc.)
│   └── app.ts               → servidor Express + rotas de webhook
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── package.json
├── tsconfig.json
├── Dockerfile
└── .env.example
```

---

## 3. Modelo de Dados

### Tabela: `authorized_numbers`

| Coluna | Tipo | Restrições | Descrição |
| ------ | ---- | ---------- | --------- |
| id | BIGSERIAL | PK | Identificador único |
| phone_number | VARCHAR(20) | NOT NULL, UNIQUE | Número WhatsApp (formato internacional, ex: `5521999999999`) |
| name | VARCHAR(255) | NOT NULL | Nome do responsável |
| role | VARCHAR(10) | NOT NULL, DEFAULT 'USER' | `USER` ou `ADMIN` |
| active | BOOLEAN | NOT NULL, DEFAULT true | Número ativo/inativo |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Data de criação |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Data de atualização |

**Índices:**
- `ux_authorized_numbers_phone` em `phone_number` (UNIQUE).

### Tabela: `conversations`

| Coluna | Tipo | Restrições | Descrição |
| ------ | ---- | ---------- | --------- |
| id | BIGSERIAL | PK | Identificador único |
| phone_number | VARCHAR(20) | NOT NULL | Número do usuário |
| current_step | VARCHAR(50) | NOT NULL, DEFAULT 'INICIO' | Etapa atual na máquina de estados |
| data | JSONB | NOT NULL, DEFAULT '{}' | Dados parciais coletados ao longo da conversa |
| started_at | TIMESTAMPTZ | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Início da conversa |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Última interação |
| completed_at | TIMESTAMPTZ | NULL | Preenchido quando a conversa é finalizada |

**Índices:**
- `idx_conversations_phone_active` em `phone_number` WHERE `completed_at IS NULL` (conversas ativas).

### Tabela: `atestados`

| Coluna | Tipo | Restrições | Descrição |
| ------ | ---- | ---------- | --------- |
| id | BIGSERIAL | PK | Identificador único |
| phone_number | VARCHAR(20) | NOT NULL | Número de quem registrou |
| timestamp | TIMESTAMPTZ | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Data/hora do registro |
| unidade_trabalho | VARCHAR(255) | NOT NULL | Unidade de trabalho |
| matricula | VARCHAR(50) | NOT NULL | Matrícula |
| nome_completo | VARCHAR(255) | NOT NULL | Nome completo (caixa alta) |
| email | VARCHAR(255) | NOT NULL | E-mail do colaborador |
| tipo_atendimento | VARCHAR(255) | NOT NULL | Tipo de atendimento |
| local_atendimento | VARCHAR(255) | NOT NULL | Local de atendimento |
| nome_clinica_hospital_laboratorio | VARCHAR(255) | NULL | Clínica / hospital / laboratório |
| nome_medico_dentista | VARCHAR(255) | NULL | Médico / dentista |
| afastamento_um_ou_mais_dias | VARCHAR(3) | NOT NULL | `SIM` ou `NÃO` |
| atendimento_horas_data | DATE | NULL | Data do atendimento (fluxo horas) |
| atendimento_horas_inicio | TIME | NULL | Hora início (fluxo horas) |
| atendimento_horas_termino | TIME | NULL | Hora término (fluxo horas) |
| afastamento_dias_data_inicial | DATE | NULL | Data inicial (fluxo dias) |
| afastamento_dias_hora_inicio | TIME | NULL | Hora início (fluxo dias) |
| afastamento_dias_hora_termino | TIME | NULL | Hora término (fluxo dias) |
| afastamento_dias_duracao | INTEGER | NULL | Duração em dias |
| afastamento_dias_data_retorno | DATE | NULL | Data de retorno ao trabalho |
| afastamento_dias_cid | VARCHAR(20) | NULL | CID |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT CURRENT_TIMESTAMP | Criação do registro |

**Índices:**
- `idx_atestados_phone` em `phone_number`.
- `idx_atestados_timestamp` em `timestamp`.

### Prisma Migrations

| Migration | Descrição |
| --------- | --------- |
| `create_authorized_numbers` | Cria tabela `authorized_numbers` com índice único em `phone_number` |
| `create_conversations` | Cria tabela `conversations` com índice parcial em conversas ativas |
| `create_atestados` | Cria tabela `atestados` com índices em `phone_number` e `timestamp` |
| `seed_admin_numbers` | Seed: insere números admin e usuários iniciais |

---

## 4. Fluxo da Conversa — Máquina de Estados

O chatbot opera como uma máquina de estados. Cada conversa tem um `current_step` e um objeto `data` (JSONB) que acumula as respostas.

### Estados e Transições

```text
INICIO
  ↓ (mensagem de boas-vindas + pergunta unidade)
UNIDADE_TRABALHO
  ↓
MATRICULA
  ↓
NOME_COMPLETO
  ↓
EMAIL
  ↓
TIPO_ATENDIMENTO
  ↓
LOCAL_ATENDIMENTO
  ↓
NOME_CLINICA (opcional, pode pular)
  ↓
NOME_MEDICO (opcional, pode pular)
  ↓
AFASTAMENTO_DIAS_PERGUNTA
  ↓ SIM                    ↓ NÃO
AFASTAMENTO_DATA_INICIAL   HORAS_DATA_ATENDIMENTO
  ↓                          ↓
AFASTAMENTO_HORA_INICIO     HORAS_INICIO
  ↓                          ↓
AFASTAMENTO_HORA_TERMINO    HORAS_TERMINO
  ↓                          ↓
AFASTAMENTO_DURACAO_DIAS    CONFIRMACAO
  ↓
AFASTAMENTO_DATA_RETORNO
  ↓
AFASTAMENTO_CID
  ↓
CONFIRMACAO
  ↓ (confirma ou cancela)
FINALIZADO
```

### Regras de Validação por Etapa

| Campo | Validação |
| ----- | --------- |
| E-mail | Formato válido (regex básico) |
| Datas | Formato `DD/MM/AAAA`, data válida |
| Horas | Formato `HH:MM`, hora válida |
| Duração em dias | Número inteiro positivo |
| Afastamento SIM/NÃO | Aceitar variações: sim, s, não, nao, n |

### Timeout

- Conversas sem interação por mais de **30 minutos** devem ser marcadas como expiradas.
- Ao receber nova mensagem de um número com conversa expirada, inicia nova conversa.

---

## 5. Objeto do Atendimento

Ao concluir a conversa, o sistema monta o objeto estruturado:

```json
{
  "timestamp": "2026-03-13T14:30:00-03:00",
  "unidade_trabalho": "UNIDADE CENTRO",
  "matricula": "12345",
  "nome_completo": "JOÃO DA SILVA",
  "email": "joao@empresa.com",
  "tipo_atendimento": "Consulta médica",
  "local_atendimento": "UPA Centro",
  "nome_clinica_hospital_laboratorio": "Hospital São Lucas",
  "nome_medico_dentista": "Dr. Carlos",
  "afastamento_um_ou_mais_dias": "SIM",
  "atendimento_horas": {
    "data_atendimento": null,
    "hora_inicio": null,
    "hora_termino": null
  },
  "afastamento_dias": {
    "data_inicial": "13/03/2026",
    "hora_inicio": "08:00",
    "hora_termino": "09:00",
    "duracao_dias": "3",
    "data_retorno_trabalho": "16/03/2026",
    "cid": "J11"
  }
}
```

---

## 6. Exportação Excel

### Mapeamento de Colunas

| Campo no objeto | Coluna na planilha | Regra |
| --------------- | ------------------ | ----- |
| timestamp | Carimbo de data/hora | Automático |
| unidade_trabalho | ASSINALE A SUA UNIDADE DE TRABALHO | Obrigatório |
| matricula | MATRÍCULA | Obrigatório |
| nome_completo | Nome Completo (ESCREVER EM CAIXA ALTA) | Obrigatório |
| email | E-mail | Obrigatório |
| tipo_atendimento | TIPO DE ATENDIMENTO | Obrigatório |
| local_atendimento | LOCAL DE ATENDIMENTO | Obrigatório |
| nome_clinica_hospital_laboratorio | NOME DA CLÍNICA / HOSPITAL / LABORATÓRIO | Quando houver |
| nome_medico_dentista | NOME DO MÉDICO / DENTISTA | Quando informado |
| afastamento_um_ou_mais_dias | O ATESTADO É DE AFASTAMENTO COM (01) UM OU MAIS DIAS | Obrigatório |
| atendimento_horas.data_atendimento | DATA DO ATENDIMENTO (data da consulta) | Se NÃO |
| atendimento_horas.hora_inicio | HORA DO INÍCIO | Se NÃO |
| atendimento_horas.hora_termino | HORA DO TÉRMINO | Se NÃO |
| afastamento_dias.data_inicial | DATA DO INICIAL (dia do atendimento) | Se SIM |
| afastamento_dias.hora_inicio | HORA INICIO (início da consulta) | Se SIM |
| afastamento_dias.hora_termino | HORA TÉRMINO (término da consulta) | Se SIM |
| afastamento_dias.duracao_dias | DURAÇÃO ESTIMADA DO TRATAMENTO EM DIAS - ABONO (apenas números) | Se SIM |
| afastamento_dias.data_retorno | DATA RETORNO AO TRABALHO | Se SIM |
| afastamento_dias.cid | CID | Se SIM |

### Regras de exportação

- Cada atendimento gera uma linha na planilha.
- Campos comuns são sempre preenchidos.
- Campos de horas OU de dias são preenchidos conforme `afastamento_um_ou_mais_dias`.
- O campo de mensagem fixa do formulário original (Google Forms) não é incluído.
- O comando `/exportar` aceita filtros opcionais: período, unidade.
- Somente números com role `ADMIN` podem executar `/exportar`.
- O arquivo `.xlsx` é enviado como documento via WhatsApp.

---

## 7. Integração WhatsApp Business API (Meta Cloud API)

### Endpoints utilizados

| Operação | Endpoint Meta |
| -------- | ------------- |
| Verificação de webhook | `GET /webhook` (challenge verification) |
| Receber mensagens | `POST /webhook` (message notifications) |
| Enviar mensagem de texto | `POST /v21.0/{phone-number-id}/messages` |
| Enviar documento | Upload media + `POST /v21.0/{phone-number-id}/messages` (type: document) |

### Segurança do Webhook

- Verificar `X-Hub-Signature-256` em todo `POST` recebido usando `WHATSAPP_APP_SECRET`.
- Rejeitar requests sem assinatura válida.
- Verificação do webhook (`GET`) usa `WHATSAPP_VERIFY_TOKEN` como challenge.

### Variáveis de ambiente obrigatórias

| Variável | Descrição |
| -------- | --------- |
| `WHATSAPP_TOKEN` | Token de acesso permanente da Meta API |
| `WHATSAPP_PHONE_NUMBER_ID` | ID do número de telefone do WhatsApp Business |
| `WHATSAPP_VERIFY_TOKEN` | Token customizado para verificação do webhook |
| `WHATSAPP_APP_SECRET` | App Secret para validação de assinatura |

### Pré-requisitos externos (fora do código)

1. Criar conta Meta Business.
2. Criar app no Meta for Developers.
3. Ativar o produto WhatsApp no app.
4. Configurar número de telefone.
5. Gerar token de acesso permanente.
6. Configurar URL do webhook (requer HTTPS — usar ngrok em desenvolvimento).