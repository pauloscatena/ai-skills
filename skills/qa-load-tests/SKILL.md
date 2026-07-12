---
name: qa-load-tests
description: >
  Cria, roda e analisa testes de carga e stress usando k6. Define cenários
  (ramp-up, sustain, spike, soak), gera scripts k6, executa contra o alvo,
  e produz relatório com latência p50/p95/p99, throughput e error rate.
  USE PARA: testes de carga, load test, stress test, performance sob
  concorrência, benchmark de API, ver throughput máximo, testar degredação.
  NÃO USE PARA: testes unitários (usar qa-unit-tests), testes de integração
  (usar qa-integration-tests), testes de UI (usar qa-e2e-tests), stress
  test em infraestrutura (usar ferramentas de monitoramento como Grafana/K6
  Cloud).
---

# QA Load Tests — Testes de Carga com k6

Skill especialista em testes de carga e stress. Gera scripts k6, executa cenários variados, e produz relatório detalhado com métricas de performance.

## Metodologia

### 1. Entender o escopo

- Identifique o alvo do teste: API endpoint, web service, GraphQL mutation.
- Pergunte (ou infira do contexto):
  - **Throughput esperado**: quantas requisições/segundo em produção?
  - **Latência aceitável**: p95 máxima aceitável?
  - **Cenário**: carga normal, pico, ou degradção?
- Verifique se k6 está instalado: `k6 version`. Se não estiver, instale:
  - Windows: `choco install k6` ou baixe de `https://k6.io/downloads/`
  - Linux: `sudo snap install k6` ou `brew install k6`

### 2. Definir cenários

| Cenário | Descrição | Uso |
|---|---|---|
| **Load (sustain)** | Carga constante no nível esperado de produção | Validar performance no dia-a-dia |
| **Spike** | Pico súbito de carga (ex: 10x em 30s) | Testar resiliência a picos |
| **Stress** | Carga crescente até quebrar | Encontrar limite de throughput |
| **Soak** | Carga moderada por longo período (30min+) | Detectar memory leaks, degredação |
| **Ramp-up** | Aumento gradual de VUs | Validar auto-scaling |

### 3. Gerar script k6

Siga o padrão abaixo para scripts:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Métricas customizadas
const errorRate = new Rate('errors');
const latencyP95 = new Trend('latency_p95');

export const options = {
  scenarios: {
    // Cenário principal — ajuste conforme necessário
    load_test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 10 },   // ramp-up
        { duration: '2m', target: 10 },    // sustain
        { duration: '30s', target: 0 },    // ramp-down
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% das requisições < 500ms
    http_req_failed: ['rate<0.01'],    // < 1% de erro
    errors: ['rate<0.01'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:5000';

export default function () {
  // === cenário de teste ===

  // 1. Autenticar (se necessário)
  const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: 'test@example.com',
    password: 'testpass',
  }), { headers: { 'Content-Type': 'application/json' } });

  check(loginRes, {
    'login status 200': (r) => r.status === 200,
    'login has token': (r) => JSON.parse(r.body).token !== undefined,
  });

  const token = loginRes.status === 200
    ? JSON.parse(loginRes.body).token
    : '';

  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  };

  // 2. Exercitar endpoint principal
  const res = http.get(`${BASE_URL}/api/training/plans`, { headers });

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'has data': (r) => JSON.parse(r.body).length > 0,
  });

  errorRate.add(res.status !== 200);
  latencyP95.add(res.timings.duration);

  sleep(1); // think time entre requests
}

export function handleSummary(data) {
  return {
    'stdout': JSON.stringify(data, null, 2),
    'load-test-report.json': JSON.stringify(data, null, 2),
  };
}
```

### 4. Executar o teste

```bash
# Cenário de carga (load)
k6 run --out json=results.json -e BASE_URL=http://localhost:5000 load-test.js

# Cenário de stress
k6 run --out json=results.json -e BASE_URL=http://localhost:5000 stress-test.js

# Com mais VUs do que o default
k6 run --vus 50 --duration 5m script.js
```

### 5. Analisar resultados

Após a execução, analise os dados do JSON de saída:

| Métrica | O que verificar | Threshold aceitável |
|---|---|---|
| `http_req_duration` (p50) | Latência mediana | < 200ms para APIs típicas |
| `http_req_duration` (p95) | Latência para 95% das requisições | < 500ms |
| `http_req_duration` (p99) | Latência para 99% das requisições | < 1000ms |
| `http_reqs` | Throughput (req/s) | Abaixo da capacidade do servidor |
| `http_req_failed` | Taxa de erro | < 1% |
| `iterations` | Total de iterações completadas | Deve ser alto (cenário completo executado) |

#### Sinais de problema

| Sinal | Significado | Provável causa |
|---|---|---|
| p95 sobe linearmente com VUs | Throughband do servidor | CPU, I/O, ou pool de conexões limitado |
| Error rate > 5% acima de X VUs | Limite de concorrência | Thread pool, connection pool, file descriptors |
| p99 muito acima do p95 | Outliers de latência | GC pause, lock contention, cold start |
| Throughput cai com mais VUs | Contention/degradação | Lock, pool exhaustion, resource starvation |

## Formato de saída

```markdown
## Relatório de Teste de Carga

**Data:** YYYY-MM-DD HH:MM
**Alvo:** URL testada
**Cenário:** [Load/Stress/Spike/Soak]
**Duração:** Xm
**VUs máximo:** N

### Métricas

| Métrica | Resultado | Threshold | Status |
|---|---|---|---|
| Latência p50 | Xms | < 200ms | ✅/❌ |
| Latência p95 | Xms | < 500ms | ✅/❌ |
| Latência p99 | Xms | < 1000ms | ✅/❌ |
| Throughput | X req/s | — | — |
| Error rate | X% | < 1% | ✅/❌ |
| Total de requests | N | — | — |

### Limites identificados

- **VUs máximos sem degradação**: N
- **Throughput máximo**: X req/s
- **Ponto de quebra**: N VUs (se aplicável)

### Recomendações

- [lista de otimizações com base nos resultados]
```

## Checklist de boas práticas

```
☐ Dados de teste realistas (não fixtures pequenos demais)
☐ Think time entre requests (simula comportamento humano)
☐ Autenticação configurada (token refresh se necessário)
☐ Thresholds definidos antes do teste (não ajustados pós-teste)
☐ Baseline capturada antes de mudanças
☐ Cenário de carga reflete volume real de produção
☐ Teste rodou em ambiente isolado (não partilhado com dev)
☐ Resultados salvos em JSON para comparação futura
```

## Comandos rápidos

```
Criar script de load test para <endpoint/API>
Rodar load test contra <URL> com <N> VUs por <duração>
Rodar stress test para encontrar o limite de <API>
Comparar resultados do teste atual com baseline anterior
```

## O que NÃO faz

- Não testa UI — isso é escopo de `qa-e2e-tests`.
- Não testa lógica isolada — isso é escopo de `qa-unit-tests`.
- Não testa integração entre módulos — isso é escopo de `qa-integration-tests`.
- Não configura infraestrutura de monitoramento (Grafana, InfluxDB) — foca no teste e no relatório.
- Não modifica o código da aplicação para torná-la mais rápida — identifica gargalos e reporta.
