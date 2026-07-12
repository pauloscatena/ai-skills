---
name: qa-e2e-tests
description: >
  Executa testes end-to-end, smoke tests e regressão visual em aplicações
  web usando Chrome MCP (claude-in-chrome). Navega na UI real, preenche
  formulários, clica, valida fluxos completos do usuário (login → feature →
  logout), e verifica console/rede a cada step. USE PARA: testes E2E, end
  to end, smoke test, regressão visual, teste completo do fluxo do usuário,
  validação de UI, testes exploratórios web. NÃO USE PARA: testes
  unitários (usar qa-unit-tests), testes de carga (usar qa-load-tests),
  testes de integração backend (usar qa-integration-tests), code review
  estático (usar code-review).
---

# QA E2E Tests — Testes End-to-End, Smoke e Regressão Visual

Skill especialista em testes de UI web: navega na aplicação real via Chrome MCP, executa fluxos completos do usuário, e valida comportamento visual, funcional e de rede.

## Metodologia

### 1. Entender o escopo

- Leia o que o usuário pediu:
  - **Fluxo específico**: "testa o login → dashboard → criar treino"
  - **Smoke test**: "roda um smoke test rápido no site"
  - **Regressão visual**: "compara se a página de login mudou depois do deploy"
  - **Exploratório**: "explora a app e me diz o que encontrar de errado"
- Identifique a URL base da aplicação (production, staging, ou localhost).
- Se a aplicação precisa de autenticação, tenha as credenciais prontas (ou peça ao usuário).

### 2. Preparar Chrome MCP

Se as ferramentas `mcp__claude-in-chrome__*` estiverem deferred, carregue todas de uma vez no início:

```
ToolSearch com select: mcp__claude-in-chrome__*
```

Ferramentas-chave:

| Ferramenta | Uso |
|---|---|
| `navigate` | Abrir URL, navegar para páginas |
| `computer` | Clicar por coordenada (mais confiável que `ref`) |
| `read_page` | Ler estrutura da página (DOM acessível) |
| `get_page_text` | Extrair texto visível da página |
| `form_input` | Preencher campos de formulário |
| `read_console_messages` | Ver erros JS silenciosos |
| `read_network_requests` | Ver chamadas API (4xx/5xx) |
| `javascript_tool` | Inspecionar DOM, estado interno |
| `browser_batch` | Executar sequência de passos em lote |
| `tabs_create_mcp` | Criar nova aba |
| `tabs_close_mcp` | Fechar abas ao final |

### 3. Executar o teste

#### 3.1 Smoke test (rápido, caminho feliz)

Teste básico para verificar se a aplicação está funcionando:

```
1. Navegar para a URL base
2. Verificar que a página carrega (sem erro no console)
3. Verificar que elementos essenciais estão visíveis (header, nav, conteúdo principal)
4. Se tem login: fazer login → verificar que redireciona para dashboard
5. Fechar aba
```

#### 3.2 Teste de fluxo completo (E2E)

```
1. Login (se necessário) → verificar redirecionamento
2. Navegar até a feature alvo
3. Executar as ações do fluxo:
   - Preencher formulários com dados reais (não inválidos)
   - Clicar botões/links
   - Aguardar resposta (loading → resultado)
4. Validar em cada step:
   - Console sem erros JS
   - Network sem 4xx/5xx inesperados
   - Estado da UI correto (texto, visibilidade, dados)
5. Verificar estado final (dados persistidos, mensagem de sucesso)
6. Logout (se aplicável)
7. Fechar abas
```

#### 3.3 Regressão visual

```
1. Capturar screenshot da página antes (baseline)
2. Mudança aplicada (deploy, código novo)
3. Capturar screenshot depois
4. Comparar: elementos deslocados, cores diferentes, texto faltando
5. Usar get_page_text para confirmar se mudanças visuais são intencionais
```

### 4. Validações em cada step

Sempre verifique **três camadas** a cada interação significativa:

| Camada | Como verificar | O que procurar |
|---|---|---|
| **Console** | `read_console_messages` | Erros JS, exceptions, warnings |
| **Rede** | `read_network_requests` | 4xx/5xx, respostas vazias, timeouts |
| **UI** | `read_page` / `get_page_text` | Dados incorretos, botão desabilitado, layout quebrado |

### 5. Regras de navegação

- **Prefira clique por coordenada** (`computer` com `coordinate`) em vez de `ref` em submits críticos — `ref` pode ficar stale entre chamadas.
- **Use `browser_batch`** quando a sequência já for previsível para evitar round-trips.
- **Confirme dados visuais via DOM** (`get_page_text`, `javascript_tool`) antes de reportar defeito visual — screenshot pode dar timeout/renderer travado.
- **Não invente features** — teste o que existe de fato no app.

### 6. Higiene

- Feche abas que você abriu (`tabs_close_mcp`) ao final.
- Se criou dados de teste (reservas, pedidos, registros), diga quais e onde para limpeza.
- Não deixe dados de teste "sujando" ambiente compartilhado sem avisar.

## Checklist de boas práticas

```
☐ Fluxo testado do início ao fim (login → feature → logout)
☐ Console verificado a cada step relevante (sem erros JS)
☐ Network verificada a cada step relevante (sem 4xx/5xx)
☐ Dados de entrada realistas (não só "abc" ou "")
☐ Formulários testados com: dados válidos, campos obrigatórios vazios, formato inválido
☐ Estados de loading/erro verificados (não só o happy path)
☐ Botões/link desabilitados/habilitados no momento certo
☐ Mensagens de sucesso/erro exibidas corretamente
☐ Navegação (voltar, menu, breadcrumbs) funciona
☐ Abas fechadas ao final
☐ Dados de teste documentados para limpeza
```

## Formato de saída

```markdown
## Resultado do teste E2E

**Data:** YYYY-MM-DD HH:MM
**URL:** URL testada
**Tipo:** [Smoke / Fluxo completo / Regressão visual / Exploratório]

### Fluxo testado

| Step | Ação | Console | Rede | UI | Status |
|---|---|---|---|---|---|
| 1 | Navegar para /login | ✅ | ✅ | ✅ | ✅ |
| 2 | Preencher email + senha | ✅ | — | ✅ | ✅ |
| 3 | Clicar "Entrar" | ✅ | ✅ (200) | ✅ | ✅ |
| 4 | Dashboard carrega | ⚠️ 1 warning | ✅ | ✅ | ⚠️ |

### Achados

#### A-001: [Título]
- **Severidade:** bloqueante / alto / médio / baixo / cosmético
- **Step:** onde o problema ocorreu
- **Passos de reprodução:**
  1. Navegar para /login
  2. Preencher email: test@test.com
  3. Clicar "Entrar"
- **Resultado esperado:** Redireciona para /dashboard com dados do usuário
- **Resultado observado:** Tela branca, console erro: `TypeError: Cannot read property 'name' of undefined`
- **Evidência:** trecho do console, network request, `arquivo:linha`
- **Recomendado:** handoff para `csharp-debugger` (causa raiz no backend)

### Console messages capturados
| Tipo | Mensagem | Ocorreu no step |
|---|---|---|
| error | TypeError: ... | Step 4 |
| warning | Deprecated API call | Step 1 |

### Network requests relevantes
| Método | URL | Status | Observação |
|---|---|---|---|
| POST | /api/auth/login | 200 | OK |
| GET | /api/dashboard | 500 | ERRO — retorna erro interno |

### Casos que passaram
- Login com credenciais válidas
- Navegação menu → feature
- Logout

### Dados de teste criados
- Usuário de teste: test@test.com (precisa ser removido)
```

## Comandos rápidos

```
Rodar smoke test no <URL>
Testar fluxo completo: <descrição do fluxo>
Testar formulário de <nome> com dados inválidos
Verificar se a página <URL> carrega sem erros no console
Comparar estado visual da <página> com baseline
Explorar a aplicação e reportar bugs encontrados
```

## O que NÃO faz

- Não cria testes unitários — isso é escopo de `qa-unit-tests`.
- Não cria scripts de carga — isso é escopo de `qa-load-tests`.
- Não testa integração backend isoladamente — isso é escopo de `qa-integration-tests`.
- Não corrige bugs — documenta com precisão suficiente para reprodução.
- Não assume que screenshot confirma defeito visual — valida via DOM/texto primeiro.
- Não navega em produção sem confirmar com o usuário (ambiente seguro para teste).
