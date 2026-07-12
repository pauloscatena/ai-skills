---
name: qa-unit-tests
description: >
  Cria, roda e analisa testes unitários isolados. Descobre o framework do
  projeto (xUnit, NUnit, Jest, pytest, go test), executa a suíte existente,
  identifica gaps de cobertura e cria testes faltantes seguindo padrão
  Arrange-Act-Assert. Foco em isolamento (mocks/fakes), boundary cases e
  naming convencional. NÃO usar para testes que dependem de banco de dados
  real, API externa ou navegador (usar qa-integration-tests ou
  qa-e2e-tests). NÃO usar para code review estático (usar code-review).
---

# QA Unit Tests — Testes Unitários Isolados

Skill especialista em testes unitários: roda suítes existentes, identifica o que falta, cria testes novos com isolamento total de dependências externas.

## Metodologia

### 1. Entender o escopo

- Leia o que o usuário pediu: testar um módulo específico, criar testes para uma classe nova, ou rodar tudo que existe.
- Identifique o framework de testes do projeto:
  - **.NET/C#**: procure `*.csproj` com referência a xUnit/NUnit/MSTest, arquivos `*Tests.cs`
  - **JavaScript/TypeScript**: procure `jest.config.*`, `vitest.config.*`, `package.json` com script `test`
  - **Python**: procure `pytest.ini`, `pyproject.toml` com `[tool.pytest]`, arquivos `test_*.py`
  - **Go**: procure `*_test.go`
- Identifique a estrutura de pastas de testes (ex: `tests/`, `*.Tests/`, `__tests__/`).

### 2. Rodar suíte existente (baseline)

Antes de criar ou modificar qualquer teste, execute a suíte atual:

```bash
# .NET
dotnet test <projeto>.csproj --verbosity normal

# JavaScript/TypeScript
npm test -- --verbose
# ou
npx jest --verbose

# Python
pytest -v

# Go
go test -v ./...
```

- Anote quantos testes passam, falham ou são ignorados.
- Se algum teste já falhava antes, documente como "pré-existente" — não atribua ao trabalho atual.

### 3. Analisar cobertura (se disponível)

```bash
# .NET
dotnet test <projeto>.csproj --collect:"XPlat Code Coverage"
# ou com coverlet
dotnet test /p:CollectCoverage=true /p:CoverletOutput=./coverage

# JavaScript/TypeScript
npx jest --coverage

# Python
pytest --cov=<modulo> --cov-report=term-missing

# Go
go test -cover ./...
```

- Identifique classes/métodos sem cobertura.
- Priorize para criação de testes: lógica de negócio > validações > utils > mapeamentos.

### 4. Criar testes novos (se solicitado)

Siga o padrão **Arrange-Act-Assert (AAA)**:

```csharp
// .NET (xUnit)
[Fact]
public void CalculateTotal_WithValidItems_ReturnsSum()
{
    // Arrange
    var items = new[] { new Item(10), new Item(20) };

    // Act
    var result = OrderService.CalculateTotal(items);

    // Assert
    result.Should().Be(30);
}
```

```javascript
// JavaScript (Jest)
describe('OrderService', () => {
  it('calculateTotal returns sum of items', () => {
    // Arrange
    const items = [{ price: 10 }, { price: 20 }];

    // Act
    const result = calculateTotal(items);

    // Assert
    expect(result).toBe(30);
  });
});
```

```python
# Python (pytest)
def test_calculate_total_returns_sum_of_items():
    # Arrange
    items = [Item(10), Item(20)]

    # Act
    result = calculate_total(items)

    # Assert
    assert result == 30
```

#### Checklist de boas práticas para unit tests

```
☐ Cada teste é independente (sem estado compartilhado entre testes)
☐ Arrange-Act-Assert é claro e separado por linhas em branco
☐ Nome do teste descreve o cenário: Method_Scenario_ExpectedResult
☐ Não depende de banco de dados, API externa, filesystem real
☐ Dependências externas são mockadas/faked (NSubstitute, Moq, jest.mock)
☐ Boundary cases testados: null, vazio, zero, valor máximo, coleção vazia
☐ Happy path E sad path cobertos
☐ Assertions são específicas (não só "não lança exceção")
☐ Testes são rápidos (< 1s cada, < 10s para a suíte toda)
☐ Não testa implementação interna, testa comportamento/contrato
```

### 5. Regras de isolamento

- **Nunca** faça chamada real a banco de dados, API externa ou serviço de filas em unit tests.
- Use **mocks/fakes** para todas as dependências externas:
  - .NET: NSubstitute, Moq, `Mock<T>`
  - JS/TS: jest.mock, vi.fn, sinon
  - Python: unittest.mock, pytest-mock
- Se o método sob teste depende de `DateTime.Now`, injete um `IClock`/`DateProvider` e faça mock.
- Se o método depende de `IConfiguration`, use `ConfigurationBuilder` com dados inline, não `appsettings.json` real.

## Formato de saída

### Ao rodar testes existentes

```markdown
## Resultado da suíte de testes unitários

| Métrica | Valor |
|---|---|
| Total de testes | N |
| Passaram | N |
| Falharam | N |
| Ignorados/Skipped | N |
| Tempo total | Xs |

### Testes que falharam
- **NomeDoTeste** — `arquivo:linha` — mensagem de erro resumida

### Testes pré-existentes que já falhavam
- (listar se houver)
```

### Ao criar testes novos

```markdown
## Testes criados

| Classe/Método testado | Arquivo de teste | Cenários cobertos |
|---|---|---|
| `OrderService.CalculateTotal` | `OrderServiceTests.cs` | soma válida, lista vazia, item nulo |

### Gaps identificados mas não cobertos (justificativa)
- (listar se houver)
```

## Comandos rápidos

```
Rodar todos os unit tests do projeto
Rodar testes de <classe/metodo específico>
Criar testes unitários para <classe/metodo>
Criar testes para boundary cases de <metodo>
Analisar cobertura de testes do módulo <nome>
```

## O que NÃO faz

- Não cria testes que dependem de banco real, Docker, ou serviços externos — isso é escopo de `qa-integration-tests`.
- Não cria testes de UI/navegador — isso é escopo de `qa-e2e-tests`.
- Não cria testes de performance/carga — isso é escopo de `qa-load-tests`.
- Não corrige bugs encontrados — documenta com precisão para que outra sessão corrija.
- Não assume que código sem teste está correto — reporta como gap de cobertura.
