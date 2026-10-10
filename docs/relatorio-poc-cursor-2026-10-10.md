# Relatório de execução — POC Cursor (P0–P3)

**Data (UTC):** 2026-10-10T17:23:09Z  
**Repositório:** `habblaud` (branch `cursor-poc`)  
**Runbook:** `docs/analise/08-runbook-poc-cursor.md`  
**Ambiente:** Linux 6.8.0-146-generic · Node v22.23.2 · Cursor IDE (captura local)

## Resumo executivo

| POC | Objetivo | Resultado |
| --- | -------- | --------- |
| **P1** | Cobertura de eventos de hook | **Sucesso** — 15/15 eventos candidatos dispararam (`sim`) |
| **P0** | Formato do `transcript_path` | **Sucesso parcial** — `jsonl` legível; tool-calls não detectados pelo inspetor |
| **P2** | Bloqueio e decisão (`allow`/`deny`) | **Sucesso** — stub até ~20 s; integração com Habblaud em 4747 validada |
| **P3** | Agente `cursor` no escritório | **Sucesso** — ingest via API + `cursorEvents: true` |
| **Gate** | `typecheck` + `npm test` | **Sucesso** — 1224 passed, 2 skipped |

Hooks instalados: harness `cursor:poc:install` + hook real `cursor:install` (merge em `~/.cursor/hooks.json`, com backups `hooks.json.habblaud-backup-*`).  
Config de decisão: `~/.habblaud/cursor-hook.json` (`port: 4747`, `decisionTimeoutS: 35`).

---

## P1 — Matriz de cobertura de hooks

Comando: `npm run cursor:poc:report`  
Diretório de captura: `~/.habblaud/cursor-poc/` (`events.jsonl`, `transcripts.txt`, `stdin/`).

| Evento | Dispara? | Contagem | Campos |
| --- | --- | --- | --- |
| preToolUse | sim | 390 | conversation_id, cursor_version, cwd, generation_id, hook_event_name, model, session_id, tool_input, tool_name, tool_use_id, transcript_path, user_email, workspace_roots |
| postToolUse | sim | 373 | conversation_id, cursor_version, cwd, duration, generation_id, hook_event_name, model, session_id, tool_input, tool_name, tool_output, tool_use_id, transcript_path, user_email, workspace_roots |
| afterAgentThought | sim | 312 | conversation_id, cursor_version, duration_ms, generation_id, hook_event_name, model, model_id, model_params, session_id, text, transcript_path, user_email, workspace_roots |
| beforeReadFile | sim | 170 | attachments, content, conversation_id, cursor_version, file_path, generation_id, hook_event_name, model, session_id, transcript_path, user_email, workspace_roots |
| beforeShellExecution | sim | 62 | command, conversation_id, cursor_version, cwd, generation_id, hook_event_name, model, sandbox, session_id, transcript_path, user_email, workspace_roots |
| afterShellExecution | sim | 60 | command, conversation_id, cursor_version, duration, generation_id, hook_event_name, model, output, sandbox, session_id, transcript_path, user_email, workspace_roots |
| afterFileEdit | sim | 29 | conversation_id, cursor_version, edits, file_path, generation_id, hook_event_name, model, session_id, transcript_path, user_email, workspace_roots |
| postToolUseFailure | sim | 13 | conversation_id, cursor_version, cwd, duration, error_message, failure_type, generation_id, hook_event_name, is_interrupt, model, session_id, tool_input, tool_name, tool_use_id, transcript_path, user_email, workspace_roots |
| beforeSubmitPrompt | sim | 10 | attachments, composer_mode, conversation_id, cursor_version, generation_id, hook_event_name, model, model_id, model_params, prompt, session_id, transcript_path, user_email, workspace_roots |
| stop | sim | 9 | cache_read_tokens, cache_write_tokens, conversation_id, cursor_version, generation_id, hook_event_name, input_tokens, loop_count, model, model_id, model_params, output_tokens, session_id, status, transcript_path, user_email, workspace_roots |
| afterAgentResponse | sim | 8 | cache_read_tokens, cache_write_tokens, conversation_id, cursor_version, generation_id, hook_event_name, input_tokens, model, model_id, model_params, output_tokens, session_id, text, transcript_path, user_email, workspace_roots |
| afterMCPExecution | sim | 4 | conversation_id, cursor_version, duration, generation_id, hook_event_name, mcp_server_name, mcp_server_url, model, result_json, session_id, tool_input, tool_name, transcript_path, user_email, workspace_roots |
| beforeMCPExecution | sim | 4 | conversation_id, cursor_version, generation_id, hook_event_name, mcp_server_name, mcp_server_url, model, session_id, tool_input, tool_name, transcript_path, url, user_email, workspace_roots |
| sessionStart | sim | 2 | composer_mode, conversation_id, cursor_version, generation_id, hook_event_name, is_background_agent, model, model_id, model_params, session_id, transcript_path, user_email, workspace_roots |
| sessionEnd | sim | 1 | conversation_id, cursor_version, duration_ms, final_status, generation_id, hook_event_name, is_background_agent, model, model_id, model_params, reason, session_id, transcript_path, user_email, workspace_roots |

**Critério P1:** atendido (ciclo de vida, ferramentas, shell, identificadores de conversa e workspace).

---

## P0 — Transcript (`transcript_path`)

Comando:

```bash
npm run cursor:poc:transcript -- --path "$(tail -1 ~/.habblaud/cursor-poc/transcripts.txt | cut -f2)"
```

**Arquivo inspecionado:**

`/home/jcoelho3/.cursor/projects/home-jcoelho3-habblaud/agent-transcripts/6087c51e-41bd-4801-89c9-84ce7775c102/6087c51e-41bd-4801-89c9-84ce7775c102.jsonl`

**Saída do inspetor:**

```
Formato: jsonl
Linhas: 60
Papéis: user, assistant
Tool-calls: não
Chaves:
  message  53
  role  53
  status  7
  type  7
```

**Critério P0:** arquivo legível com turnos (`user`/`assistant`); derivação de atividades só por transcript ainda limitada (sem campos de tool-call reconhecidos pelo inspetor — lacuna C1).

---

## P2 — Bloqueio do hook de decisão

### Harness local (`cursor:poc:decision`)

| Cenário | Comando | Resultado |
| --- | --- | --- |
| Curto | `--hold 2 --limit 5` | `allow` em ~2,4 s · exit 0 |
| Médio | `--hold 20 --limit 25` | `allow` em ~20,5 s · exit 0 |
| Limite seguro | `--hold 30 --limit 5` | Bloqueio ~30 s · sem `allow` no stdout · exit ≠ 0 (esperado) |

**Nota:** o harness usa `HOME` temporário; o timeout efetivo do hook no stub segue o padrão **25 s** (`decisionTimeoutS` do arquivo real não se aplica ao `HOME` falso). Por isso `--hold 30` no stub não imprime decisão mesmo com `--limit 35`.

### Integração com Habblaud (porta 4747)

Script ad hoc: `hook.mjs` + `sessionStart` + `beforeShellExecution` + decisão via `POST /api/permissions/:id/decision`.

| Fluxo | stdout do hook |
| --- | --- |
| **allow** | `{"permission":"allow"}` |
| **deny** | `{"permission":"deny","user_message":"Recusado pelo usuário no Habblaud: teste poc"}` |

**Critério P2:** bloqueio confiável e formato nativo Cursor validados (stub até ~25 s; ao vivo com servidor até `decisionTimeoutS` configurado).

**Pendência opcional:** validar o mesmo fluxo clicando em **Permitir/Recusar** na UI em http://127.0.0.1:4747 durante um shell no Agent Chat.

---

## P3 — Ingest no escritório

Servidor: `HABBLAUD_CURSOR=1 npm run dev` → http://127.0.0.1:4747

### `/api/health` (coleta)

- `cursorEvents`: **true**
- `permissions`: **true**
- Fonte `cursor`: `sessions: 1`, `ok: true`

### Sequência simulada (`POST /api/cursor/events`)

- `sessionStart` → `beforeShellExecution` → `afterShellExecution` → `preToolUse` → `stop`: todos **`{"ok":true}`**
- Agente no snapshot: `provider: "cursor"`, `roomId: /home/jcoelho3/habblaud`, status `idle` após `stop`

### Casos negativos

| Caso | HTTP / corpo |
| --- | --- |
| `hook_event_name` desconhecido | 200 `{"ok":false}` |
| Body sem `event` válido | **400** |
| `Host: evil.lan` | **403** |

**Critério P3:** atendido (agente aparece, estados e atividades derivados dos hooks; sem regressão observada em Codex na mesma instância).

---

## Gate automatizado

```text
npm run typecheck  → OK
npm test           → 1224 passed | 2 skipped (101 arquivos)
```

Validação de código prévia: `.specs/features/cursor-poc/validation.md` (29/29 ACs em testes unitários).

---

## Evidências em disco

| Artefato | Caminho |
| --- | --- |
| Log de eventos | `~/.habblaud/cursor-poc/events.jsonl` |
| Índice de transcripts | `~/.habblaud/cursor-poc/transcripts.txt` |
| Stdin por evento | `~/.habblaud/cursor-poc/stdin/` |
| Hooks Cursor | `~/.cursor/hooks.json` (+ backups `hooks.json.habblaud-backup-*`) |
| Config hook decisão | `~/.habblaud/cursor-hook.json` |

---

## Limpeza (quando encerrar a POC)

```bash
npm run cursor:poc:uninstall
npm run cursor:uninstall
# opcional: rm -rf ~/.habblaud/cursor-poc/
# opcional: rm ~/.habblaud/cursor-hook.json
```

---

## Próximos passos (fora deste relatório)

- Revisão do relatório e go/no-go para P4–P8 (`.specs/STATE.md`).
- Atualizar `07-poc-priorizados.md` (fora do fork) com achados de P0 (tool-calls no jsonl).
- Captura de tela do escritório durante `working` (entrega visual P3).
