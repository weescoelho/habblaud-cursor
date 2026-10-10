# 05 — Matriz de compatibilidade com o Cursor

Legenda de veredito:
- ✅ **Direto** — o Cursor tem superfície equivalente e o Habblaud já abstrai.
- 🟡 **Adaptável** — existe superfície, mas exige código/mapeamento novo.
- 🔴 **Brecha** — não há superfície conhecida; risco alto; exige validação/produto novo.
- ⚪ **N/A** — não se aplica.

Legenda de evidência: **F** fato (doc oficial/código), **H** hipótese, **?** a validar.

## 5.1 Matriz de capacidades

| Capacidade do Habblaud | Como funciona no Claude Code (F) | Superfície no Cursor | Veredito | Evidência / lacuna |
| --- | --- | --- | --- | --- |
| Presença de sessão aberta | `sessions/<pid>.json` (registro local) | Hooks `sessionStart`/`sessionEnd`; **sem** registro local conhecido | 🟡/🔴 | F: `cursor.com/docs/hooks`; **?** C2 |
| Atividades ao vivo (o que faz agora) | Transcript JSONL + polling | Hooks `preToolUse`/`postToolUse`/`afterFileEdit`/`afterShellExecution`; `transcript_path` | 🟡 | F: hooks; **?** C1 (schema do transcript) |
| Subagentes | Arquivos em `subagents/` + workflows | `subagentStart`/`subagentStop` (com `subagent_id`, `subagent_type`, `task`) | 🟡→✅ | F: `cursor.com/docs/hooks` |
| Status `working` | `busy` no registro / turno aberto | `beforeSubmitPrompt`/`preToolUse`→working; `stop`→idle | 🟡 | F: hooks |
| Status `waiting` ("precisa de você") | Registro `waiting` + hook `PermissionRequest` | `before*Execution` com `permission: ask`; sem `Notification`/`PermissionRequest` | 🔴 | F: `third-party-hooks` (**PermissionRequest: No**) |
| Status `shell` | Registro `shell` + notificações | `beforeShellExecution`/`afterShellExecution` | 🟡→✅ (melhor) | F: hooks |
| Status `idle` | Registro `idle`/fim de turno | `stop`/`afterAgentResponse` | ✅ | F: hooks |
| Status `done` (subagente) | Fim do subagente | `subagentStop` | ✅ | F: hooks |
| Responder permissão (allow/deny) | Hook `PermissionRequest` + long-poll + `hookSpecificOutput.decision` | Hook de decisão (`beforeShellExecution`/`beforeMCPExecution`/`beforeReadFile`/`preToolUse`) com `permission: allow\|deny\|ask`, **long-poll possível** | 🟡/🔴 | F: hooks expõem decisão; **?** C4 (bloquear esperando/timeout) |
| "Sempre permitir" (regras) | `permission_suggestions` + `updatedPermissions` | Não há `updatedPermissions` documentado; `after*` podem editar config? | 🔴 | F: ausente na doc de hooks Cursor; **?** C4 |
| Perguntas (`AskUserQuestion`) | Ferramenta + `answers` | Sem ferramenta/hook equivalente; `stop`.`followup_message` só continua | 🔴 | F/H; **?** C7 |
| Mandar mensagens na sessão | `$.prompt.submit({text,asUser:true})` (mod) | Sem API pública de injeção; `followup_message` só em `stop`/`subagentStop` | 🔴/🟡 | F: hooks; **?** C7 |
| Terminal (reconstrução da conversa) | Parser do transcript JSONL | Depende do `transcript_path` (existente, schema desconhecido) | 🟡/🔴 | F: `transcript_path` na doc; **?** C1 |
| Histórico de sessões (7 dias) | Varredura de `projects/` | Depende do transcript / armazenamento do Cursor | 🟡/🔴 | **?** C1/C8 |
| Tarefas (checklist) | Do transcript | Não há hook de tarefas; provavelmente do transcript | 🟡/🔴 | **?** C1 |
| Tokens / custo | Do transcript | Depende do transcript | 🟡/🔴 | **?** C1 |
| Uso do plano (5h/semanal) | Mod/statusline `rate_limits` | Sem fonte local documentada | 🔴 | **?** C6/C10 |
| Múltiplas contas | Config dirs `~/.claude*` | Cursor é essencialmente 1 usuário/conta (`user_email`) | ⚪/🟡 | F: `user_email` no hook |
| Contas/uso no topo | `AccountsService` | Sem equivalente; pode omitir o cartão | ⚪ | — |
| Eventos do GitHub | Do transcript | Depende do transcript (`gh`/`git`) | 🟡/🔴 | **?** C1/C11 |
| Editor de personagem / sala | `PUT /api/agents/:id/character` (local) | Reusa o servidor; independe da ferramenta | ✅ | — |
| Timeline / "Meu dia" | Sobre o `Office` | Reusa o servidor | ✅ | — |
| Selo de provedor na UI | `Provider` | Adicionar `'cursor'` | ✅ | — |
| Trava local (terminal/ações) | `HABBLAUD_TERMINAL` etc. | Reusa | ✅ | — |

## 5.2 Mapa de hooks: Claude Code → Cursor (fato)

O Cursor carrega hooks do Claude de `.claude/settings.json` e `~/.claude/settings.json`
(requer "Include Third-Party Plugins, Skills, and Other Configs", ligado por padrão) e mapeia
nomes automaticamente:

| Claude Code | Cursor | Suportado |
| --- | --- | --- |
| `PreToolUse` | `preToolUse` | Sim |
| `PostToolUse` | `postToolUse` | Sim |
| `UserPromptSubmit` | `beforeSubmitPrompt` | Sim |
| `Stop` | `stop` | Sim |
| `SubagentStop` | `subagentStop` | Sim (o `subagentStart` é só nativo Cursor) |
| `SessionStart` | `sessionStart` | Sim |
| `SessionEnd` | `sessionEnd` | Sim |
| `PreCompact` | `preCompact` | Sim |
| `Notification` | — | **Não** |
| `PermissionRequest` | — | **Não** |

> Consequência direta: o **hook `habblaud-permissoes` não funciona no Cursor**. "Precisa de você"
> e "responder pelo escritório" precisam de um **hook próprio do Cursor**.

## 5.3 Superfícies do Cursor relevantes (fato)

- **Entrada comum dos hooks**: `conversation_id`, `generation_id`, `model`/`model_id`/`model_params`,
  `hook_event_name`, `cursor_version`, `workspace_roots` (= raiz do projeto → **sala**),
  `user_email`, `transcript_path` (`null` se transcripts desabilitados).
- **Saída/permissão**: `permission: allow|deny|ask`; `user_message`; `agent_message`;
  formatos **flat** ou **aninhado** (`hookSpecificOutput`) — o aninhado do Claude é aceito.
- **Loop**: `stop`/`subagentStop` com `followup_message` ou `decision: block` (+`reason`)
  → análogo de "continuar" / injetar texto.
- **Config**: `~/.cursor/hooks.json`, `<projeto>/.cursor/hooks.json`, enterprise
  `/etc/cursor/hooks.json`; merge Enterprise→Team→Project→User→Claude project→Claude user.
- **Matchers**: regex por tipo de ferramenta (`Shell`, `Read`, `Write`, `Grep`, `Task`,
  `MCP:<tool>`), por tipo de subagente, por comando de shell.
- **CLI headless** (`cursor-agent -p --output-format stream-json`): NDJSON com
  `system`/`user`/`assistant`/`tool_call`/`result` e `session_id` — **fonte alternativa** para
  atividades no modo CLI, mas **não** um arquivo persistente documentado.

## 5.4 Riscos e mitigações

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Transcript do Cursor com schema desconhecido | Alta | Alto | PoC **P0**: capturar `transcript_path`, descobrir formato, mapear p/ `transcript.ts` |
| CLI do Cursor dispara poucos/nenhum hook | Média-Alta | Médio | Focar IDE primeiro; CLI via `--output-format stream-json` (P4) |
| Não dá para bloquear esperando no hook de decisão | Média | Alto | PoC **P2**; se não, degradar para "terminal decide" (só observar) |
| Sem injetar mensagem na sessão | Alta | Médio | Não oferecer "Mandar mensagens" no Cursor (v1); ou só via `stop.followup_message` |
| Sem dados de uso do plano | Alta | Baixo | Ocultar cartão de uso para agentes Cursor |
| `transcript_path` null / desabilitado | Média | Alto | Detectar e mostrar só presença/status via hooks |

## 5.5 O que **não** precisa ser investigado (já resolvido)

- O Habblaud **já** carrega hooks do Claude no Cursor (compatibilidade de terceiros) — podemos
  instalar hooks em `.claude/settings.json` **ou** `.cursor/hooks.json`. Preferir o formato
  nativo do Cursor para usar `subagentStart`, `afterAgentResponse/Thought` e `loop_limit`.
- O Habblaud **já** suporta vários provedores; a UI já exibe selo por `provider`.
