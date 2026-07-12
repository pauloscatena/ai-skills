---
name: dotnet-debug-docker
description: >
  Depuração de aplicações .NET rodando em containers Docker e docker-compose.
  Descobre serviços, coleta logs, faz attach via vsdbg, e executa
  diagnósticos dentro do container. USE PARA: debug em Docker, docker-compose
  debugging, attach em container, vsdbg, logs de container, diagnóstico de
  microserviços em container. NÃO USE PARA: debug local sem Docker (usar
  dotnet-debug-breakpoints), crash dumps (usar dotnet-debug-diagnostics),
  code review (usar code-review).
---

# .NET Debug Docker — Depuração em Containers Docker

Skill especialista em depurar aplicações .NET dentro de containers Docker: descoberta de serviços, logs, attach via vsdbg, e diagnósticos in-container.

## Metodologia

### 1. Descobrir o ambiente

Mapeie serviço → container → PID antes de qualquer coisa:

```powershell
# Serviços definidos no compose e seus containers
docker compose ps --format json

# Containers rodando (fora de compose também)
docker ps --filter "name=<projeto>"

# PID do processo dotnet dentro do container
docker exec <container> ps aux
```

> **Nota:** `docker compose ps --format json` retorna uma **linha JSON por container**, não um array. Itere linha a linha (`ConvertFrom-Json` por linha no PowerShell).

### 2. Coletar logs (sempre o primeiro passo)

Mesmo antes de tentar attach, colete logs:

```powershell
# Um serviço específico
docker compose logs -f --since 10m <service>

# Todos os serviços (para correlacionar)
docker compose logs -f --since 10m

# Últimas N linhas com timestamp
docker logs --timestamps --tail 200 <container>

# Forçar timestamp do Docker daemon (útil para correlacionar entre serviços)
docker compose logs -t --since 10m
```

**Correlacionar logs entre serviços**: quando o bug envolve múltiplos serviços (API + worker + banco), puxe logs de **todos** e correlacione por timestamp — a causa muitas vezes está upstream. Nem toda imagem emite timestamp próprio (ASP.NET Core não emite por padrão, Postgres/Ollama sim) — use `-t` para forçar.

### 3. Diagnósticos básicos sem attach

```powershell
# Env vars, health, restart count, mounts
docker inspect <container>

# CPU/mem no momento do problema
docker stats --no-stream <container>

# Status do processo dentro do container
docker exec <container> cat /proc/<pid>/status
```

### 4. Attach ao processo dentro do container

#### Pré-requisitos — confirme antes de tentar

```powershell
# Verificar se curl/wget existe (necessário para instalar vsdbg)
docker exec <container> sh -c "command -v curl || command -v wget"

# Verificar capabilities do container
docker inspect <container> --format '{{.HostConfig.CapAdd}} {{.HostConfig.SecurityOpt}}'
```

**Imagens `aspnet` runtime-only** (inclusive .NET 10 sobre Ubuntu Noble) costumam **não ter curl nem wget** — o script `getvsdbgsh` não roda sem uma dessas. E mesmo com vsdbg instalado, attach via `ptrace()` **falha silenciosamente** se o container não tiver `CAP_SYS_PTRACE` (não vem habilitado por padrão).

Se qualquer checagem falhar, **não force instalação ad-hoc** — reporte a limitação e proponha workarounds.

#### VS Code (pipeTransport)

Preferência: instalar `curl` + rodar `getvsdbgsh` num **estágio de build separado no Dockerfile** (`FROM base AS debug`), nunca via `docker exec` na imagem de produção.

```yaml
# docker-compose.override.yml (só dev)
services:
  api:
    cap_add:
      - SYS_PTRACE
    # security_opt: ["seccomp:unconfined"]  # só se SYS_PTRACE não bastar
```

```json
// .vscode/launch.json
{
  "name": ".NET Core Docker Attach",
  "type": "coreclr",
  "request": "attach",
  "processId": "${command:pickProcess}",
  "pipeTransport": {
    "pipeProgram": "docker",
    "pipeArgs": ["exec", "-i", "<container>"],
    "debuggerPath": "/vsdbg/vsdbg",
    "pipeCwd": "${workspaceFolder}"
  },
  "sourceFileMap": { "/src": "${workspaceFolder}" }
}
```

#### Visual Studio (Attach to Process)

`Debug > Attach to Process` com connection type "Docker (Linux Container)" — suporte nativo. Via EnvDTE, tente `$dte.Debugger.Transports` para localizar o transporte Docker. Se falhar, oriente o usuário a fazer manualmente pela UI.

#### Causa comum de "breakpoint não bate"

Verifique `sourceFileMap`/mapeamento de caminho (`/src` no container vs. caminho local).

### 5. Diagnóstico avançado dentro do container

```powershell
# Instalar ferramentas temporariamente
docker exec <container> dotnet tool install --tool-path /tmp/tools dotnet-trace
docker exec <container> dotnet tool install --tool-path /tmp/tools dotnet-dump

# Rodar coleta
docker exec <container> /tmp/tools/dotnet-trace collect -p <pid> --duration 30s
docker exec <container> /tmp/tools/dotnet-dump collect -p <pid>

# Copiar para análise local
docker cp <container>:/tmp/trace.nettrace .
docker cp <container>:/tmp/coredump .

# Se compose expõe /tmp como volume compartilhado, prefira rodar
# ferramentas do host apontando pro socket IPC — evita instalar no container
```

### 6. Limpeza

- Não pare, reinicie ou remova containers de outros serviços (ex: banco com estado) sem confirmar.
- Remova vsdbg ou ferramentas instaladas temporariamente se possível.
- Não deixe containers com `cap_add: SYS_PTRACE` em produção.

## Checklist de boas práticas

```
☐ Serviço → container → PID mapeado antes de qualquer coisa
☐ Logs coletados de TODOS os serviços relevantes (não só o "óbvio")
☐ Timestamps correlacionados entre serviços
☐ Pré-requisitos de attach verificados (curl, capabilities)
☐ sourceFileMap configurado corretamente
☐ Não se assume que ptrace vai funcionar (verificar CAP_SYS_PTRACE)
☐ Containers de outros serviços NÃO foram perturbados
☐ Diagnóstico avançado: dotnet-trace/dump copiados para host
☐ Limpeza: ferramentas temporárias removidas
```

## Formato de saída

```markdown
## Debug Docker

**Serviço:** <nome do serviço>
**Container:** <container ID/nome>
**Imagem:** <nome da imagem>
**PID:** <pid dentro do container>

### Ambiente
- Docker Compose version: X
- capabilities: SYS_PTRACE (sim/não)
- Debug image (vsdbg instalado): sim/não

### Logs relevantes (com timestamp)
```
[2026-07-10T13:20:01Z] [Error] NullReferenceException...
[2026-07-10T13:20:01Z] [Info] Request to /api/plans started...
```

### Diagnóstico do container
| Métrica | Valor |
|---|---|
| CPU | X% |
| Memória | X MB |
| Restart count | N |
| Health | healthy/unhealthy |

### Evidência coletada
[varáveis, call stack, logs — mesmo formato do dotnet-debug-breakpoints]

### Causa raiz
[mesmo formato]
```

## O que NÃO faz

- Não debuga processos locais sem Docker — usar `dotnet-debug-breakpoints`.
- Não analisa crash dumps — usar `dotnet-debug-diagnostics`.
- Não para/reinicia containers de outros serviços sem confirmação.
- Não instala vsdbg em imagem de produção via `docker exec`.
- Não aplica correções — documenta e sugere plano (usar `dotnet-debug-fix`).
