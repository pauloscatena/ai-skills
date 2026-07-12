---
name: dotnet-debug-diagnostics
description: >
  Diagnóstico post-mortem de processos .NET: crash dumps (dotnet-dump),
  traces (dotnet-trace), counters (dotnet-counters), memory dumps, e
  análise de memória. USE PARA: crash dump analysis, memory leak, dotnet-dump,
  dotnet-trace, dotnet-counters, analyze dump, investigation de hang/freeze,
  performance profiling, GC analysis, diagnóstico post-mortem. NÃO USO PARA:
  debug interativo com breakpoints (usar dotnet-debug-breakpoints), debug
  em Docker (usar dotnet-debug-docker), code review (usar code-review).
---

# .NET Debug Diagnostics — Diagnóstico Post-Mortem

Skill especialista em análise de crash dumps, traces e counters de processos .NET. Ferramenta principal para investigar hangs, memory leaks, crashes e problemas de performance onde debug interativo não é viável.

## Metodologia

### 1. Decidir a ferramenta

| Sintoma | Ferramenta | Comando |
|---|---|---|
| Processo travou (hang/freeze) | `dotnet-dump` | `dotnet-dump collect` → analisar threads |
| Memory leak suspeito | `dotnet-dump` + `dotnet-counters` | Coletar dump + monitorar GC |
| Performance degradando | `dotnet-trace` | `dotnet-trace collect` com eventos de CPU |
| Crash com exception não tratada | `dotnet-dump` | Dump automático no crash ou manual |
| GC pressure | `dotnet-counters` | Monitorar `dotnet-counter` em tempo real |
| Threading issues (deadlock) | `dotnet-dump` | Analisar threads e locks |

### 2. Coletar evidência

#### dotnet-dump (crash dumps)

```bash
# Coletar dump de um processo rodando
dotnet-dump collect -p <pid> -o /tmp/coredump

# Coletar dump de processo dentro de container
docker exec <container> dotnet-dump collect -p <pid> -o /tmp/coredump
docker cp <container>:/tmp/coredump ./coredump

# Dump automático em crash (configurar no projeto):
# <PropertyGroup>
#   <DumpArfifactOnCrash>true</DumpArfifactOnCrash>
# </PropertyGroup>
```

#### dotnet-trace (performance profiling)

```bash
# Coletar trace com eventos padrão
dotnet-trace collect -p <pid> --duration 30s -o trace.nettrace

# Coletar com profiling CPU
dotnet-trace collect -p <pid> --profile cpu-usage --duration 30s

# Coletar com GC events
dotnet-trace collect -p <pid> --keywords GC --duration 60s

# Coletar com events customizados
dotnet-trace collect -p <pid> --events Microsoft-DotNETCore-SampleProfiler --duration 10s
```

#### dotnet-counters (monitoramento em tempo real)

```bash
# Listar contadores disponíveis
dotnet-counters list

# Monitorar contadores-chave (CPU, memória, GC)
dotnet-counters monitor -p <pid> \
  --counters System.Runtime,Microsoft.AspNetCore.Hosting

# Métricas-chave para observar:
# - CPU Usage
# - Working Set (memória)
# - GC Heap Size
# - Gen 0/1/2 Collections
# - Gen 0/1/2 Heap Size
# - ThreadPool Thread Count
# - Monitor Lock Contention Count
# - Exception Count
```

### 3. Analisar dump

```bash
# Abrir dump no dotnet-dump (LDB shell)
dotnet-dump analyze /tmp/coredump

# Comandos essenciais no LDB:
# threads                    → listar todas as threads
# thread <id>                → selecionar thread
# clrstack                   → call stack gerenciado
# clrstack -l                → com local variables
# clrstack -p                → com parâmetros
# pe <variable>              → imprimir objeto (expandir)
# dumpobj <address>          → dump de objeto
# dumparray <address>        → dump de array
# gcroot <address>           → quem referencia este objeto
# eeheap -gc                 → heap gerenciado (tamanhos de segmento)
# finalizerqueue             → objetos aguardando finalização
# dumpheap -stat              → estatísticas do heap (tipos + contagem)
# dumpheap -type <TypeName>  → todos os objetos de um tipo
# sos MAddress               → endereços de memória (para memory leak)
```

#### Análise de memory leak

```bash
# 1. Identificar quais tipos mais instâncias têm
dotnet-dump analyze coredump
> dumpheap -stat

# 2. Para o tipo suspeito, listar instâncias
> dumpheap -type MeuTipo

# 3. Pegar GC root de uma instância
> gcroot <address>

# 4. Verificar se finalizer está pendurando
> finalizequeue
```

#### Análise de hang/freeze

```bash
# 1. Ver todas as threads
> threads

# 2. Para cada thread, ver call stack
> thread <id>
> clrstack

# 3. Procurar por:
# - Thread bloqueada em Monitor.Enter (deadlock)
# - Thread em Wait/WaitOne sem timeout
# - Thread bloqueada em I/O assíncrono sem completion

# 4. Se encontrar deadlock, mapear quem segura qual lock
```

### 4. Correlacionar com código

Após identificar o problema no dump/trace, **sempre volte ao código**:

```bash
# Mapear endereço de memória → arquivo:linha (se PDB disponível)
# No dotnet-dump, use clrstack -l para ver file/line

# No trace, usar PerfView ou Visual Studio para abrir .nettrace
# e ver call tree com tempo gasto por método
```

## Checklist de boas práticas

```
☐ Dump coletado no momento certo (durante o hang/crash, não depois)
☐ Trace com duração suficiente para capturar o problema
☐ Contadores monitorados com baseline (antes vs depois)
☐ Dump analisado com: threads, heap stats, gcroot
☐ Memory leak: dumpheap -stat → dumpheap -type → gcroot
☐ Hang: threads → clrstack em cada thread → procurar locks
☐ Trace aberto com PerfView/VS para flame graph
☐ Causa raiz mapeada para arquivo:linha no código
☐ Dump/trace salvos para referência futura
```

## Formato de saída

```markdown
## Diagnóstico Post-Mortem

**Tipo de problema:** [Crash / Hang / Memory Leak / Performance]
**Processo:** nome.exe (PID)
**Ferramenta(s) usada(s):** dotnet-dump / dotnet-trace / dotnet-counters

### Dados coletados
| Fonte | Arquivo | Tamanho |
|---|---|---|
| Crash dump | coredump | X MB |
| Trace | trace.nettrace | X MB |
| Counters | (monitoramento em tempo real) | — |

### Análise

#### Heap Summary (se memory leak)
| Tipo | Instâncias | Tamanho |
|---|---|---|
| MeuTipo | 150.000 | 45 MB |
| string | 890.000 | 23 MB |

#### Thread Analysis (se hang)
| Thread | Estado | Call Stack | Lock? |
|---|---|---|---|
| #1 | Blocked | MethodA → Monitor.Enter | Sim, segurando Lock1 |
| #5 | Blocked | MethodB → Monitor.Enter | Sim, segurando Lock2, esperando Lock1 |

#### Performance (se trace)
| Método | Tempo total | % CPU | Chamadas |
|---|---|---|---|
| ProcessOrder | 45s | 38% | 1.200 |
| SaveToDatabase | 23s | 19% | 1.200 |

### Causa raiz
- **Arquivo:** `Arquivo.cs:42`
- **Problema:** [descrição concreta]
- **Evidência:** [trecho do dump/trace que confirma]
```

## O que NÃO faz

- Não debuga com breakpoints em tempo real — usar `dotnet-debug-breakpoints`.
- Não debuga containers Docker — usar `dotnet-debug-docker`.
- Não aplica correções — documenta e sugere plano (usar `dotnet-debug-fix`).
- Não coleta dumps de processos de produção sem autorização.
- Não deixa dumps ou traces grandes em disco sem avisar ao usuário.
