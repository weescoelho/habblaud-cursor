# 03 — Inventário de módulos relevantes

Coluna **Acoplamento**: `—` = agnóstico de ferramenta; `CC` = específico do Claude Code;
`Codex` = específico do Codex. Onde há acoplamento, o seam de generalização é indicado.

## 3.1 Protocolo compartilhado (`shared/`)

| Arquivo | Responsabilidade | Acoplamento |
| --- | --- | --- |
| `shared/types.ts` | Protocolo servidor↔cliente: `Provider`, `AgentStatus`, `Activity`, `AgentInfo`, `OfficeSnapshot`, `ServerMessage`, terminal, permissões, mensagens, uso | `Provider` (só o enum é acoplável) |
| `shared/activity.ts` | Tradução ferramenta→atividade PT-BR + **mascaramento de segredos** + heurísticas de shell | **CC** (nomes de ferramenta do Claude; ver `04`) |
| `shared/answers.ts` | Perguntas/respostas do `AskUserQuestion` (posições ↔ textos) | CC |
| `shared/github.ts` | Eventos do GitHub derivados do transcript | CC (comandos `gh`/`git`, `gitOperation`) |
| `shared/appearance.ts`, `shared/names.ts`, `shared/hash.ts` | Aparência/nomes determinísticos dos personagens | — |
| `shared/timeline.ts`, `shared/daystats.ts` | Formato da timeline e das estatísticas | — |
| `shared/messages.ts` | Tipos/limites da fila de mensagens | — |
| `shared/demo/*` | Simulador do modo demonstração | — |

## 3.2 Servidor — núcleo

| Arquivo | Responsabilidade | Acoplamento |
| --- | --- | --- |
| `server/index.ts` | Composition root: monta Office, SourceSet, Hub, registros, persistência | ponto de extensão |
| `server/config.ts` | Config por env (`HABBLAUD_*`), travas de recurso, loopback | extensível (`HABBLAUD_CURSOR`) |
| `server/model/office.ts` | Modelo do escritório: agentes/salas/feed/avisos; `beginBoot/endBoot` | — |
| `server/model/rooms.ts`, `names.ts`, `room-aliases.ts` | Salas, nomes, apelidos | — |
| `server/http/app.ts` | Roteador HTTP + guardas | extensível (nova rota) |
| `server/http/sse.ts` | `Hub` (snapshot/feed/notice) | — |
| `server/http/guard.ts` | Trava de Host/Origin/Content-Type (DNS rebinding/CSRF) | — |
| `server/http/static.ts`, `build.ts` | Serve `dist/client`; detecta build desatualizado | — |
| `server/log.ts` | Log com `warnOnce` | — |
| `server/legacy.ts` | Migração CodeTown→Habblaud | — |

## 3.3 Servidor — fontes (o seam multi-provedor)

| Arquivo | Responsabilidade | Acoplamento |
| --- | --- | --- |
| `server/sources/source.ts` | `AgentSource`/`SourceSet`, `HistoryProvider`/`HistorySet` | **seam** |
| `server/sources/watcher.ts` | `ClaudeWatcher` (presença, transcript, subagentes, shells) | **CC** |
| `server/sources/registry.ts` | Registro `sessions/<pid>.json` + status | CC |
| `server/sources/transcript.ts` | Parser JSONL do Claude (linhas, tarefas, uso, sinais) | CC |
| `server/sources/subagents.ts` | Subagentes/workflows do Claude | CC |
| `server/sources/shells.ts` | Rastreador de shells (Bash/Monitor) | **CC** (Bash/Monitor) |
| `server/sources/tail.ts` | `FileTail` — leitura incremental/`seekTail` | — |
| `server/sources/terminal.ts` | `TerminalParser` default (Claude) | CC |
| `server/sources/history.ts` | `SessionHistory` (histórico do Claude) | CC |
| `server/sources/github.ts` / `github-live.ts` | Eventos do GitHub do transcript | CC |
| `server/sources/codex/*` | **Segundo provedor**: `source.ts` (CodexSource), `files.ts`, `rollout.ts`, `terminal.ts`, `accounts.ts`, `history.ts`, `live.ts` | Codex |

## 3.4 Servidor — contas e uso

| Arquivo | Responsabilidade | Acoplamento |
| --- | --- | --- |
| `server/accounts/detect.ts` | Descoberta de config dirs; alias de shell; leitura escopada do `.claude.json` | CC |
| `server/accounts/service.ts` | `AccountsService`: contas, `setProviderAccounts`, uso | genérico (aceita provedores) |
| `server/accounts/usage.ts`, `statusline.ts` | Janelas de uso a partir dos arquivos | CC/Codex |

## 3.5 Servidor — ações de volta à sessão

| Arquivo | Responsabilidade | Acoplamento |
| --- | --- | --- |
| `server/permissions/registry.ts` | Pedidos pendentes, long-poll, decisão | genérico (com `provider`/`account`) |
| `server/permissions/http.ts` | Rotas `/api/permissions/*` | genérico |
| `server/permissions/transcript.ts` | Achar resposta dada no terminal | CC |
| `server/permissions/codex.ts` | Visão de ferramenta do Codex | Codex |
| `server/messages/registry.ts` | Fila + presença + status de entrega | genérico |
| `server/messages/http.ts` | Rotas `/api/messages/*`, `/api/mod/inbox*` | genérico |
| `server/messages/codex.ts` | Entrega via `codex queue`/bridge | Codex |

## 3.6 Servidor — Codex (HTTP) e histórico

| Arquivo | Responsabilidade | Acoplamento |
| --- | --- | --- |
| `server/codex/http.ts` | `POST /api/codex/events` (ingest de hooks) | Codex |
| `server/sources/codex/rollout.ts` | Parser do rollout do Codex | Codex |
| `server/history/timeline.ts`, `daystats.ts` | Persistência (timelapse, Meu dia) | — |
| `server/http/{sessions,terminal,stats,timeline,messages,permissions}.ts` | Rotas | genérico |
| `server/updates/checker.ts` | Versão nova (GitHub) | — |

## 3.7 Mods/plugins (`mod/`) e scripts

| Arquivo | Responsabilidade | Acoplamento |
| --- | --- | --- |
| `mod/habblaud/hooks/register.ts` | Mod Claude: uso do plano + statusline + `/habblaud` | **CC + mod API** |
| `mod/habblaud-permissoes/hooks/permission-hook.mjs` | Hook `PermissionRequest` (permissão + `AskUserQuestion`) | **CC** |
| `mod/habblaud-mensagens/hooks/register.ts` | Injeta mensagens com `$.prompt.submit` | **CC + mod API** |
| `mod/habblaud-codex/hook.mjs` | Hook do Codex (forward de eventos + PermissionRequest) | Codex |
| `scripts/mod-install.ts` | Instala marketplace/plugins via `claude plugin` | CC |
| `scripts/codex-install.ts` | Injeta hooks em `<CODEX_HOME>/hooks.json` + hash de confiança | Codex |
| `scripts/hooks-install.ts`, `statusline-*.ts` | Jeito antigo (settings.json) | CC |
| `scripts/docker-up.ts`, `build-server.mjs`, `release.ts` | Build/docker/release | — |

## 3.8 Cliente (`client/src/`) — relevante para o subproduto

| Área | Arquivos | Papel |
| --- | --- | --- |
| Rede | `net/store.ts`, `net/timeline.ts` | Estado via SSE, ações, replay |
| Provider na UI | `ui/provider.ts` | Selo/rotulação por `provider` (CODEX etc.) |
| Mundo | `world/index.ts`, `world/sim/*`, `world/render/*` | Personagens, salas, shell, GitHub FX |
| Interface | `ui/index.ts`, `ui/model.ts`, `ui/permission.ts`, `ui/composer.ts`, `ui/terminal.ts`, `ui/sidebar.ts`, `ui/usage.ts` | Painéis, permissão, mensagens, terminal, uso |
| Arte | `art/*` | Pixel art procedural (nada específico de ferramenta) |

**Conclusão do inventário**: o acoplamento ao Claude Code está **concentrado em**
`server/sources/{watcher,registry,transcript,subagents,shells,history,github}.ts`,
`server/accounts/detect.ts`, `shared/{activity,answers,github}.ts` e `mod/`. Todo o resto é
agnóstico. Um provedor novo reaproveita `office.ts`, `http/*`, `history/*`, `client/*` e a
maior parte de `shared/`.
