---
name: nosql-validator
description: "Valida modelos de dados NoSQL — document stores, key-value, column-family, graph databases. Analisa padrão de acesso, denormalização, particionamento, consistência, throughput e custo. Gera relatório detalhado e plano de implementação. USE PARA: validar modelo MongoDB, revisar schema DynamoDB, auditar partition key, review Cassandra model, Redis data structure design, Cosmos DB container design, Firebase Firestore rules, model validation, access pattern analysis, NoSQL capacity planning. NÃO USE PARA: modelos relacionais (use /model-validator), migrações SQL (use /model-validator migrations), deploy de infraestrutura."
---

# NoSQL Validator — Validação de Modelo de Dados NoSQL

Skill especialista em validar, auditar e otimizar modelos de dados NoSQL. Analisa o modelo completo (estrutura de documentos, padrões de acesso, particionamento, replicação, consistência, custo) e gera um **relatório detalhado** com problemas categorizados e um **plano de implementação**.

## Database Engines Suportados

| Engine | Tipo | Checklist Específico |
|---|---|---|
| **MongoDB** | Document | Seção MongoDB |
| **DynamoDB** | Key-Value / Wide-column | Seção DynamoDB |
| **Cassandra / ScyllaDB** | Column-family | Seção Cassandra |
| **Redis** | Key-Value / Structure | Seção Redis |
| **Cosmos DB** | Multi-model | Seção Cosmos DB |
| **Firestore** | Document | Seção Firestore |
| **Neo4j / ArangoDB** | Graph | Seção Graph |

---

## Checklists Universais (qualquer NoSQL)

### 1. PADRÕES DE ACESSO (Access Patterns)

```
CHECKLIST:
☐ Todos os padrões de acesso foram mapeados ANTES de desenhar o modelo?
☐ Cada query foi listada com: frequência, latência requerida, throughput requerido?
☐ O modelo suporta TODOS os padrões de acesso sem scan?
☐ Padrões de escrita foram mapeados com a mesma atenção dos de leitura?
☐ Hot partitions foram identificados (padrões com concentração de tráfego)?
☐ Padrões de acesso futuros (6-12 meses) foram considerados?
```

**Regras de validação:**
- Modelo desenhado sem mapeamento de access patterns → ERRO CRÍTICO
- Query que requer scan full → ERRO (escalabilidade comprometida)
- Padrão de acesso identificado mas sem índice/estrutura adequada → ERRO

### 2. DENORMALIZAÇÃO

```
CHECKLIST:
☐ A denormalização é justificada por padrão de acesso (não por preguiça)?
☐ Dados duplicados têm estratégia de sincronização (write-through, event-driven)?
☐ Dados denormalizados têm fonte de verdade (source of truth) definida?
☐ O custo de escrita duplicada foi avaliado vs custo de leitura?
☐ Documentos embedados têm tamanho estimado (não crescem indefinidamente)?
☐ Referências (lookup) são usadas quando embedação não é justificável?
☐ Sub-documentos embedados têm lifecycle similar ao documento pai?
```

**Regras de validação:**
- Embedação sem justificativa de access pattern → ALERTA
- Documento sem fonte de verdade definida → ERRO (dados inconsistentes)
- Embedação de coleção que cresce ilimitadamente → ERRO CRÍTICO
- Lookup sem índice na coleção referenciada → ERRO

### 3. CONSISTÊNCIA E TRANSACIONALIDADE

```
CHECKLIST:
☐ O nível de consistência foi definido por operação (strong, eventual, causal)?
☐ Operações multi-documento usam transações quando necessário?
☐ Write-and-read consistente (read-after-write) foi validado?
☐ Conflitos de escrita concorrente foram considerados?
☐ TTL (Time To Live) foi aplicado onde dados expiram?
☐ Versioning de documentos foi implementado quando necessário?
☐ Saga pattern ou outbox pattern para operações distribuídas?
```

**Regras de validação:**
- Operação financeira com consistência eventual → ERRO CRÍTICO
- Dados que expiram sem TTL → WARN (crescimento infinito)
- Escrita concorrente sem conflito resolution → ALERTA

### 4. SEGURANÇA

```
CHECKLIST:
☐ Regras de acesso (Firestore Rules, IAM, VPC) estão configuradas?
☐ Dados sensíveis não estão em campos plaintext?
☐ Chaves de criptografia estão gerenciadas (KMS, não hardcoded)?
☐ Audit log está habilitado para operações críticas?
☐ Tenant isolation está implementada (collection per tenant, filter, ou DDB)?
☐ Backups estão configurados e testados?
☐ Point-in-time recovery está habilitado?
```

---

## Checklist MongoDB

### 5.1 Estrutura de Documentos

```
CHECKLIST:
☐ Documentos têm schema definido (JSON Schema ou Mongoose/Zod)?
☐ Tamanho do documento < 16MB (limite BSON)?
☐ Campos de busca frequentes são top-level (não aninhados profundos)?
☐ Arrays têm tamanho estimado e limitado?
☐ Embedded documents < 100KB quando possível?
☐ ObjectId vs UUID: decisão documentada e consistente?
☐ Campos de metadata (_id, createdAt, updatedAt) estão presentes?
☐ Named fields > numbered fields (não usar array para dados estruturados)?
```

### 5.2 Índices

```
CHECKLIST:
☐ Índices cobrem todas as queries do padrão de acesso?
☐ Índices compostos seguem a regra ESR (Equality, Sort, Range)?
☐ Índices parciais (partial filter) para queries filtradas?
☐ Índices multikey para queries em arrays?
☐ Nenhum índice duplicado ou não utilizado?
☐ Índices textuais para full-text search?
☐ Explain plan analisado para cada query critica?
☐ Compound indexes cobrem sort order (não somente equality)?
```

**Regra ESR (Equality-Sort-Range):**
```
Índice composto deve seguir: [campos de igualdade] + [campos de ordenação] + [campos de range]

Exemplo:
Query: { status: "active", createdAt: { $gt: date } }.sort({ score: -1 })
Índice: { status: 1, score: -1, createdAt: 1 }
         ^equality  ^sort      ^range
```

### 5.3 Sharding

```
CHECKLIST:
☐ Shard key escolhido tem alta cardinalidade?
☐ Shard key distribui carga uniformemente (não causa hot shard)?
☐ Shard key permite queries com scatter-gather mínimo?
☐ Shard key é imutável (não muda ao longo do tempo)?
☐ Zone sharding foi considerado para dados geográficos?
☐ Chunks balancing está habilitado e monotorado?
☐ Queries sem shard key foram mapeadas (scatter-gather)?
```

**Regras de validação:**
- Shard key com baixa cardinalidade (ex: status enum) → ERRO CRÍTICO
- Shard key que causa hot shard (ex: timestamp para writes recentes) → ERRO
- Queries frequentes sem shard key → ALERTA (performance degradada)

### 5.4 Padrões de Acesso MongoDB

| Padrão | Estrutura Recomendada |
|---|---|
| Busca por ID | Documento direto |
| Busca por campo único | Índice nesse campo |
| Busca por múltiplos campos | Índice composto (ESR) |
| Busca em array | Multikey index |
| Full-text search | Atlas Search / text index |
| Busca geoespacial | 2dsphere index |
| Agregação complexa | Aggregation pipeline + indexes |
| Relação 1:N com leitura junto | Embedded document |
| Relação 1:N com leitura separada | Reference + lookup |
| Relação N:M | Reference bidirecional ou junction collection |
| Dados temporários com expiração | TTL index |

---

## Checklist DynamoDB

### 6.1 Partition Key e Sort Key

```
CHECKLIST:
☐ Partition key tem alta cardinalidade (mínimo 10x o número de partitions)?
☐ Sort key permite range queries para o padrão de acesso?
☐ Composite sort key (SK) foi usado para múltiplos padrões em uma tabela?
☐ Partition key não é um campo que muda (não updateable)?
☐ Oversized partitions (< 10GB) foram estimados?
☐ Single-table design foi avaliado vs multi-table?
```

### 6.2 Single-Table Design

```
CHECKLIST:
☐ Access patterns foram listados ANTES de definir PK/SK?
☐ Cada item type tem PK/SK pattern documentado:
  - PK: ENTITY_TYPE#id
  - SK: METADATA | RELATED_TYPE#id | SORT_KEY
☐ GSI (Global Secondary Indexes) foram minimizados (máx 20 por tabela)?
☐ GSI usam projeção adequada (KEYS_ONLY, INCLUDE, ALL)?
☐ LSI (Local Secondary Indexes) foram evitados (limitação de 5 por tabela)?
☐ Sparse indexes foram usados para queries com baixa selectivity?
☐ Hot partitions em GSIs foram avaliados?
```

### 6.3 Capacity e Custos

```
CHECKLIST:
☐ Read capacity units (RCU) foram estimados por padrão de acesso?
☐ Write capacity units (WCU) foram estimados por padrão de acesso?
☐ On-demand mode vs provisioned mode foi avaliado?
☐ Auto-scaling está configurado?
☐ DAX (DynamoDB Accelerator) foi considerado para read-heavy workloads?
☐ DynamoDB Streams foi habilitado para change data capture?
☐ TTL está habilitado para dados que expiram?
☐ Backup point-in-time recovery está habilitado?
☐ Custos mensais estimados (read + write + storage + DAX)?
```

### 6.4 Padrões de Acesso DynamoDB

| Padrão | Estrutura |
|---|---|
| Get item by ID | GetItem (PK + SK) |
| Query by partition | Query (PK = X) |
| Query by partition + range | Query (PK = X AND SK BETWEEN A AND B) |
| Scan by type | GSI ou sparse index |
| Aggregation | DynamoDB Streams + Lambda ou DAX |
| Relational join | Single-table design com composite SK |
| Time-series data | SK = TIMESTAMP#sort_key |
| Leaderboard | GSI + SK (score-based) |
| Many-to-many | Junction item com PK_A + SK_B e PK_B + SK_A |

---

## Checklist Cassandra / ScyllaDB

### 7.1 Modelagem

```
CHECKLIST:
☐ Queries foram desenhadas ANTES da tabela (query-first modeling)?
☐ Cada tabela suporta EXATAMENTE um padrão de acesso?
☐ Partition key distribui dados uniformemente?
☐ Clustering columns definem a ordem de leitura correta?
☐ Tabelas não duplicadas desnecessariamente (data duplication justificada)?
☐ Counter tables foram usadas para contadores atômicos?
☐ Static columns foram usadas para dados partilhados na partition?
```

### 7.2 Particionamento

```
CHECKLIST:
☐ Partition size < 100MB (ideal < 10MB)?
☐ Partições não são "large partitions" (milhares de rows)?
☐ Token-aware routing está habilitado no driver?
☐ Replication factor é adequado (mínimo 3 para produção)?
☐ NetworkTopologyStrategy é usado (não SimpleStrategy)?
☐ Vnodes estão configurados corretamente?
```

### 7.3 Consistência

```
CHECKLIST:
☐ CL (Consistency Level) definido por operação:
  - ONE: writes rápidos, read-after-write não garantido
  - QUORUM: strong consistency com RF >= 3
  - ALL: máxima consistência, menor disponibilidade
☐ Tunable consistency está documentada por query?
☐ Lightweight transactions (LWT) usadas para conditional writes?
☐ Batch statements são usadas para atomicidade (LOGGED batch)?
```

---

## Checklist Redis

### 8.1 Estrutura de Dados

```
CHECKLIST:
☐ Tipo de dado escolhido é o mais adequado (String, Hash, List, Set, Sorted Set, Stream, HyperLogLog)?
☐ Tamanho dos values < 512KB (ideal < 1KB)?
☐ Keys seguem padrão consistente: {entity}:{id}:{subkey}?
☐ Keys têm TTL definido (nenhum key sem expiração em produção)?
☐ Memory usage estimado por key e total?
☐ Eviction policy adequada (allkeys-lru, volatile-lru, noeviction)?
☐ Big keys foram detectadas (> 1MB ou > 10K elements)?
```

### 8.2 Padrões de Acesso

```
CHECKLIST:
☐ Read-heavy: data cached com invalidation strategy definida?
☐ Write-through vs write-behind vs cache-aside escolhido?
☐ Cache stampede foi prevenido (mutex, lock, probabilistic early recomputation)?
☐ Hot keys foram identificados e shardadas (Redis Cluster)?
☐ Lua scripts usadas para atomicidade multi-key?
☐ Pub/Sub para real-time messaging foi avaliado vs Streams?
☐ Rate limiting está implementado corretamente?
```

### 8.3 Redis Cluster

```
CHECKLIST:
☐ Cluster mode habilitado para produção?
☐ Mínimo 6 masters (3 masters + 3 replicas)?
☐ Hash slots são distribuídos uniformemente?
☐ Multi-key operations usam hash tags {same_slot}?
☐ Cluster está monotorado (memory, connections, rejected commands)?
☐ Failover automático foi testado?
```

---

## Checklist Cosmos DB

### 9.1 Container Design

```
CHECKLIST:
☐ Partition key escolhido tem cardinalidade adequada?
☐ Throughput provisionado (RU/s) foi estimado por operação?
☐ Autoscale está habilitado?
☐ Stored procedures foram usadas para operações atômicas?
☐ UDFs (User Defined Functions) foram usadas para lógica server-side?
☐ Triggers foram usados para validação pré/pós-escrita?
☐ Partições não excedem 10GB?
```

### 9.2 Padrões de Acesso

```
CHECKLIST:
☐ Cross-partition queries foram minimizadas?
☐ Queries usam partition key no filtro quando possível?
☐ SDKs usam partition key nos point reads?
☐ Items de diferentes tipos coexistem no mesmo container (multi-tenant pattern)?
☐ Change feed foi habilitado para sincronização?
☐ TTL está habilitado para dados temporários?
```

---

## Checklist Firestore

### 10.1 Estrutura

```
CHECKLIST:
☐ Coleções e documentos seguem a hierarquia de dados?
☐ Subcoleções são usadas para dados fortemente acoplados ao pai?
☐ Referências (DocumentReference) são usadas para dados compartilhados?
☐ Documentos < 1MB (limite Firestore)?
☐ Nests aninhados < 5 níveis?
☐ Regras de segurança (Firestore Rules) estão definidas?
☐ Índices compostos foram criados para queries multi-campo?
```

### 10.2 Padrões de Acesso

```
CHECKLIST:
☐ Queries usam filtros suportados por índices?
☐ Não há queries que requerem múltiplas passes no cliente?
☐ Batch writes são usadas para múltiplas escritas atômicas?
☐ Transactions são usadas para reads + writes consistentes?
☐ Shallow queries (limit 1) para listagens de coleção?
☐ Pagination usa cursor (startAfter) não offset?
☐ Real-time listeners têm escopo definido (não escutam coleção inteira)?
```

---

## Checklist Graph (Neo4j / ArangoDB)

### 11.1 Modelagem

```
CHECKLIST:
☐ Nós representam entidades, relações representam adjacências?
☐ Propriedades em nós E relações foram usadas corretamente?
☐ Labels foram definidas para cada tipo de nó?
☐ Tipos de relação foram definidos e documentados?
☐ Direction das relações foi definida (dirigida vs não-dirigida)?
☐ Fan-out patterns foram mapeados (query que espalha por muitos nós)?
☐ Graph patterns (traverse, shortest path, community detection) foram mapeados?
```

### 11.2 Performance

```
CHECKLIST:
☐ Índices em propriedades de busca frequente criados?
☐ Índices compostos para propriedades filtradas frequentemente?
☐ Query depth limitada (< 5 hops para queries online)?
☐ Batch operations usam UNWIND (não loop no cliente)?
☐ Expensive queries usam cached results ou pre-computation?
☐ APOC ou extensões customizadas foram avaliadas?
☐ Memory limits para queries de traverse definidos?
```

---

## Workflow de Execução

### Passo 1: Coleta

```
1.1 Identificar o engine NoSQL utilizado (MongoDB, DynamoDB, etc.)
1.2 Localizar arquivos de schema/model/definition:
    - MongoDB: Mongoose schemas, JSON Schema, collection structures
    - DynamoDB: table definitions, CloudFormation/Terraform, access patterns doc
    - Cassandra: CQL files, schema definitions
    - Redis: key patterns, data structure usage
    - Cosmos DB: container definitions, stored procedures
    - Firestore: rules files, collection structure
    - Graph: node/relationship definitions, Cypher/Gremlin queries
1.3 Localizar configurações de infraestrutura (Terraform, CloudFormation, CDK)
1.4 Localizar queries/operations no código-fonte
1.5 Ler todos os arquivos relevantes em paralelo
```

### Passo 2: Análise

Executar os checklists universais + checklist específico do engine.
Para cada item:
- ✅ OK — conformidade verificada
- ⚠️ WARN — recomendação de melhoria
- ❌ ERRO — problema que afeta funcionamento
- 🔴 CRÍTICO — problema que causa falha em produção

### Passo 3: Relatório

O relatório DEVE seguir este formato:

```markdown
# Relatório de Validação NoSQL

**Data:** YYYY-MM-DD HH:MM
**Engine:** [MongoDB / DynamoDB / Cassandra / Redis / Cosmos DB / Firestore / Neo4j]
**Coleções/Tables:** N
**Operações mapeadas:** N

## Resumo Executivo

| Severidade | Quantidade |
|---|---|
| 🔴 CRÍTICO | N |
| ❌ ERRO | N |
| ⚠️ WARN | N |
| ✅ OK | N |

## Score Geral: X/100

[Ponderação:
- Crítico = -20 pontos cada
- Erro = -10 pontos cada
- Warn = -3 pontos cada
- OK = 0 pontos]

---

## 🔴 CRÍTICOS

### C-001: [Título]
- **Checklist:** [dimensão do checklist]
- **Localização:** arquivo:linha (ou nome da collection/table)
- **Problema:** [descrição clara]
- **Impacto:** [o que pode acontecer em produção]
- **Correção:** [como resolver, com código quando aplicável]

---

## ❌ ERROS

### E-001: [Título]
- **Checklist:** [dimensão]
- **Localização:** arquivo:linha
- **Problema:** [descrição]
- **Impacto:** [consequência]
- **Correção:** [solução]

---

## ⚠️ WARNINGS

### W-001: [Título]
- **Checklist:** [dimensão]
- **Localização:** arquivo:linha
- **Problema:** [descrição]
- **Recomendação:** [sugestão]

---

## ✅ APROVADOS

[Dimensões sem problemas]

---

## 📊 Análise de Padrões de Acesso

### Padrões Mapeados
| # | Padrão | Frequência | Tipo | Estrutura Adequada? |
|---|---|---|---|---|
| 1 | [descrição] | [QPS estimado] | [read/write] | [✅/❌] |

### Padrões Não Suportados pelo Modelo
| # | Padrão | Problema | Solução |
|---|---|---|---|

---

## 📊 Análise de Índices

### Índices Existentes
| Collection/Table | Índice | Colunas/Fields | Tipo |
|---|---|---|---|

### Índices Recomendados
| Collection/Table | Fields | Justificativa | Prioridade |
|---|---|---|---|

### Índices para Remover
| Collection/Table | Índice | Motivo |
|---|---|---|

---

## 📊 Análise de Capacidade (DynamoDB/Cosmos DB)

| Operação | Read/Write | RU estimado | Frequência | Total/mês |
|---|---|---|---|---|

**Custo estimado:** $XX/mês

---

## 📊 Análise de Tamanho

| Collection/Table | Documentos | Tamanho médio | Tamanho total estimado |
|---|---|---|---|

---

## 📋 Plano de Implementação

### Fase 1 — Críticos (imediato)
| # | Ação | Localização | Esforço |
|---|---|---|---|

### Fase 2 — Erros (esta sprint)
| # | Ação | Localização | Esforço |
|---|---|---|---|

### Fase 3 — Warnings (próxima sprint)
| # | Ação | Localização | Esforço |
|---|---|---|---|

### Fase 4 — Otimizações (backlog)
| # | Ação | Localização | Esforço |
|---|---|---|---|

**Estimativa total:** X pontos (1 ponto = ~1 hora)
```

### Passo 4: Execução (se solicitado)

Se o usuário pedir para executar:
1. Aplicar correções Fase 1
2. Validar com testes/integração
3. Criar índices necessários
4. Atualizar schemas
5. Rodar migrations de schema se aplicável
6. Reportar resultado

---

## Comandos

```
/nosql-validator full              → Validação completa
/nosql-validator access-patterns   → Só padrões de acesso
/nosql-validator indexes           → Só índices
/nosql-validator capacity          → Só análise de capacidade/custo
/nosql-validator security          → Só segurança
/nosql-validator size              → Só análise de tamanho
/nosql-validator fix               → Valida + executa correções
/nosql-validator report            → Gera relatório sem executar
```

---

## Regras

- **Padrões de acesso primeiro.** Sem access patterns mapeados, não há modelo correto.
- **Denormalização é ferramenta, não pecado.** Justificar com dados de acesso, não com preguiça.
- **Cada engine tem suas limitações.** Respeitar limits documentados (16MB BSON, 10GB partition, etc.).
- **Custo importa.** Sempre estimar impacto financeiro de índices, throughput e storage.
- **Nunca executar sem aprovação** (exceto com `/nosql-validator fix`).
- **Cada problema DEVE ter evidência.**
- **Testar operações críticas** antes de confirmar que modelo funciona.
- **Respeitar convenções do projeto** quando existentes.
- **Big keys, hot partitions, e large partitions** são os inimigos #1 de NoSQL em produção.

---

## Referências

- [MongoDB Schema Design Best Practices](https://www.mongodb.com/blog/post/building-with-patterns-a-summary)
- [DynamoDB Single-Table Design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/primary-key-index-alt.html)
- [Data Modeling in Apache Cassandra](https://cassandra.apache.org/doc/latest/cassandra/data_modeling/)
- [Redis Data Structures](https://redis.io/docs/data-types/)
- [Cosmos DB Partitioning](https://learn.microsoft.com/azure/cosmos-db/partitioning-overview)
- [Firestore Data Model](https://firebase.google.com/docs/firestore/data-model)
- [Neo4j Data Modeling](https://neo4j.com/docs/getting-started/graph-modeling/)
