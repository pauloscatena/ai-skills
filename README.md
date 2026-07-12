# ai-skills

Skills e agentes pessoais para o Claude Code, desenvolvidos e testados com o [skill-creator](https://github.com/anthropics/skills).

## Skills

- **[powerpuff-review](skills/powerpuff-review/SKILL.md)** — Revisão de código feita por três personas com foco próprio (arquitetura, legibilidade/DX, bugs/segurança), inspiradas nas Meninas Superpoderosas. Rode com um pedido do tipo "usa a revisão das Powerpuff Girls nesse arquivo/PR". Testado com evals em `powerpuff-review/evals/evals.json` — versão com skill teve 93,3% de taxa de acerto contra 69,5% da linha de base sem skill.
- **[model-validator](skills/model-validator/SKILL.md)** — Valida modelos de dados relacionais de forma profunda — entidades, relacionamentos, normalização, índices, queries, performance e strategy. Gera relatório detalhado de problemas categorizados por severidade e plano de implementação em 4 fases. Cobertura: 7 dimensões (entidades, multi-tenancy, índices, queries, estratégia, segurança, migrations). Uso: "valida o modelo de dados desse projeto" ou `/model-validator full`.
- **[nosql-validator](skills/nosql-validator/SKILL.md)** — Valida modelos de dados NoSQL — document stores, key-value, column-family, graph databases. Analisa padrão de acesso, denormalização, particionamento, consistência, throughput e custo. Suporta: MongoDB, DynamoDB, Cassandra, Redis, Cosmos DB, Firestore, Neo4j. Uso: "valida o modelo MongoDB desse projeto" ou `/nosql-validator full`.

### Instalação de skills

Copie a pasta da skill desejada para `~/.claude/skills/<nome-da-skill>/`.

## Agentes

- **[csharp-debugger](agents/csharp-debugger.md)** — Desenvolvedor sênior C#/.NET especializado em depuração real: attach a processo, breakpoints e execução controlada no Visual Studio (via automação COM/EnvDTE) e no VS Code (via `netcoredbg`/`launch.json`), incluindo interceptação de containers Docker/docker-compose (descoberta de serviço→container→PID, logs correlacionados, `docker inspect`/`docker stats`, e attach via `vsdbg` com checagem prévia de viabilidade — `curl`/`wget`/`CAP_SYS_PTRACE`). Ao final sempre entrega um relatório detalhado e um plano de implementação, sem aplicar a correção sozinho. Validado em campo contra um projeto real com stack docker-compose (.NET + Postgres + Ollama).
- **[qa-tester](agents/qa-tester.md)** — Engenheiro de QA sênior que executa validação de verdade em vez de inferir pelo código: roda a suíte de testes automatizada existente como baseline e, em projetos web, navega de verdade via `claude-in-chrome` (formulários, cliques, navegação) checando console e network a cada fluxo. Cobre o roteiro fornecido pelo usuário mais casos de boas práticas de QA (fronteiras, campo vazio, erro/loading). Não corrige código nem invoca outros agentes — sinaliza no relatório quando um achado pede handoff para o `csharp-debugger` (causa raiz em C#/.NET). Relatório final sempre com escopo testado, resultado da suíte automatizada, achados com severidade/repro/evidência, e casos que passaram. Validado em campo contra um projeto real (login, exploração de painel web, achados reais de UX/consistência de dados).
- **[dba](agents/dba.md)** — DBA agent que valida e otimiza bancos de dados relacionais e NoSQL. Audita modelos, índices, queries, performance, segurança e custo. Utiliza as skills `model-validator` (relacional) e `nosql-validator` (NoSQL) para gerar relatórios detalhados e planos de implementação. Detecta automaticamente o engine do banco (via Docker Compose, configs, packages) e carrega a skill adequada. Uso: "valida o banco desse projeto" ou `/dba full`.

### Instalação de agentes

Copie o arquivo do agente desejado para `~/.claude/agents/<nome-do-agente>.md` — fica disponível automaticamente em qualquer projeto (escopo de usuário, não por repositório).
