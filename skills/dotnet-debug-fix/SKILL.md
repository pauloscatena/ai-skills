---
name: dotnet-debug-fix
description: >
  Analisa causa raiz de bugs .NET e gera plano de correção estruturado,
  apresentando melhor prática vs fix mínimo com trade-offs explícitos.
  Valida correção com build + testes. USE PARA: plano de correção, fix
  planning, root cause analysis, correção de bug .NET, validação de fix,
  melhor prática vs fix mínimo, recomendar testes de regressão. NÃO USE
  PARA: debug interativo com breakpoints (usar dotnet-debug-breakpoints),
  debug em Docker (usar dotnet-debug-docker), diagnóstico post-mortem
  (usar dotnet-debug-diagnostics), code review genérico (usar code-review).
---

# .NET Debug Fix — Análise de Causa Raiz e Plano de Correção

Skill especialista em transformar evidência de debug em plano de correção concreto: causa raiz, correção arquivo por arquivo, melhor prática vs fix mínimo com trade-offs, e validação com build + testes.

## Metodologia

### 1. Consolidar evidência

Receba de qualquer skill de debug anterior (breakpoints, docker, diagnostics):
- **Sintoma**: o que o usuário reportou
- **Ambiente**: como foi executado (local, Docker, container)
- **Evidência concreta**: stack trace, valores de variáveis, arquivo:linha
- **Causa raiz identificada**: o que está errado e por quê

Se a causa raiz ainda não foi identificada, **não invente uma** — volte para a skill de debug apropriada.

### 2. Analisar o impacto

Para cada arquivo/linha afetada:

```
☐ Qual é o escopo do bug? (afeta 1 método? 1 classe? 1 fluxo inteiro?)
☐ Quais outros pontos do código são afetados indiretamente?
☐ O bug tem potencial de causar data corruption ou security issue?
☐ Existem testes existentes que deveriam ter pego isso?
```

### 3. Definir correção

Para cada correção, apresente **duas opções** quando aplicável:

#### Opção A: Melhor prática (mais robusta)
- Implementação completa que resolve causa raiz
- Cobre edge cases
- Segue padrões do projeto
- Mais esforço, menos dívida técnica

#### Opção B: Fix mínimo (resolve só o sintoma)
- Menor mudança possível para parar de quebrar
- Pode não cobrir edge cases
- Mais rápido de implementar
- Mais dívida técnica

**Trade-off explícito:**

| Critério | Melhor prática | Fix mínimo |
|---|---|---|
| Esforço | X horas | Y minutos |
| Risco de regressão | Baixo | Médio |
| Cobre edge cases | Sim | Não |
| Dívida técnica | Nenhuma | Média |
| Manutenção futura | Fácil | Difícil |

> A decisão final é sempre do usuário. Sua responsabilidade é garantir que a melhor opção esteja na mesa.

### 4. Plano arquivo por arquivo

```markdown
### Correção 1: [Título curto]

**Arquivo:** `src/Module/Service.cs`
**Linha(s):** 42-55

**Causa raiz:** [uma frase]
**Melhor prática:** [descrição]
**Fix mínimo:** [descrição]

#### Passos (melhor prática)
1. Adicionar validação de null antes de acessar propriedade (linha 42)
2. Extrair método `ValidateInput()` para reutilização
3. Adicionar unit test para o caso nulo

#### Passos (fix mínimo)
1. Adicionar `if (x == null) return;` na linha 42

#### Riscos
- [risco 1]
- [risco 2]
```

### 5. Recomendar testes de validação

```
☐ Unit test para o cenário que causava o bug
☐ Unit test para edge cases relacionados
☐ Integration test se o bug envolvia interação entre módulos
☐ Rodar suíte existente para verificar regressão
```

### 6. Validar a correção (se o usuário pedir para implementar)

Após aplicar a correção:

```bash
# Build
dotnet build <projeto>.csproj

# Testes unitários
dotnet test <projeto>.csproj --filter Category=Unit

# Testes de integração (se aplicável)
dotnet test <projeto>.csproj --filter Category=Integration

# Lint/analysers
dotnet format --verify-no-changes
```

- Se build falhar → corrigir erro de compilação primeiro
- Se teste existente quebrar → verificar se é regressão da correção
- Se tudo passar → correção validada

## Checklist de boas práticas

```
☐ Causa raiz identificada com arquivo:linha (não "provavelmente é...")
☐ Impacto analisado: escopo do bug e pontos afetados
☐ Melhor prática E fix mínimo apresentados com trade-offs
☐ Plano arquivo por arquivo com passos concretos
☐ Testes de validação recomendados
☐ Build + testes rodados após correção (se implementada)
☐ Não há suposições não verificadas no plano
☐ Edge cases cobertos pela melhor prática
```

## Formato de saída completo

```markdown
## Plano de Correção

**Bug:** [título/descrição]
**Causa raiz:** `Arquivo.cs:42` — [descrição concreta]
**Severidade:** bloqueante / alto / médio / baixo

### Correção recomendada

#### 🏆 Melhor prática
- **Esforço:** X horas
- **Descrição:** [plano concreto]

#### ⚡ Fix mínimo
- **Esforço:** Y minutos
- **Descrição:** [plano concreto]

#### Trade-off

| Critério | Melhor prática | Fix mínimo |
|---|---|---|
| Esforço | Xh | Ym |
| Risco | Baixo | Médio |
| Edge cases | Cobertos | Não |
| Dívida técnica | Nenhuma | Média |

### Arquivos afetados

| Arquivo | Linha | Mudança necessária |
|---|---|---|
| `Service.cs` | 42 | Adicionar validação null |
| `ServiceTests.cs` | — | Novo teste para caso nulo |

### Testes de validação
- [ ] Unit test: `Service.Handle_WithNullInput_ThrowsValidationException`
- [ ] Rodar suíte existente: `dotnet test`

### Validação (após implementação)
- [ ] `dotnet build` — OK
- [ ] `dotnet test` — OK
- [ ] `dotnet format --verify-no-changes` — OK
```

## O que NÃO faz

- Não executa debug interativo — isso é `dotnet-debug-breakpoints`.
- Não debuga containers — isso é `dotnet-debug-docker`.
- Não analisa dumps — isso é `dotnet-debug-diagnostics`.
- Não aplica correções sem confirmação explícita do usuário.
- Não inventa causa raiz quando a evidência é insuficiente.
- Não escolhe entre melhor prática e fix mínimo pelo usuário — apresenta os dois.
