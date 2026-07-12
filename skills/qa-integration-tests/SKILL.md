---
name: qa-integration-tests
description: >
  Cria, roda e analisa testes de integração — validação entre módulos,
  contratos de API, interação com banco de dados (via Testcontainers ou
  banco de teste), handlers + repositórios, transações, e filas/queues.
  USE PARA: integration tests, testes com banco real, testes de API
  endpoint-to-endpoint, testes de handlers com repositório, validação de
  transações, testes com Docker/Testcontainers. NÃO USE PARA: testes
  isolados sem dependência externa (usar qa-unit-tests), testes de
  performance/carga (usar qa-load-tests), testes de UI (usar qa-e2e-tests).
---

# QA Integration Tests — Testes de Integração

Skill especialista em testes de integração: valida a interação real entre componentes (API + DB, handler + repositório, módulo A + módulo B) usando bancos de teste, Testcontainers, e contratos de API.

## Metodologia

### 1. Entender o escopo

- Identifique o que está sendo integrado:
  - **API + Banco**: endpoints que leem/escrevem dados reais
  - **Handler + Repositório**: command/query handlers com acesso a DB
  - **Módulo A + Módulo B**: comunicação interna entre bounded contexts
  - **API + Fila**: producers/consumers de messages
  - **API + Serviço externo**: integrações com third-party APIs (mockadas ou reais)
- Identifique o framework de testes e se o projeto já usa Testcontainers:
  - Procure referências a `Testcontainers` nos `.csproj`
  - Procure `docker-compose.yml` ou `TestcontainersBuilder` no código
  - Verifique se Docker está rodando: `docker info`

### 2. Rodar suíte existente

```bash
# .NET — rodar só testes de integração (tipicamente marcados com [Trait("Category", "Integration")])
dotnet test <projeto>.csproj --filter Category=Integration --verbosity normal

# Python
pytest -m integration -v

# JavaScript/TypeScript
npm test -- --testPathPattern=integration --verbose
```

- Anote resultado: quantos passam, quantos falham.
- Se o projeto não tem testes de integração, prossiga para criação.

### 3. Criar testes de integração

#### Padrão para .NET com Testcontainers

```csharp
public class TrainingPlanRepositoryTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer;
    private readonly AppDbContext _context;
    private readonly TrainingPlanRepository _repository;

    public TrainingPlanRepositoryTests()
    {
        _dbContainer = new PostgreSqlBuilder()
            .WithDatabase("test_db")
            .WithUsername("test")
            .WithPassword("test")
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(_dbContainer.GetConnectionString())
            .Options;

        _context = new AppDbContext(options);
        await _context.Database.EnsureCreatedAsync();

        _repository = new TrainingPlanRepository(_context);
    }

    public async Task DisposeAsync()
    {
        await _dbContainer.DisposeAsync();
    }

    [Fact]
    public async Task CreateAsync_WithValidPlan_PersistsToDatabase()
    {
        // Arrange
        var plan = TrainingPlan.Create("Marathon Base", 24);

        // Act
        await _repository.CreateAsync(plan);
        await _context.SaveChangesAsync();

        // Assert
        var saved = await _context.TrainingPlans.FindAsync(plan.Id);
        saved.Should().NotBeNull();
        saved!.Name.Should().Be("Marathon Base");
        saved.Weeks.Should().Be(24);
    }

    [Fact]
    public async Task GetByIdAsync_NonExistentId_ReturnsNull()
    {
        // Act
        var result = await _repository.GetByIdAsync(Guid.NewGuid());

        // Assert
        result.Should().BeNull();
    }
}
```

#### Padrão para testes de API (WebApplicationFactory)

```csharp
public class TrainingPlanApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public TrainingPlanApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Substituir DB real por Testcontainer ou InMemory
                services.AddDbContext<AppDbContext>(options =>
                    options.UseInMemoryDatabase("TestDb"));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task GetPlans_ReturnsOkWithPlans()
    {
        // Act
        var response = await _client.GetAsync("/api/training/plans");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var plans = await response.Content.ReadFromJsonAsync<List<TrainingPlanDto>>();
        plans.Should().NotBeNull();
    }

    [Fact]
    public async Task CreatePlan_WithInvalidData_ReturnsBadRequest()
    {
        // Arrange
        var invalidPlan = new { Name = "", Weeks = -1 };

        // Act
        var response = await _client.PostAsJsonAsync("/api/training/plans", invalidPlan);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
}
```

#### Padrão para Python com pytest

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture(scope="module")
def db_session():
    engine = create_engine("postgresql://test:test@localhost:5432/test_db")
    Session = sessionmaker(bind=engine)
    session = Session()
    yield session
    session.close()

def test_create_training_plan(db_session):
    # Arrange
    plan = TrainingPlan(name="Marathon Base", weeks=24)

    # Act
    db_session.add(plan)
    db_session.commit()

    # Assert
    saved = db_session.query(TrainingPlan).filter_by(id=plan.id).first()
    assert saved is not None
    assert saved.name == "Marathon Base"
    assert saved.weeks == 24
```

### Checklist de boas práticas

```
☐ Cada teste é independente (setup/teardown isolam dados)
☐ Testes usam banco de teste (Testcontainers, InMemory, não o DB real de dev)
☐ Transações são revertidas após cada teste (não sujam o banco)
☐ Dados de teste são criados no Arrange, não dependem de seed prévio
☐ Testes cobrem: happy path, validação, erro, não-encontrado
☐ APIs testam status code E corpo da resposta
☐ Handlers testam: comando válido → efeito colateral correto
☐ Handlers testam: comando inválido → exceção/rejeição apropriada
☐ Testes de integração não testam lógica complexa de negócio (isso é unit)
☐ Timeout configurado para chamadas externas (não hang infinito)
☐ Testes rodam em < 5s cada (se demoram, revisar setup)
☐ Testes de integração são marcados/taggeados para rodar separadamente
```

### 4. Validar transações

Para testes que envolvem transações, verifique:

```csharp
[Fact]
public async Task CreateOrder_WithInvalidItem_RollsBackEntireTransaction()
{
    // Arrange
    using var scope = _context.Database.BeginTransaction();

    // Act + Assert
    await Assert.ThrowsAsync<InvalidOperationException>(async () =>
    {
        await _orderService.CreateOrderAsync(new OrderRequest
        {
            Items = new[] { new OrderItem(productId: Guid.Empty, quantity: -1) }
        });
    });

    // Verificar que nada foi persistido
    var orders = await _context.Orders.ToListAsync();
    orders.Should().BeEmpty();

    await scope.RollbackAsync();
}
```

## Formato de saída

```markdown
## Resultado dos testes de integração

| Métrica | Valor |
|---|---|
| Total de testes | N |
| Passaram | N |
| Falharam | N |
| Tempo total | Xs |

### Testes que falharam
- **NomeDoTeste** — `arquivo:linha` — mensagem de erro + causa raiz provável

### Cobertura de integração

| Componente | Testado? | Observação |
|---|---|---|
| TrainingPlanRepository | ✅ | Create, GetById, Update |
| OrderHandler | ❌ | Não há testes de integração |
| API /training/plans | ✅ | GET, POST, validação |

### Gaps identificados
- [lista de componentes sem teste de integração, com justificativa de prioridade]
```

## Comandos rápidos

```
Rodar testes de integração do projeto
Criar testes de integração para <repositorio/handler/API>
Testar transação de <operacao> (commit/rollback)
Validar contrato da API <endpoint> contra o banco
Configurar Testcontainers para o projeto
```

## O que NÃO faz

- Não cria testes isolados sem dependência externa — isso é escopo de `qa-unit-tests`.
- não testa performance sob carga — isso é escopo de `qa-load-tests`.
- Não navega em UI — isso é escopo de `qa-e2e-tests`.
- Não modifica o schema do banco — cria testes contra o schema existente.
- Não acessa banco de produção — sempre banco de teste ou Testcontainer.
