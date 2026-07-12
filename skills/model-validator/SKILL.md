---
name: model-validator
description: "Valida modelos de dados relacionais de forma profunda — entidades, relacionamentos, normalização, índices, queries, performance e strategy. Gera relatório detalhado de problemas e plano de implementação. USE PARA: validar modelo de dados, revisar schema EF Core, auditar índices, sugerir índices, otimizar queries, verificar normalização, checar relações N+1, validar mapeamento ORM, auditar multi-tenancy no modelo, planejar migrações. NÃO USE PARA: escrever specs de produto (use /spec), gerar documentação técnica (use /docs), deploy de infraestrutura."
---

# Model Validator — Validação de Modelo de Dados Relacional

Skill especialista em validar, auditar e otimizar modelos de dados relacionais. Analisa o modelo completo (entidades, value objects, configurações EF Core, migrations, queries) e gera um **relatório detalhado** com problemas categorizados por severidade e um **plano de implementação** com prioridades.

## Escopo de Análise

A validação cobre **7 dimensões**, cada uma com checklists específicos:

---

### 1. ESTRUTURA ENTIDADE-RELACIONAMENTO

#### 1.1 Entidades e Identidade

```
CHECKLIST:
☐ Todo AggregateRoot herda de AggregateRoot (Guid Id, CreatedAt, UpdatedAt)?
☐ Todo entity tem construtor privado parameterless para EF Core?
☐ Todo entity tem factory method static Create(...)?
☐ PKs são Guid (nunca int/long) e geradas no domínio (nunca pelo banco)?
☐ Child entities (não-AggregateRoot) herdam a PK do pai ou têm proprio Id?
☐ Nenhuma entidade usa ValueGeneratedOnAdd (exceto value objects)?
```

**Regras de validação:**
- Se uma entidade herda `AggregateRoot` mas não tem `Guid Id` → ERRO
- Se uma entidade tem `int Id` ou `long Id` → ERRO CRÍTICO (convenção violada)
- Se uma entidade child (não AggregateRoot) não tem `Id` e não é Owned → ALERTA
- Se uma entidade não tem construtor privado parameterless → WARN (EF Core pode falhar)

#### 1.2 Relacionamentos

```
CHECKLIST:
☐ FKs são explícitas (propriedade Guid XyzId + navegação Xyz)?
☐ Todo FK tem índice (mesmo que implícito via navigation)?
☐ Relacionamentos 1:N usam HasMany().WithOne().HasForeignKey()?
☐ Relacionamentos 1:1 usam HasOne().WithOne().HasForeignKey()?
☐ Owned types usam OwnsOne() ou OwnsMany()?
☐ Nenhum loop de referência circular entre entidades?
☐ Cascade delete configurado corretamente (não Deleted por engano)?
☐ Navegações lazy loading estão desabilitadas (explicitamente)?
```

**Regras de validação:**
- FK sem índice → ERRO (perfis de query lentos)
- Cascade delete em entidade que não é child → ALERTA (risco de exclusão em cascata indevida)
- Navegação sem FK explícita → WARN (EF Core gera shadow property)
- Relacionamento sem configuração EF → ERRO (convenção pode mapear errado)

#### 1.3 Normalização

```
CHECKLIST:
☐ 1FN: Colunas são atômicas (não contêm listas, JSONs não estruturados)?
☐ 2FN: Atributos parciais dependem da PK completa?
☐ 3FN: Não há atributos que dependem de não-chave?
☐ Colunas JSON (jsonb) são justificadas (dados semi-estruturados)?
☐ Tabelas de junção desnecessárias foram evitadas?
☐ Colunas de enum são armazenadas como string ou int (não varchar livre)?
```

**Regras de validação:**
- Campo JSON sem justificativa → ALERTA (perde query capability)
- Coluna enum como VARCHAR sem CHECK constraint → WARN
- Tabela de junção sem cascade adequado → ERRO

---

### 2. MULTI-TENANCY

```
CHECKLIST:
☐ Toda entidade tenant-scoped implementa ITenantScoped?
☐ Nenhuma entidade tenant-scoped esquece o TenantId na criação?
☐ Global query filter está aplicado via reflexão no AppDbContext?
☐ Tenant (a entidade) NÃO implementa ITenantScoped?
☐ AIProviderConfig NÃO implementa ITenantScoped (SuperAdmin)?
☐ Value Objects herdam ITenantScoped quando necessário?
☐ Junction tables (ex: partner_student_consents) têm TenantId?
☐ Seed data respeita o multi-tenancy?
```

**Regras de validação:**
- Entidade tenant-scoped sem ITenantScoped → ERRO CRÍTICO (vazamento de dados)
- Tenant entity com ITenantScoped → ERRO (circular dependency no filter)
- Entidade sem global query filter que deveria ter → ALERTA
- Junction table sem TenantId → ERRO (dados misturados entre tenants)

---

### 3. ÍNDICES

#### 3.1 Índices Explícitos (EF Core)

```
CHECKLIST:
☐ Índices em colunas de FK (HasIndex(x => x.FkId))?
☐ Índices compostos para queries frequentes ( HasIndex(x => new { x.A, x.B }) )?
☐ Índices únicos para campos com constraint de unicidade (email, slug)?
☐ Índices parciais (WHERE clause) para filtered queries?
☐ Índices para ordenação frequente (CreatedAt DESC)?
☐ Nenhum índice duplicado (explícito + implícito)?
☐ Índices com INCLUDE para covering indexes?
```

#### 3.2 Análise de Índices Faltantes

Para cada entidade, verifique:

| Padrão de Query | Índice Necessário |
|---|---|
| `WHERE TenantId = X` | Global query filter já cobre (OK) |
| `WHERE TenantId = X AND Status = Y` | Índice composto (TenantId, Status) |
| `WHERE TenantId = X ORDER BY CreatedAt DESC` | Índice (TenantId, CreatedAt DESC) |
| `WHERE FK_Id = X` | Índice na FK |
| `WHERE FK_Id = X AND TenantId = Y` | Índice composto (FK_Id, TenantId) |
| `WHERE Email = X` | Índice único |
| `WHERE Slug = X` | Índice único |
| JSONB query (ex: `Trail`) | GIN index |

**Regras de validação:**
- FK sem índice explícito e sem query filter → ERRO
- Query filter + ORDER BY sem índice composto → WARN (performance)
- GIN index em jsonb sem uso → ALERTA (espaço desperdiçado)

#### 3.3 Índices do PostgreSQL

```
CHECKLIST:
☐ pgvector: IVFFlat ou HNSW index para Vector similarity?
☐ pgvector: Lists parameter calibrado (rows/1000 para IVFFlat)?
☐ JSONB: GIN index para colunas jsonb consultadas?
☐ Text search: GIN para full-text search?
☐ BRIN para colunas com ordem natural (timestamps, sequential IDs)?
☐ Partial indexes para status enums (ex: WHERE status = 'active')?
```

---

### 4. QUERIES E PERFORMANCE

#### 4.1 N+1 Detection

```
CHECKLIST:
☐ Nenhuma entidade carrega coleções via lazy loading?
☐ Usuários de Include() estão documentados (query projection)?
☐ Coleções paginadas usam Skip/Take + Count em query separada?
☐ Queries que carregam >3 níveis de profundidade (Include → Include → Include)?
☐ Entidades com muitas propriedades (wide tables > 20 colunas)?
```

#### 4.2 Query Patterns

```
CHECKLIST:
☐ Queries usam AsNoTracking() para read-only?
☐ Queries usam projection (Select) para campos necessários?
☐ Bulk operations usam ExecuteUpdate/ExecuteDelete (não carrega memória)?
☐ Soft delete está implementado (IsDeleted + query filter)?
☐ Paginação usa keyset (WHERE id > @lastId) ou OFFSET?
☐ Aggregate queries (COUNT, SUM, AVG) usam query separada?
```

#### 4.3 Mapping Eficiência

```
CHECKLIST:
☐ Value Objects usam OwnsOne() (não propriedades separadas)?
☐ Conversões de enum usam HasConversion<string>() ou HasConversion<int>()?
☐ JSON converters são eficientes (não allocam por row)?
☐ Colunas com default value são mapeadas na configuração?
☐ Timestamps são DateTimeOffset UTC (não DateTime)?
```

---

### 5. ESTRATÉGIA DE MODELO

#### 5.1 Adequação ao Domínio

```
CHECKLIST:
☐ Aggregate boundaries estão corretos (uma transação por aggregate)?
☐ Entidades Value (não-root) não são acessadas independentemente?
☐ Referências entre aggregates são por Id (não navegação direta)?
☐ Domain Events são usados para comunicação entre aggregates?
☐ Model está alignado com Bounded Contexts?
```

#### 5.2 Estratégia Recomendada

A skill deve recomendar uma das seguintes estratégias para cada módulo:

| Estratégia | Quando Aplicar |
|---|---|
| **Simples (1 tabela)** | Entidade sem dependências, poucas propriedades |
| **Aggregate com owned** | Entidade com coleções fracas (fases, blocos) |
| **Separate table** | Entidade com ciclo de vida próprio |
| **JSONB column** | Dados semi-estruturados, raramente consultados individualmente |
| **Table per hierarchy** | Hierarquia de tipos com polymorfismo |
| **Table per type** | Hierarquia com propriedades distintas por tipo |

---

### 6. SEGURANÇA DO MODELO

```
CHECKLIST:
☐ Nenhum campo de senha (hash) exposto em navegação/serialização?
☐ Colunas sensíveis (tokens, API keys) não aparecem em DTOs?
☐ Soft delete não expõe dados deletados sem filtro?
☐ Tenant isolation não pode ser bypassado por query manual?
☐ Colunas de audit (CreatedAt, UpdatedAt) não podem ser falsificadas?
☐ Migrations são idempotentes (Down() implementado)?
```

---

### 7. MIGRAÇÕES

```
CHECKLIST:
☐ Todas as migrations têm Down() implementado?
☐ Migrations são aditivas (não recriam tabelas existentes)?
☐ Índices são criados em migrations (não só noModelCreating)?
☐ Colunas NOT NULL têm default value na migration?
☐ Renames usam RenameColumn/RenameTable (não Drop + Create)?
☐ Migrations não contêm dados hardcoded (seed data separado)?
☐ Snapshot está atualizado com o estado atual do modelo?
```

---

## Workflow de Execução

### Passo 1: Coleta

```
1.1 Localizar todos os arquivos de entidade (Domain/Entities/)
1.2 Localizar todas as configurações EF Core (Persistence/)
1.3 Localizar o AppDbContext
1.4 Localizar migrations (se existirem)
1.5 Ler todos os arquivos relevantes em paralelo
```

### Passo 2: Análise por Dimensão

Executar as 7 dimensões em sequência. Para cada dimensão:
- Marcar cada item do checklist como ✅ (OK), ⚠️ (WARN), ❌ (ERRO), ou 🔴 (CRÍTICO)
- Coletar evidência (caminho do arquivo + linha)

### Passo 3: Geração do Relatório

O relatório DEVE seguir este formato exato:

```markdown
# Relatório de Validação do Modelo de Dados

**Data:** YYYY-MM-DD HH:MM
**Escopo:** [nomes dos módulos analisados]
**Total de entidades:** N
**Total de configurações EF:** N

## Resumo Executivo

| Severidade | Quantidade |
|---|---|
| 🔴 CRÍTICO | N |
| ❌ ERRO | N |
| ⚠️ WARN | N |
| ✅ OK | N |

## Score Geral: X/100

[Uma nota de 0-100 baseada na ponderação:
- Crítico = -20 pontos cada
- Erro = -10 pontos cada
- Warn = -3 pontos cada
- OK = 0 pontos]

---

## 🔴 CRÍTICOS (requerem ação imediata)

### C-001: [Título]
- **Dimensão:** [qual dimensão]
- **Arquivo:** caminho:linha
- **Problema:** [descrição clara]
- **Impacto:** [o que pode acontecer]
- **Correção:** [como resolver]

---

## ❌ ERROS

### E-001: [Título]
- **Dimensão:** [qual dimensão]
- **Arquivo:** caminho:linha
- **Problema:** [descrição clara]
- **Impacto:** [o que pode acontecer]
- **Correção:** [como resolver]

---

## ⚠️ WARNINGS

### W-001: [Título]
- **Dimensão:** [qual dimensão]
- **Arquivo:** caminho:linha
- **Problema:** [descrição clara]
- **Recomendação:** [sugestão de melhoria]

---

## ✅ APROVADOS (sem problemas)

[Lista das dimensões que passaram sem problemas]

---

## 📊 Análise de Índices

### Índices Existentes
| Tabela | Índice | Colunas | Tipo |
|---|---|---|---|

### Índices Recomendados
| Tabela | Colunas | Justificativa | Prioridade |
|---|---|---|---|

### Índices para Remover (duplicados/inesperados)
| Tabela | Índice | Motivo |
|---|---|---|

---

## 📊 Análise de Queries (se aplicável)

### Queries Identificadas no Código
| Query | Localização | Problema? |
|---|---|---|

### Otimizações Sugeridas
| Query Atual | Sugestão | Impacto |
|---|---|---|

---

## 📋 Plano de Implementação

### Fase 1 — Críticos (imediato)
| # | Ação | Arquivo | Esforço |
|---|---|---|---|

### Fase 2 — Erros (esta sprint)
| # | Ação | Arquivo | Esforço |
|---|---|---|---|

### Fase 3 — Warnings (próxima sprint)
| # | Ação | Arquivo | Esforço |
|---|---|---|---|

### Fase 4 — Otimizações (backlog)
| # | Ação | Arquivo | Esforço |
|---|---|---|---|

**Estimativa total de esforço:** X pontos (1 ponto = ~1 hora de trabalho)
```

### Passo 4: Execução (se solicitado)

Se o usuário pedir para executar as correções:
1. Criar branch de correção
2. Aplicar correções Fase 1 primeiro
3. Rodar `dotnet build` para validar
4. Criar migration se necessário
5. Rodar `dotnet test` para validar
6. Reportar resultado

---

## Comandos de Execução

```
/model-validator full           → Validação completa (7 dimensões)
/model-validator entities      → Só dimensão 1 (entidades)
/model-validator tenancy        → Só dimensão 2 (multi-tenancy)
/model-validator indexes        → Só dimensão 3 (índices)
/model-validator queries        → Só dimensão 4 (queries)
/model-validator strategy       → Só dimensão 5 (estratégia)
/model-validator security       → Só dimensão 6 (segurança)
/model-validator migrations     → Só dimensão 7 (migrations)
/model-validator fix            → Valida + executa correções aprovadas
/model-validator report         → Gera relatório sem executar nada
```

---

## Regras

- **Nunca executar correções sem aprovação explícita do usuário** (exceto com `/model-validator fix`).
- **Cada problema DEVE ter evidência** (arquivo + linha). Sem evidência, não é problema.
- **Severidade baseada em impacto real**, não em opinião estética.
- **Índices são sugeridos, não impostos** — o usuário decide após ver a justificativa.
- **Migrations são geradas como último passo**, nunca antes de validar o build.
- **Sempre rodar `dotnet build` e `dotnet test` após correções.**
- **Respeitar convenções existentes do projeto** (hexagonal, multi-tenancy, Guid PKs).
- **JSONB é válido** quando os dados são semi-estruturados e raramente filtrados individualmente.
- **pgvector indexes** devem calibrar `lists` e `probes` baseado no volume de dados.

---

## Referências

- [EF Core Indexing](https://learn.microsoft.com/ef/core/modeling/indexes)
- [PostgreSQL Indexing](https://www.postgresql.org/docs/current/indexes.html)
- [pgvector Indexing](https://github.com/pgvector/pgvector#indexing)
- [N+1 Problem](https://stackoverflow.com/questions/9719701/the-n1-query-problem)
