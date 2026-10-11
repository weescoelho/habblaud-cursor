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
| Atividades ao vivo (o que faz agora) | Transcript JSONL + polling | Hooks `preToolUse`/`postToolUse`/`afterFileEdit`/`afterShellExecution`; `transcript_path` | ✅ (hooks) | F: hooks **confirmados na P1 (15 eventos)**; transcript JSONL legível e **tool-calls mapeados** — `message.content[].tool_use` (C1 ✅). Atividades deriváveis também do transcript |
| Subagentes | Arquivos em `subagents/` + workflows | `subagentStart`/`subagentStop` (com `subagent_id`, `subagent_type`, `task`) | 🟡→✅ | F: `cursor.com/docs/hooks`; **P4**: método corrigido (os hooks de subagente agora são instalados) — re-testar ao vivo |
| Status `working` | `busy` no registro / turno aberto | `beforeSubmitPrompt`/`preToolUse`→working; `stop`→idle | 🟡 | F: hooks |
| Status `waiting` ("precisa de você") | Registro `waiting` + hook `PermissionRequest` | `before*Execution` com `permission: ask`; sem `Notification`/`PermissionRequest` | 🔴 | F: `third-party-hooks` (**PermissionRequest: No**) |
| Status `shell` | Registro `shell` + notificações | `beforeShellExecution`/`afterShellExecution` | 🟡→✅ (melhor) | F: hooks |
| Status `idle` | Registro `idle`/fim de turno | `stop`/`afterAgentResponse` | ✅ | F: hooks |
| Status `done` (subagente) | Fim do subagente | `subagentStop` | ✅ | F: hooks |
| Responder permissão (allow/deny) | Hook `PermissionRequest` + long-poll + `hookSpecificOutput.decision` | Hook de decisão (`beforeShellExecution`/`beforeMCPExecution`/`beforeReadFile`/`preToolUse`) com `permission: allow\|deny\|ask`, **long-poll possível** | 🟡→✅ | F: hooks expõem decisão; **C4 ✅** na P2 (`allow`/`deny` validados, bloqueio ~20 s; config 35 s; ≥30 s por medir) |
| "Sempre permitir" (regras) | `permission_suggestions` + `updatedPermissions` | Não há `updatedPermissions` documentado; `after*` podem editar config? | 🔴 | F: ausente na doc de hooks Cursor; **?** C4 |
| Perguntas (`AskUserQuestion`) | Ferramenta + `answers` | Sem ferramenta/hook equivalente; `stop`.`followup_message` só continua | 🔴 | F/H; **?** C7 |
| Mandar mensagens na sessão | `$.prompt.submit({text,asUser:true})` (mod) | Sem API pública de injeção; `followup_message` só em `stop`/`subagentStop` | 🔴/🟡 | F: hooks; **P6**: o probe devolve `{"followup_message":…}` no `stop` (forma validada) — efeito ao vivo por medir (C7) |
| Terminal (reconstrução da conversa) | Parser do transcript JSONL | `transcript_path` JSONL (schema **mapeado**); hooks dão o ao vivo | 🟡 | **C1 ✅** (2026-10-10): `transcript.ts` lê `{role,message:{content:[…]}}`; terminal completo é trabalho de Fase 1 |
| Histórico de sessões (7 dias) | Varredura de `projects/` | Transcript por conversa em `~/.cursor/projects/*/agent-transcripts/` | 🟡 | **C1 ✅**; **C8** (`conversation_id`↔transcript) por mapear na Fase 1 |
| Tarefas (checklist) | Do transcript | Não há hook de tarefas; viria do transcript | 🟡 | **C1 ✅**; nenhuma ferramenta de tarefas observada na amostra — a confirmar na Fase 1 |
| Tokens / custo | Do transcript | Hooks `stop`/`afterAgentResponse` trazem `input_tokens`/`output_tokens`/`cache_read_tokens`/`cache_write_tokens` | ✅ (hooks) | F: relatório P1 (campos de tokens nos hooks) |
| Uso do plano (5h/semanal) | Mod/statusline `rate_limits` | Sem fonte local documentada | 🔴 (confirmado) | **P8**: `usage` não achou fonte de primeira parte (só config/telemetria e `plans/`) → ocultar o cartão |
| Múltiplas contas | Config dirs `~/.claude*` | Cursor é essencialmente 1 usuário/conta (`user_email`) | ⚪/🟡 | F: `user_email` no hook |
| Contas/uso no topo | `AccountsService` | Sem equivalente; pode omitir o cartão | ⚪ | — |
| Eventos do GitHub | Do transcript | Depende do transcript (`gh`/`git`) | 🟡 | **C1 ✅** (tool-calls no transcript); **C11 ✅** (`tool_name` estável); **P7**: o `history` reusa o parser e detecta `git push`/`gh pr create\|merge` |
| Editor de personagem / sala | `PUT /api/agents/:id/character` (local) | Reusa o servidor; independe da ferramenta | ✅ | — |
| Timeline / "Meu dia" | Sobre o `Office` | Reusa o servidor | ✅ | — |
| Selo de provedor na UI | `Provider` | Adicionar `'cursor'` | ✅ | — |
| Trava local (terminal/ações) | `HABBLAUD_TERMINAL` etc. | Reusa | ✅ | — |

> **POC P0–P3 (2026-10-10)**: hooks da IDE **confirmados** (15 eventos); decisão `allow`/`deny` **validada**
> (bloqueio ~20 s medido; config 35 s; ≥30 s por medir); agente no escritório **OK**. O `transcript_path` é
> **JSONL legível** e os **tool-calls foram mapeados** (**C1 ✅**, no mesmo dia): vêm aninhados em
> `message.content[].tool_use`; o `transcript.ts` lê o formato e o `describeCursorTool` traduz os nomes
> (`Shell`/`StrReplace`/`CallDynamicTool`/`Glob`). Terminal/tarefas/tokens/histórico passam a ser trabalho
> de Fase 1. Relatório: `../relatorio-poc-cursor-2026-10-10.md`; amostra: `../resultado_test_c1.txt`.
>
> **POC P4–P8 (2026-10-11)**: harness entregue (`usage`/`subagents`/`cli`/`followup`/`history` + probe de
> followup) e validado contra o export real no que não exige a IDE: **P8** (uso/limites) → **🔴 confirmado**
> (ocultar o cartão); **P6** (injeção) → forma validada, efeito ao vivo por medir; **P7** (histórico +
> efeitos `git`/`gh`) → **sucesso parcial** (2 sessões reconstruídas); **P4** (subagentes) → método
> corrigido, **re-testar ao vivo**; **P5** (CLI) → instrumentado, NDJSON por coletar. Runbook:
> `10-runbook-poc-cursor-p4-p8.md` (fork); resultados: `07-poc-priorizados.md`.

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
  atividades no modo CLI, mas **não** um arquivo persistente documentado. O harness caracteriza esse
  stream (`cursor:poc:cli`, P5).

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
