# Decisões Compartilhadas — Base do Projeto

> **Projeto:** starter-kit
> **Origem de referência:** LojaSync (`C:\projetos\lojasync`)
> **Data:** 08/03/2026
>
> Este documento registra decisões transversais extraídas do projeto de referência e adotadas, adaptadas ou conscientemente não adotadas no starter-kit.

---

## 1. Decisões Adotadas

| Decisão | Status | Aplicação na base |
| ------- | ------ | ----------------- |
| Timezone padrão `America/Sao_Paulo` | Adotada | Convenção operacional do projeto |
| PostgreSQL com `TIMESTAMPTZ` | Adotada | Colunas temporais persistidas com timezone |
| Backend preferindo `Instant` | Adotada | Persistência e processamento temporal interno |
| Flyway como fonte de verdade do schema | Adotada | Evolução estrutural por migration |
| `ddl-auto: validate` | Adotada | Sem geração automática de schema pelo Hibernate |
| `createdAt`, `updatedAt` e `version` em entidades relevantes | Adotada | Base estrutural do domínio |
| Concorrência otimista com `@Version` | Adotada | Prevenção de sobrescrita silenciosa |
| Backend como fonte de verdade de autorização | Adotada | Frontend não define permissão |
| Actuator e health checks obrigatórios | Adotada | Observabilidade mínima da plataforma |
| Backup/restore como preocupação arquitetural | Adotada | Quando houver estado fora do banco |
| Arquivos operacionais dentro do perímetro de segurança | Adotada | Backup, restore e operação |
| Testcontainers com PostgreSQL | Adotada | Testes mais próximos da produção |

---

## 2. Decisões Adaptadas

| Decisão de referência | Adaptação no starter-kit | Motivo |
| --------------------- | ------------------------ | ------ |
| `storeId` como parte estrutural de toda entidade | Contexto organizacional opcional por projeto | A base deve servir também para sistemas single-tenant |
| UUID como padrão universal de identificador | Base mantém `BIGSERIAL`; UUID fica como decisão consciente por projeto | Menor complexidade inicial e melhor aderência ao cenário local padrão |
| Estado fora do banco fortemente presente | Tratado como capacidade arquitetural, não obrigação inicial | O starter ainda não possui módulo que exija arquivos operacionais |

---

## 3. Decisões Não Adotadas por Padrão

| Decisão | Motivo |
| ------- | ------ |
| PWA como padrão do starter | Aumenta complexidade e não é requisito da base atual |
| API administrativa de backup/restore no baseline | Útil em sistemas operacionais mais complexos, mas desnecessária no ponto de partida |
| Dependência de `ZoneId.systemDefault()` | Gera ambiguidade operacional entre ambientes |
| CORS implícito ou desabilitado por conveniência | Preferência por configuração explícita e previsível |

---

## 4. Diretriz de Uso

Ao derivar novos projetos a partir desta base:

1. manter as decisões adotadas como padrão, salvo motivo técnico forte para mudança.
2. registrar qualquer mudança em data, segurança, persistência ou operação como decisão arquitetural explícita.
3. avaliar UUID, contexto organizacional e artefatos operacionais conforme a necessidade real do domínio.
