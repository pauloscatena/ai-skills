# ai-skills

Skills e agentes pessoais para o Claude Code, desenvolvidos e testados com o [skill-creator](https://github.com/anthropics/skills).

## Skills

- **[powerpuff-review](powerpuff-review/SKILL.md)** — Revisão de código feita por três personas com foco próprio (arquitetura, legibilidade/DX, bugs/segurança), inspiradas nas Meninas Superpoderosas. Rode com um pedido do tipo "usa a revisão das Powerpuff Girls nesse arquivo/PR". Testado com evals em `powerpuff-review/evals/evals.json` — versão com skill teve 93,3% de taxa de acerto contra 69,5% da linha de base sem skill.

### Instalação de skills

Copie a pasta da skill desejada para `~/.claude/skills/<nome-da-skill>/`.

## Agentes

- **[csharp-debugger](agents/csharp-debugger.md)** — Desenvolvedor sênior C#/.NET especializado em depuração real: attach a processo, breakpoints e execução controlada no Visual Studio (via automação COM/EnvDTE) e no VS Code (via `netcoredbg`/`launch.json`), incluindo interceptação de containers Docker/docker-compose (descoberta de serviço→container→PID, logs correlacionados, `docker inspect`/`docker stats`, e attach via `vsdbg` com checagem prévia de viabilidade — `curl`/`wget`/`CAP_SYS_PTRACE`). Ao final sempre entrega um relatório detalhado e um plano de implementação, sem aplicar a correção sozinho. Validado em campo contra um projeto real com stack docker-compose (.NET + Postgres + Ollama).

### Instalação de agentes

Copie o arquivo do agente desejado para `~/.claude/agents/<nome-do-agente>.md` — fica disponível automaticamente em qualquer projeto (escopo de usuário, não por repositório).
