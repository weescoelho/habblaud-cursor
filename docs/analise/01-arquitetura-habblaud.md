# 01 — Mapa da arquitetura do Habblaud

## 1. Visão geral

O Habblaud é um **observador local, somente leitura**, das sessões de agentes de programação.
Ele não instrumenta o agente: ele lê os artefatos que a própria ferramenta grava em disco,
constrói um modelo de estado e o publica num navegador.

```
Ferramentas (Claude Code, Codex)  ──gravam──▶  Arquivos locais
                                                    │
                                     leitura (polling ~1s + fs.watch)
                                                    ▼
                                          Servidor Node (server/)
                                     ┌───────────────────────────────┐
                                     │ Fontes (AgentSource/SourceSet) │  presença + atividades
                                     │ Modelo do escritório (Office)  │  agentes/salas/feed/avisos
                                     │ Hub SSE (http/sse.ts)          │  snapshot/feed/notice
                                     │ Registros (permissões/mensagens)│  ações de volta à sessão
                                     │ Histórico (timeline/daystats)  │  persistência em dataDir
                                     └───────────────────────────────┘
                                                    │  SSE (GET /api/stream)
                                                    ▼
                                    Navegador (client/): canvas + UI
```

**Stack** (fato, `README.md:123-139`): Node.js 22.12+, TypeScript, **zero dependências de
runtime** no servidor; cliente Vite; interface e comentários em PT-BR.

## 2. Camadas e responsabilidades

| Camada | Pasta | Responsabilidade | Depende de ferramenta? |
| --- | --- | --- | --- |
| Protocolo | `shared/` | Tipos do protocolo, tradução de atividades PT-BR, nomes, aparência, simulador de demo, github, respostas | **Não** (ver `04`) |
| Servidor | `server/` | HTTP+SSE, contas/uso, leitura de sessões/transcripts, modelo do escritório, registros | **Sim** (nas fontes) |
| Cliente | `client/src/` | Arte pixel-art procedural, mundo/simulação, UI, SSE store | **Não** |
| Mods/plugins | `mod/` | Integração *dentro* da ferramenta (usage, statusline, permissões, mensagens) | **Sim** (Claude Code) |
| Scripts | `scripts/` | Build, docker, instaladores (mod, tap, hooks), assets | **Sim** |

## 3. Núcleo do servidor (`server/index.ts`)

O `index.ts` é o "composition root": monta todas as peças e as conecta (linhas 1-198).

- `AccountsService` (`server/accounts/service.ts`) — contas detectadas e uso.
- `Office` (`server/model/office.ts`) — modelo central: agentes, salas, feed, avisos.
- `SourceSet` — conjunto de `AgentSource` (Claude e Codex).
- `Hub` (`server/http/sse.ts`) — broadcast SSE; `current()` dá o snapshot.
- `HistorySet` — histórico do terminal por provedor.
- `PermissionRegistry` / `MessageRegistry` — ações de volta à sessão.
- `DayStatsService` / `TimelineRecorder` — persistência (`dataDir/stats`, `dataDir/timeline`).
- `UpdateChecker` — única chamada externa (GitHub).

O relógio `office.tick()` roda a cada 250 ms (`server/index.ts:167-173`).

## 4. As três costuras de multi-provedor

### 4.1 `Provider` (protocolo)

```ts
export type Provider = 'claude' | 'codex';   // shared/types.ts:17
```

O campo `provider` é **opcional** em todo o protocolo; **ausente = `claude`**. Regra de evolução
declarada: mudanças em `types.ts` devem ser **aditivas** (`shared/types.ts:1-5`). → Adicionar
`'cursor'` é uma mudança aditiva.

### 4.2 `AgentSource` / `SourceSet` (presença ao vivo)

`server/sources/source.ts:21-38` define o contrato:

```ts
interface AgentSource {
  readonly provider: Provider;
  start(): void | Promise<void>;         // boot + acompanhamento
  stop(): void;
  sources(): SourceInfo[];               // uma entrada por conta (meta.sources, /api/health)
  transcriptPathOf(agentId): string|undefined;
  terminalParser?(agentId): TerminalParser|undefined; // ausente = parser do Claude
}
```

`SourceSet` (linhas 41-112) trata todas as fontes como uma só: `sources()`,
`transcriptPathOf()` (primeira que conhece) e `parserFor()` (parser da fonte dona do transcript).
O boot usa `office.beginBoot()`/`endBoot()` em par para não emitir avisos fora de ordem
(linhas 11-14).

### 4.3 `HistoryProvider` / `HistorySet` (histórico)

`server/sources/source.ts:138-183`. Cada provedor lista sessões recentes e resolve
`(conta, sessionId)` para um transcript + parser. `HistorySet.list()` tolera a falha de um
provedor sem derrubar os outros.

### 4.4 Como o Codex entrou (precedente direto)

1. Fonte dedicada: `server/sources/codex/` (`source.ts`, `files.ts`, `rollout.ts`, `terminal.ts`,
   `accounts.ts`, `history.ts`, `live.ts`).
2. Registro em `server/index.ts:96-98`: `const codex = codexDirs.length ? new CodexSource(...) : undefined; agents.add(codex)`.
3. Rota de ingest: `POST /api/codex/events` (`server/codex/http.ts`) alimenta `CodexLive.applyHookEvent`.
4. Hook no lado da ferramenta: `mod/habblaud-codex/hook.mjs`, instalado por `scripts/codex-install.ts`.
5. Histórico: `new CodexHistory(...)` adicionado ao `HistorySet` (`server/index.ts:117`).
6. Trava de recurso: `HABBLAUD_CODEX=0` (`server/config.ts:51-52,171`).

## 5. Conceitos centrais do modelo

### 5.1 Status do agente

`AgentStatus = 'working' | 'waiting' | 'idle' | 'shell' | 'done' | 'offline'`
(`shared/types.ts:30`). Traduz o comportamento do personagem no canvas:

| Status | Significado |
| --- | --- |
| `working` | Processando um turno (digitando na mesa) |
| `waiting` | Precisa do usuário (permissão, pergunta); mão levantada |
| `idle` | Terminou o turno; circula pelo escritório |
| `shell` | Ocioso, mas com shell(s) em segundo plano rodando |
| `done` | Subagente concluiu e entrega o resultado |
| `offline` | Sessão encerrada |

### 5.2 Atividade

`Activity` (`shared/types.ts:73-91`) com `kind` tipado (`ActivityKind`, linhas 48-71: `read`,
`edit`, `run`, `test`, `git`, `web`, `plan`, `delegate`, `ask`, `mcp`, `skill`…). A tradução de
ferramenta→atividade é `shared/activity.ts` (código puro; servidor e demo usam o mesmo).

### 5.3 Registros de ação (sessão → escritório → sessão)

- `PermissionRegistry` (`server/permissions/registry.ts`): pedido pendente, long-poll, decisão
  (`allow`/`deny`/`terminal`/`answer`), descarte de órfãos.
- `MessageRegistry` (`server/messages/registry.ts`): fila de mensagens, presença (`reachable`),
  entrega ao Claude por plugin e ao Codex por `codex queue`/bridge.

### 5.4 Persistência e recursos opcionais

- `HABBLAUD_DATA_DIR` (padrão `~/.habblaud`): `names.json`, `rooms.json`, `updates.json`,
  `stats/`, `timeline/`.
- Travas de recurso: `HABBLAUD_TERMINAL`, `HABBLAUD_MENSAGENS`, `HABBLAUD_CODEX`,
  `HABBLAUD_TIMELINE` (`server/config.ts`; `README.md:595-616`).
- Terminal (e tudo que *age* sobre a sessão) só liga com **bind local** — é a trava de
  privacidade (`server/config.ts:113-144`).

## 6. Ciclo de vida do boot (fato)

```
main()
 ├─ loadConfig() / migração de estado legado
 ├─ NameStore, RoomAliases, UpdateChecker, AccountsService
 ├─ Office(...)  ── referências tardias ao SourceSet/Permission/Message
 ├─ SourceSet([claude, codex?])
 ├─ Hub(office)
 ├─ DayStats, TimelineRecorder, TerminalStreams, HistorySet
 ├─ PermissionRegistry, MessageRegistry
 ├─ agents.start()      # boot síncrono/assíncrono; beginBoot/endBoot
 ├─ accounts.start(); hub.start(); ...; stats.start(); updates.start()
 ├─ setInterval(office.tick, 250ms)
 └─ http server (guard → api → vite/static)
```

Shutdown limpa timers, watchers e flusha nomes (`server/index.ts:298-319`).

## 7. Onde um provedor "Cursor" se encaixa (resumo)

- Novo valor em `Provider` (aditivo).
- Nova pasta `server/sources/cursor/` com `CursorSource implements AgentSource` (+ `CursorLive`
  para ingest de hooks), `CursorHistory`.
- Nova rota `POST /api/cursor/events` no padrão de `server/codex/http.ts`.
- Registro em `server/index.ts` no bloco do Codex (linhas 91-117).
- Trava de recurso `HABBLAUD_CURSOR` em `server/config.ts`.
- `mod/habblaud-cursor/hook.mjs` + instalador `scripts/cursor-install.ts`.
- **Nada** no cliente, no `Office` estrutural, no `Hub`, no tradutor de atividades ou na arte
  precisa mudar (exceto rótulos/selos de UI, que já são dirigidos por `provider`).

Detalhes em `06-arquitetura-proposta.md`.
