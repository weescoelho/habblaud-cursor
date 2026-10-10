# 04 — Integrações específicas do Claude Code

Este documento lista **tudo** que o Habblaud assume sobre o Claude Code, onde isso está no
código e o quanto já está abstraído. É a base para a matriz de compatibilidade (`05`).

## 4.1 Inventário de dependências

| # | Mecanismo do Claude Code | Onde o Habblaud usa | Abstraído? |
| --- | --- | --- | --- |
| D1 | Registro de sessões abertas `sessions/<pid>.json` (`pid`, `cwd`, `status`, `version`, `sessionId`) | `server/sources/registry.ts:1-94`; consumido por `watcher.ts` | Não (Claude-only) |
| D2 | Transcript JSONL `projects/<cwd>/<sessionId>.jsonl` (linhas `user`/`assistant`/`tool_use`/`tool_result`/`system`, `usage`, `gitBranch`, `permissionMode`) | `server/sources/transcript.ts`; `watcher.ts` | Não |
| D3 | Subagentes em `<sessão>/subagents/**/agent-*.jsonl` + `.meta.json` + `workflows/<run>/journal.jsonl` | `server/sources/subagents.ts`; `watcher.ts:611-826` | Não |
| D4 | Título customizado `custom-title.json` (`/rename`) | `watcher.ts:894-902` | Não |
| D5 | Hook `PermissionRequest` (`hookSpecificOutput.decision`, `updatedPermissions`, `interrupt`) | `mod/habblaud-permissoes/hooks/permission-hook.mjs:131-155`; `server/permissions/registry.ts` | Parcial (registro é genérico; contrato é Claude) |
| D6 | Ferramenta `AskUserQuestion` (perguntas + `answers` por texto) | `shared/answers.ts`; `shared/activity.ts:16-37`; `permission-hook.mjs:101-124` | Não |
| D7 | Hooks do mod API (`$.session.usage().rateLimits`, `$.prompt.submit({text,asUser})`, `$.session.surfaces()`, `$.ui.status`, `$.clock.*`, `$.http.fetch`) | `mod/habblaud/hooks/register.ts`; `mod/habblaud-mensagens/hooks/register.ts` | Não (só Claude 2.1.287+) |
| D8 | Statusline (`rate_limits`) — tap antigo | `scripts/statusline-tap.mjs` | Não |
| D9 | Contas = config dirs (`~/.claude*`, `CLAUDE_CONFIG_DIR`); alias de shell; `.claude.json` (email/org/uso) | `server/accounts/detect.ts` | Não |
| D10 | Plugins/marketplace (`claude plugin`, `.claude-plugin/marketplace.json`, `settings.json`) | `scripts/mod-install.ts`; `mod/*/.claude-plugin/` | Não |
| D11 | Ferramentas com nomes fixos (`Read`, `Edit`, `Write`, `Bash`, `Grep`, `Glob`, `Task`, `WebFetch`, `WebSearch`, `Monitor`, MCP `mcp__*`) | `shared/activity.ts` | Não (mapa de nomes) |
| D12 | Shells em segundo plano + notificações (`<task-notification>`, `run_in_background`) | `server/sources/shells.ts`; `watcher.ts:70-89,372-384` | Não |
| D13 | Eventos do GitHub no transcript (comandos `gh`/`git`, `gitOperation`) | `server/sources/github.ts`; `shared/github.ts` | Não |
| D14 | Tokens/custo no transcript (`usage`, `costUSD`) | `server/sources/transcript.ts`; `AgentStats` | Não |

## 4.2 Detalhe por mecanismo

### D1/D9 — Presença e contas (a base da observação sem hooks)
O Habblaud **não precisa de nenhum plugin** para saber quem está no escritório: lê
`sessions/<pid>.json`. O `status` (`busy|idle|waiting|shell`) dirige o comportamento do
personagem; a versão `2.1.292+` grava `shell` (`registry.ts:69,80-94`). Fora do Docker, o PID é
checado (`pidAlive`) para não deixar sessões fantasmas.

→ **No Cursor**: não há registro equivalente documentado (lacuna **C2**). Sem ele, a presença
depende de hooks (`sessionStart`/`sessionEnd`) — o que reduz a robustez do modo "somente arquivos".

### D2/D3/D4 — Transcript e subagentes
O parser (`transcript.ts`) é o coração das atividades, tarefas, tokens, branch, modelo, sinais
de shell e GitHub. O `FileTail` faz leitura incremental com `seekTail` para não ler gigabytes.
Subagentes incluem **workflows** (`journal.jsonl`).

→ **No Cursor**: `transcript_path` existe nos hooks, mas o **schema não é documentado**
(lacuna **C1**). É a dependência mais crítica para paridade de funcionalidade.

### D5/D6 — Permissões e perguntas (o "precisa de você")
O `mod/habblaud-permissoes` é um hook `PermissionRequest` **comum** (não é mod): lê o JSON do
stdin, registra em `POST /api/permissions`, espera a decisão em long-poll e imprime
`hookSpecificOutput.decision`. Para `AskUserQuestion`, a resposta volta como
`updatedInput: {...tool_input, answers: {pergunta: rótulos}}`. Vale o que responder primeiro
(terminal ou escritório).

→ **No Cursor**: `PermissionRequest` **não é suportado** (`third-party-hooks`). O "responder pelo
escritório" precisa ser **reconstruído** sobre hooks de decisão (`beforeShellExecution`,
`beforeMCPExecution`, `beforeReadFile`, `preToolUse`, `subagentStart`) que retornam
`permission: allow|deny|ask`. `AskUserQuestion` não tem equivalente direto; perguntas podem ser
aproximadas por `stop`/`followup_message` (lacuna **C4/C7**).

### D7/D8 — Uso do plano
Dois níveis: (a) **mod moderno** (2.1.287+) lê `$.session.usage().rateLimits` e grava
`~/.habblaud/usage/<conta>.json`; (b) **tap de statusline** (versões antigas) intercepta o
`statusLine.command` e lê `rate_limits`. O servidor só lê o arquivo final.

→ **No Cursor**: nenhuma superfície documentada de uso/limites por hook (lacuna **C6/C10**).
O cartão de uso provavelmente fica **vazio** até validar (ou usar a API/dashboard do Cursor).

### D7 — Mod API (o que já não é portável "como está")
`mod/README.md:48-53,107-112` lista o contrato exato dos mods. Ele é um recurso **early access**
do Claude Code ≥ 2.1.287 e **não existe no Cursor**. Qualquer reuso precisa ser reescrito como
*hook* (`hooks.json`), não como *mod*.

### D10 — Distribuição
`claude plugin` + marketplace (`mod/.claude-plugin/marketplace.json`) instala os plugins.
O Cursor tem **"Customize"** (plugins/marketplace) e carrega hooks de terceiros do Claude
(`.claude/settings.json`), mas as mecânicas de publicação são diferentes (lacuna **C9**).

### D11 — Nomes de ferramentas
`shared/activity.ts` traduz nomes como `Bash`, `Read`, `Edit`, `Task`… O Cursor usa nomes
próximos (`Shell`, `Read`, `Write`, `Grep`, `Task`) e a doc de third-party mapeia
`Bash→Shell`, `Edit→Write` (`third-party-hooks`). O tradutor é reutilizável **com uma tabela de
alias** de nomes.

### D12/D13/D14 — Shells, GitHub, tokens/custo
Derivados do transcript do Claude. No Cursor: shells têm hooks dedicados
(`beforeShellExecution`/`afterShellExecution`), o que é **melhor** que no Claude para "esperando
o shell"; GitHub depende do conteúdo do transcript (lacuna **C1**); tokens/custo dependem do
schema do transcript (lacuna **C1**).

## 4.3 O que é portável como está

- `shared/`: aparência, nomes, hash, timeline, daystats, demo, messages (o *tradutor*
  `activity.ts` é reutilizável com alias de nomes).
- `server/`: `model/office.ts`, `http/*`, `history/*`, `updates/`, `log.ts`, `config.ts`
  (com novo flag), `build.ts`.
- `client/`: tudo (o selo por `provider` já existe).
- Padrão de rota de ingest + `CodexLive` (copiar para Cursor).

## 4.4 O que **não** é portável e precisa de trabalho novo (Cursor)

1. Fonte de presença/atividades (`registry` + `transcript` + `subagents`) → **CursorSource**.
2. Permissões/perguntas → **hook de decisão do Cursor** (o `PermissionRequest` não existe).
3. Mod API (`register.ts`) → **hook.mjs** comum (sem `$.*`).
4. Instalador (`claude plugin`) → instalador de `hooks.json` do Cursor.
5. Uso do plano → sem fonte conhecida (validar) → provavelmente degradar para "não exibe".

## 4.5 Conclusão

O Habblaud foi construído **primeiro para o Claude Code** e depois generalizado para o Codex.
A generalização existente cobre *presença + atividades + terminal + histórico*, mas **não**
cobre o *transporte dos hooks* nem a *semântica de permissão/pergunta* — cada provedor os
redefine. Portanto, o esforço do Cursor concentra-se em: (a) descobrir o transcript, (b) mapear
os hooks de decisão do Cursor para o modelo `PermissionRequestInfo` do Habblaud, e (c) um
hook/instalador próprio. Tudo o mais é reuso.
