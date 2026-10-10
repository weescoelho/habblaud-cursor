# 02 — Fluxos de dados

Convenções: `─▶` fluxo; `[arquivo]` artefato em disco; `║` fronteira de processo; `~` polling;
`≈` long-poll/espera. Todos os fluxos são **somente leitura**, exceto os marcados como ação
(permissões, mensagens, personagem, sala) — que exigem bind local.

## 1. Fluxo principal do Claude Code (presença + atividades)

```
╔══════════════ Claude Code (por conta) ══════════════╗
║  sessions/<pid>.json   (registro de sessão aberta)   ║
║  projects/<cwd>/<sid>.jsonl  (transcript principal)  ║
║  projects/<cwd>/<sid>/subagents/**/agent-*.jsonl     ║
║  projects/<cwd>/<sid>/custom-title.json              ║
╚═════════════════════╤═══════════════════════════════╝
                      │  leitura incremental
        ┌─────────────┴───────────────┐
        │ ClaudeWatcher (AgentSource) │   ~1s polling + fs.watch (acelerador)
        │  registry.ts  → presença     │
        │  transcript.ts→ atividades   │
        │  subagents.ts → subagentes   │
        │  shells.ts    → shells       │
        └─────────────┬───────────────┘
                      │  addMain / addActivity / addSub / setStatus / setShells / githubEvent
                      ▼
              ┌──────────────┐   commit() (throttle)   ┌──────────┐
              │    Office    │ ──────────────────────▶ │   Hub    │ ──SSE──▶ navegador
              │ (modelo)     │ ◀── snapshot/feed/notice │ (sse.ts) │
              └──────────────┘                         └──────────┘
```

Pontos-chave (fato):
- Presença: `sessions/<pid>.json` (status `busy|idle|waiting|shell`, `pid`, `cwd`, versão)
  — `server/sources/registry.ts:9-94`.
- Atividades: parser do fim do transcript (`FileTail`, `seekTail` ~1 MB) + leitura do começo em
  segundo plano para a linha do tempo longa — `server/sources/watcher.ts:458-490`.
- Subagentes: varredura de `subagents/` (inclusive **workflows** com `journal.jsonl`) —
  `server/sources/watcher.ts:648-826`.
- Shells: Bash/Monitor em segundo plano viram `ShellJob`; terminam por notificação — `shells.ts`.

## 2. Fluxo do Codex (segundo provedor — modelo p/ Cursor)

```
╔════════════════════ Codex (por CODEX_HOME) ════════════════════╗
║  thread-writer-locks/<thread>.lock  (thread carregado)        ║
║  sessions/AAAA/MM/DD/rollout-*.jsonl (conversa)               ║
║  hooks.json ──▶ mod/habblaud-codex/hook.mjs  (ao vivo)        ║
╚════════════════════╤══════════════════════════════════════════╝
                     │                              │
        ┌────────────┴───────────┐      POST /api/codex/events
        │ CodexSource            │◀─────────────────┘  {account, event}
        │  presentThreads()       │   CodexLive.applyHookEvent()
        │  pump()→rollout.ts      │   (adianta status/atividade; PermissionRequest→waiting)
        └────────────┬───────────┘
                     ▼
                  Office ──▶ Hub ──▶ SSE
```

O Codex demonstra o padrão completo de um provedor: **arquivos** para o histórico/atividades +
**hooks** para o "ao vivo" que os arquivos não contêm (ex.: pedido de aprovação) —
`server/sources/codex/source.ts:1-26,788-870`.

## 3. Fluxo "responder pelo escritório" (ação)

```
Claude Code ──PermissionRequest──▶ permission-hook.mjs
   │  (stdin JSON: session_id, tool_name, tool_input,
   │   permission_suggestions)
   │  POST /api/permissions ─────▶ PermissionRegistry
   │                                   │ publica AgentInfo.permission; avisa no feed
   │  ◀── 201 {id} ───────────────────┘
   │  ≈ GET /api/permissions/:id/wait (long-poll, ≤25s, em laço)
   │                                   ▲
   │                                   │ POST /api/permissions/:id/decision
   │                                   │ (navegador: allow/deny/terminal/answer)
   │  ◀── {status:'decided', behavior…}
   ▼
stdout → hookSpecificOutput.decision → Claude Code aplica
```

- Hook: `mod/habblaud-permissoes/hooks/permission-hook.mjs:199-255`.
- Registro: `server/permissions/registry.ts:1-17`.
- Descarte de órfãos (hook morreu, usuário respondeu no terminal): `registry.ts` + leitura do
  transcript (`scanToolCall`).
- Codex usa as **mesmas** rotas com `provider:'codex'`, mas só `allow`/`deny` (+ motivo)
  — `mod/habblaud-codex/hook.mjs:137-155`.

## 4. Fluxo "mandar mensagens" (ação)

```
Navegador ──POST /api/messages──▶ MessageRegistry (fila + status)
                                        ▲
mod/habblaud-mensagens (por sessão)     │ POST /api/mod/inbox {session, account}   (a cada 2s)
   │  ◀── {id, text}  ──────────────────┘   (a pergunta marca presença → AgentInfo.canMessage)
   │  $.prompt.submit({text, asUser:true})        ── entrega na sessão ──▶ Claude Code
   └── POST /api/mod/inbox/ack {results:[{id, ok, error?}]} ──────────────▶ MessageRegistry
```

Codex: entrega por `codex queue` (modo Node) ou pelo auxiliar `codex:bridge` (Docker)
— `server/messages/codex.ts`; `README.md:484-487`.

## 5. Fluxo de uso do plano (5h / semanal)

```
Claude Code ──statusline/$.session.usage().rateLimits──▶ mod/habblaud (register.ts)
                                                            │ grava
                                                            ▼
                                   ~/.habblaud/usage/<conta>.json   (source:'mod')
                                                            │ (ou tap antigo: statusline)
                                                            ▼
        AccountsService ◀── lê usageDir ── UsageWindow{utilization,resetsAt}
              │
              ▼
        AccountInfo.usage  ──▶  Office  ──▶  SSE (cartão de uso)

Codex: rate_limits lidos do próprio rollout (token_count) → AccountUsage (source:'codex')
```

- `shared/types.ts:268-300`; `mod/README.md:15-24`; `server/sources/codex/source.ts:719-786`.

## 6. Fluxo do terminal (SSE dedicado)

```
GET /api/agents/:id/terminal
   │  resolve transcriptPathOf(id) → SourceSet → parserFor(id)
   ▼
TerminalStreams ── lê transcript (FileTail) ──▶ TerminalMessage
   init   = conversa recente (substitui tudo)
   append = entradas novas (user/assistant/thinking/tool/result/system)
```

- Protocolo: `shared/types.ts:436-483`; rota/handler: `server/http/terminal.ts`.
- Histórico de sessões encerradas: `GET /api/sessions/recent` e
  `GET /api/sessions/:conta/:sessionId/terminal` via `HistorySet`.
- Cada provedor fornece seu próprio `terminalParser` (Claude: `sources/terminal.ts`;
  Codex: `sources/codex/terminal.ts`).

## 7. Fluxo de persistência (timeline + Meu dia)

```
Hub.onSnapshot ──▶ TimelineRecorder ──▶ <dataDir>/timeline/<dia>.jsonl (gzip, 7 dias)
office (amostra/1s) ──▶ DayStatsService ──▶ <dataDir>/stats/<dia>.json (30 dias)
```

- `server/index.ts:105-121`; `server/history/timeline.ts`; `server/history/daystats.ts`.
- Não dependem da ferramenta: operam sobre o snapshot do `Office`.

## 8. Fluxo SSE → cliente

```
GET /api/stream  (event: snapshot | feed | notice)
   │
   ▼
client/src/net/store.ts  (estado no navegador; mock/timelapse; ações)
   │
   ▼
client/src/world/ + client/src/ui/  → desenha no <canvas> e na interface
```

- `shared/types.ts:330-393` (OfficeSnapshot/FeedItem/Notice/ServerMessage).
- O store também chama as ações: permissões (`/api/permissions/...`), mensagens
  (`/api/messages`), personagem (`PUT /api/agents/:id/character`), rename de sala.

## 9. Laços de atualização (resumo)

| Fluxo | Cadência | Mecanismo |
| --- | --- | --- |
| Presença/atividades Claude | ~1 s + fs.watch | polling (`watcher.ts`) |
| Presença/atividades Codex | ~1 s + fs.watch + hooks | polling + `POST /api/codex/events` |
| Uso do plano | ~1 s (leitura) / ao vivo (mod) | arquivos em `usageDir` |
| Snapshot SSE | a cada mudança (throttle) | `Hub` |
| Relógio do escritório | 250 ms | `office.tick()` |
| Timeline | throttle por snapshot | `TimelineRecorder` |
| Meu dia | amostra 1 s | `DayStatsService` |
| Long-poll de permissão | ≤25 s por resposta, em laço | `PermissionRegistry` |
| Inbox de mensagens | 2 s (plugin) | `MessageRegistry` |
