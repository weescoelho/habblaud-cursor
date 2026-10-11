# 06 — Proposta de arquitetura para o subproduto Cursor

> **Recomendação**: **estender o Habblaud** com um terceiro provedor `cursor` (o pedido
> "se for possível estender, é prioridade" é viável e é o melhor caminho). Não reescrever.

## 6.1 Princípios

1. **Aditivo**: `Provider = 'claude' | 'codex' | 'cursor'`; nada existente muda de forma.
2. **Hooks primeiro no Cursor**: sem um registro de sessões local conhecido, o "ao vivo" vem dos
   hooks; os arquivos (transcript) complementam histórico/atividades quando disponíveis.
3. **Degradação graciosa**: cada capacidade (atividades, terminal, permissões, mensagens, uso)
   liga/desliga independentemente conforme o que o Cursor oferecer. Nunca quebrar o Claude/Codex.
4. **Reuso máximo**: `Office`, `Hub`, HTTP, timeline, daystats, cliente, arte e a maior parte de
   `shared/` ficam intactos.
5. **Trava de privacidade igual**: tudo que *age* segue `HABBLAUD_TERMINAL` + bind local.

## 6.2 Arquitetura proposta

```
╔══════════════════════════ Cursor (IDE) ══════════════════════════╗
║  ~/.cursor/hooks.json / <projeto>/.cursor/hooks.json             ║
║      └─▶ mod/habblaud-cursor/hook.mjs  (command hook)            ║
║              │ POST /api/cursor/events  (observação, prazo curto)║
║              │ (permissão) POST /api/permissions + long-poll      ║
║  transcript_path (a validar) ── leitura ──┐                      ║
╚═══════════════════════════════════════════╪══════════════════════╝
                                            │
                        ┌───────────────────┴────────────────────┐
                        ▼                                        ▼
        ┌───────────────────────────┐          ┌──────────────────────────┐
        │ CursorSource (AgentSource)│          │ CursorLive (ingest)      │
        │  - presença por hook       │◀─────────│  applyHookEvent(...)     │
        │  - pump() de transcripts   │          └──────────────────────────┘
        │  - subagentes por hook     │
        └───────────────┬───────────┘
                        ▼
                     Office ──▶ Hub ──▶ SSE ──▶ cliente  (inalterado)
```

### Arquivos novos (espelhando `server/sources/codex/`)

| Arquivo | Papel | Espelha |
| --- | --- | --- |
| `server/sources/cursor/source.ts` | `CursorSource implements AgentSource, CursorLive` — presença, status, atividades, subagentes | `codex/source.ts` |
| `server/sources/cursor/transcript.ts` | Parser do transcript do Cursor → `Activity`/`AgentStats`/`TaskItem` (formato a validar) | `sources/transcript.ts` / `codex/rollout.ts` |
| `server/sources/cursor/terminal.ts` | `createCursorTerminalParser()` | `codex/terminal.ts` |
| `server/sources/cursor/history.ts` | `CursorHistory implements HistoryProvider` (se houver arquivos) | `codex/history.ts` |
| `server/sources/cursor/accounts.ts` | Detecção de "contas"/workspaces do Cursor (provavelmente 1) | `codex/accounts.ts` |
| `server/sources/cursor/live.ts` | Interface `CursorLive` (ingest de hooks) | `codex/live.ts` |
| `server/cursor/http.ts` | `POST /api/cursor/events` | `server/codex/http.ts` |
| `mod/habblaud-cursor/hook.mjs` | Hook do Cursor (forward + decisão de permissão) | `mod/habblaud-codex/hook.mjs` |
| `scripts/cursor-install.ts` | Instala/mergeia `hooks.json` (com backup) | `scripts/codex-install.ts` |

### Alterações em arquivos existentes (pequenas e aditivas)

| Arquivo | Mudança |
| --- | --- |
| `shared/types.ts:17` | `Provider` ganha `'cursor'` |
| `server/config.ts` | Flag `cursor` (`HABBLAUD_CURSOR`, padrão ligado) e dirs do Cursor |
| `server/index.ts:91-117` | Instancia `CursorSource`, adiciona ao `SourceSet` e ao `HistorySet`; passa `cursorLive` ao `createApiHandler` |
| `server/http/app.ts` | Rota `POST /api/cursor/events` (padrão `codexLive`) |
| `shared/activity.ts` | Tabela de **alias de nomes** (`Shell`→Bash etc.) — reaproveita o tradutor |
| `client/src/ui/provider.ts` | Rótulo/selo "CURSOR" (já é dirigido por `provider`) |

## 6.3 Mapeamento de eventos: hook do Cursor → mutação no Office

| Hook do Cursor | `CursorLive` → Office | Status resultante | Evidência |
| --- | --- | --- | --- |
| `sessionStart` | `addMain`/`touch` (workspace_roots = projeto/sala) | `idle`/`working` | F (`workspace_roots`) |
| `beforeSubmitPrompt` | `addActivity(prompt)`, `decide(working)` | `working` | F |
| `preToolUse` / `postToolUse` / `postToolUseFailure` | `addActivity(tool)`; `working` | `working` | F |
| `beforeShellExecution` / `afterShellExecution` | `ShellJob` (start/end), status `shell` | `shell`/`working` | F |
| `beforeMCPExecution` / `afterMCPExecution` | `addActivity(mcp)` | `working` | F |
| `beforeReadFile` / `afterFileEdit` | `addActivity(read/edit)` | `working` | F |
| `subagentStart` / `subagentStop` | `addSub`/`completeSub` (usar `subagent_type`, `task`) | `working`/`done` | F |
| `afterAgentResponse` / `afterAgentThought` | `addActivity(respond/think)` | `working` | F |
| `stop` | `decide(idle)` | `idle` | F |
| `preCompact` | `addActivity(compact)` | — | F |
| `sessionEnd` | `closeMain` | `offline` | F |
| *(decisão)* `before*Execution` retornando `ask` | registrar `PermissionRequestInfo`, `decide(waiting)` | `waiting` | F/H |

Status derivado das **bordas** (início/fim de turno, início/fim de tool), como já faz o
`CodexSource.decide()` (`codex/source.ts:666-684`) — a informação mais nova vence.

## 6.4 Fluxo "responder pelo escritório" no Cursor (novo)

O Cursor não tem `PermissionRequest`, mas os hooks `before*Execution` **decidem**. Propomos o
padrão do Codex:

```
Cursor IDE ──beforeShellExecution──▶ mod/habblaud-cursor/hook.mjs
   │ stdin JSON { conversation_id, command, cwd, ... }
   │ 1) POST /api/cursor/events        (registra; prazo curto)
   │ 2) POST /api/permissions          {provider:'cursor', ...}
   │ 3) ≈ GET /api/permissions/:id/wait (long-poll, até o timeout do hook)
   ▼
stdout { "permission": "allow" | "deny", "user_message": "...", "agent_message": "..." }
```

- **Reuso**: `PermissionRegistry` é agnóstico (já aceita `provider`/`account`);
  `PermissionRequestInfo` cobre `allow`/`deny`/`message`.
- **Limitações vs Claude**: sem `updatedPermissions` e sem `AskUserQuestion`; `interrupt` não
  tem equivalente → v1 suporta **allow/deny + motivo**. O "sempre permitir" é **emulado** por uma
  política local lida pelo hook (ver `.specs/features/cursor-permissoes/`; `AD-002`).
- **Risco**: se o hook não puder esperar (timeout curto), o Habblaud só observa e o terminal
  decide (degradação) — validar em **P2**.

> Observação: o Cursor **também** tem a ferramenta `Task` e `beforeReadFile`; o hook de permissão
> cobre shell, MCP e arquivo. `preToolUse` (com `permission: ask`, não aplicado hoje) é candidato
> futuro para permissões genéricas.

## 6.5 Uso do plano e contas

- **Contas**: o Cursor é essencialmente uma conta (o `user_email` vem nos hooks). Não há
  detecção multi-conta documentada — o subproduto pode exibir um único "Cursor" como conta, sem
  o cartão de uso.
- **Uso/limites**: sem fonte local conhecida (lacuna **C6/C10**). Estratégia: **ocultar** o
  cartão para sessões Cursor na v1; se a API/dashboard do Cursor expuser limites, avaliar depois.

## 6.6 Mensagens (v2)

Sem `$.prompt.submit`, não há injeção confiável de mensagem numa sessão viva. Opções a validar
(**C7**):
1. `stop`/`subagentStop` com `followup_message` → só injeta **ao fim do turno** (análogo parcial).
2. Uma extensão/CLI dedicada (fora do escopo da v1).
3. **v1: não oferecer** "Mandar mensagens" no Cursor; o `composer` fica oculto quando
   `provider === 'cursor'` e a capacidade não estiver confirmada.

## 6.7 Instalação e distribuição

- Hook instalado em `~/.cursor/hooks.json` (global) e/ou `<projeto>/.cursor/hooks.json`
  (versionável), com **merge** e **backup**, no padrão de `scripts/codex-install.ts`.
- **Não** depender de "Include Third-Party Plugins" (é o caminho Claude); usar o formato nativo
  do Cursor para ter `subagentStart`, `afterAgentResponse/Thought` e `loop_limit`.
- Comandos: `npm run cursor:install` / `cursor:status` / `cursor:uninstall`.

## 6.8 Faseamento sugerido

| Fase | Escopo | Depende de |
| --- | --- | --- |
| 0 | PoCs de validação (ver `07`) | — |
| 1 | `Provider:'cursor'` + `CursorSource` com presença/atividades **só por hooks** (sem transcript) | P1/P3 |
| 2 | Permissão por escritório (allow/deny) via hook de decisão | P2 |
| 3 | Transcript: atividades ricas, terminal, tarefas, tokens | P0 |
| 4 | Subagentes, GitHub, histórico | P4/P7 |
| 5 | Uso do plano / mensagens (se viável) | P8/P6 |

> **Fase 2b — Fluxo de permissão sem travar** (planejada, não iniciada): cortar os eventos que pedem
> decisão (padrão: só `beforeShellExecution`/`beforeMCPExecution`; `preToolUse`/`beforeReadFile` viram
> observação), aplicar regras locais `allow`/`deny` no hook e emular o "sempre permitir"
> (`AD-002`). Roteiro/tarefas em `.specs/features/cursor-permissoes/`; operação no runbook `11`.

## 6.9 Decisões em aberto (a fechar antes/junto do faseamento)

- Formato/local real do `transcript_path` e periodicidade (C1/C12).
- Timeout máximo e semântica de bloqueio dos hooks de decisão (C4).
- Existência de "registro de sessões" do Cursor (C2) — define presença sem hook.
- Relação `conversation_id` ↔ `session_id` ↔ transcript ↔ subagente (C8).
- Uso do plano (C6/C10) e injeção de mensagem (C7).
