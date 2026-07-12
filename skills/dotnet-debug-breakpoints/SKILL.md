---
name: dotnet-debug-breakpoints
description: >
  Depuração de processos .NET locais com attach, breakpoints e inspeção de
  estado em tempo de execução. Suporta Visual Studio (EnvDTE), VS Code
  (netcoredbg) e execução headless. USE PARA: attach em processo .NET,
  setar breakpoints, step-through (over/into/out), inspecionar variáveis
  e call stack, debugar aplicação localmente. NÃO USE PARA: debug em
  containers Docker (usar dotnet-debug-docker), análise de crash dumps
  (usar dotnet-debug-diagnostics), code review sem bug (usar code-review).
---

# .NET Debug Breakpoints — Depuração Local com Breakpoints

Skill especialista em attach, breakpoints e inspeção de estado em processos .NET locais.

## Metodologia

### 1. Preparar o ambiente

- Localize a solução (`.sln`), projeto (`.csproj`) e ponto de entrada.
- Verifique se a build é **Debug** (com PDBs) — em Release com otimizações, breakpoints podem não bater na linha certa.
- Execute `dotnet build` para garantir binário e símbolos atuais.

### 2. Identificar mecanismo de attach

#### a) Visual Studio aberto (Windows, via EnvDTE)

```powershell
# Conectar ao VS rodando
$dte = [System.Runtime.InteropServices.Marshal]::GetActiveObject("VisualStudio.DTE.17.0")

# Setar breakpoint
$dte.Debugger.Breakpoints.Add("", "C:\caminho\Arquivo.cs", 42)

# Encontrar processo local
$proc = $dte.Debugger.LocalProcesses | Where-Object { $_.Name -like "*NomeDoApp.exe*" }

# Attach
$proc.Attach()
$dte.Debugger.Go()

# Inspeção
$dte.Debugger.CurrentStackFrame
$dte.Debugger.CurrentStackFrame.Locals
$dte.Debugger.CurrentStackFrame.Parent.Threads

# Step-through
$dte.Debugger.StepOver()
$dte.Debugger.StepInto()
$dte.Debugger.StepOut()
```

Se a versão do DTE não for conhecida (`17.0`, `16.0`), tente descobrir com `Marshal::BindToMoniker(...)` ou pergunte a versão do VS.

#### b) VS Code / headless (netcoredbg)

```bash
# Usar netcoredbg em modo interpreter (GDB-MI)
netcoredbg --interpreter=mi --engineLogging

# Comandos GDB-MI:
# -file-exec-and-symbols <path-to-dll>
# -exec-arguments <args>
# -break-insert <file:line>
# -exec-run
# -exec-step-over
# -exec-step-into
# -exec-step-out
# -stack-list-variables --all
# -var-create <name> - * <expression>
# -exec-continue
# -break-delete <id>
```

Alternativa mais simples: editar `.vscode/launch.json` com config `attach`/`launch` e orientar F5.

#### c) Quando não viável interceptar em tempo real

Use diagnóstico post-mortem:
- `dotnet-dump collect` + `dotnet-dump analyze`
- `dotnet-trace`
- Instrumentação temporária (`Console.WriteLine`, `Debugger.Break()`) — **reverter depois**.

### 3. Executar e observar

1. Rode a aplicação com o debugger anexado.
2. Reproduza o cenário do bug.
3. Quando parar no breakpoint, inspecione:
   - **Call stack**: onde estou na cadeia de chamadas
   - **Variáveis locais**: valores reais dos objetos
   - **Watch expressions**: expressões customizadas
   - **Condition breakpoints**: só para quando `x == null` ou `count > 10`
4. Ajuste hipótese e repita até identificar causa raiz.

### 4. Limpeza

- Remova todos os breakpoints e instrumentação temporária.
- Faça detach do processo (não deixe travado).
- Não deixe `Debugger.Break()` ou logs de debug no código.

## Checklist de boas práticas

```
☐ Build em Debug (não Release) antes de attach
☐ Binário e PDBs atualizados (dotnet build)
☐ Breakpoints setados ANTES de attach/run
☐ Conferido se breakpoint é condicional (evitar parada excessiva)
☐ Variáveis inspecionadas com valores reais (não inferidos)
☐ Call stack completo analisado
☐ Causa raiz identificada com arquivo:linha exata
☐ Limpeza: breakpoints removidos, detach feito, instrumentação revertida
```

## Formato de saída

```markdown
## Debug com Breakpoints

**Processo:** nome.exe (PID)
**Ambiente:** Visual Studio 17.0 / VS Code / headless
**Build:** Debug com PDBs

### Configuração
- Breakpoints setados em: arquivo:linha, arquivo:linha
- Condições especiais: (se houver)

### Evidência coletada

| Breakpoint | Variáveis | Call Stack | Observação |
|---|---|---|---|
| Arquivo.cs:42 | `x = null`, `count = 0` | MethodA → MethodB → MethodC | Null reference aqui |

### Causa raiz
- **Arquivo:** `Arquivo.cs:42`
- **Problema:** [descrição concreta]
- **Valor esperado vs obtido:** [especificar]
```

## O que NÃO faz

- Não debuga containers Docker — usar `dotnet-debug-docker`.
- Não analisa crash dumps — usar `dotnet-debug-diagnostics`.
- Não aplica correções — documenta e sugere plano (usar `dotnet-debug-fix`).
- Não dá attach em processos de produção sem confirmação explícita.
