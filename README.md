# ai-skills

Skills pessoais para o Claude Code, desenvolvidas e testadas com o [skill-creator](https://github.com/anthropics/skills).

## Skills

- **[powerpuff-review](powerpuff-review/SKILL.md)** — Revisão de código feita por três personas com foco próprio (arquitetura, legibilidade/DX, bugs/segurança), inspiradas nas Meninas Superpoderosas. Rode com um pedido do tipo "usa a revisão das Powerpuff Girls nesse arquivo/PR". Testado com evals em `powerpuff-review/evals/evals.json` — versão com skill teve 93,3% de taxa de acerto contra 69,5% da linha de base sem skill.

## Instalação

Copie a pasta da skill desejada para `~/.claude/skills/<nome-da-skill>/`.
