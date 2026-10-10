# Análise do Habblaud e proposta de suporte ao Cursor

> Investigação técnica (estática, sobre o código-fonte) do repositório Habblaud
> (`https://github.com/marmottajr/habblaud`, cópia local `./projects/habblaud-main`), com
> o objetivo de projetar/estender o produto para acompanhar agentes do **Cursor**.
>
> Este documento **não implementa** nada. Ele consolida evidências, incertezas e uma
> recomendação. A implementação só deve começar depois de validar os pontos marcados como
> *a validar* (ver `05` e `07`).

## Como ler

| Arquivo | Entregável |
| --- | --- |
| `00-resumo-e-ledger.md` (este) | Resumo executivo + **ledger de evidências** (fato / hipótese / não investigado) |
| `01-arquitetura-habblaud.md` | Mapa da arquitetura do Habblaud |
| `02-fluxos-de-dados.md` | Diagrama dos principais fluxos de dados |
| `03-inventario-modulos.md` | Inventário de módulos com referências a arquivos |
| `04-integracoes-claude-code.md` | Análise das integrações específicas do Claude Code |
| `05-matriz-compatibilidade-cursor.md` | Matriz de compatibilidade com o Cursor |
| `06-arquitetura-proposta.md` | Proposta de arquitetura para o subproduto Cursor |
| `07-poc-priorizados.md` | Lista priorizada de provas de conceito |

## Resumo executivo

O Habblaud é o "escritório virtual" dos agentes de programação: cada projeto vira uma sala e
cada sessão de agente vira um personagem em pixel art que mostra, em tempo quase real, o que
o agente está fazendo. Ele é um serviço Node **sem dependências de runtime**, que só **lê**
arquivos que as ferramentas já gravam localmente (transcripts e registros de sessão), deriva um
modelo de estado (`Office`) e o transmite ao navegador por **SSE**; o cliente desenha tudo em
`<canvas>`.

### O achado central (fato)

O Habblaud **já foi projetado para múltiplos provedores**. Existem três costuras explícitas:

1. `Provider` no protocolo compartilhado — hoje `'claude' | 'codex'` (`shared/types.ts:17`);
2. `AgentSource` / `SourceSet` — uma fonte por ferramenta, ligada em runtime
   (`server/sources/source.ts:21-112`);
3. `HistoryProvider` / `HistorySet` — histórico do terminal por ferramenta
   (`server/sources/source.ts:138-183`).

O **Codex é a prova viva** de que adicionar uma segunda ferramenta é uma extensão, não uma
reescrita: `server/sources/codex/` implementa `CodexSource implements AgentSource, CodexLive`
(`source.ts:170`), recebe eventos por uma rota de ingest de hooks (`POST /api/codex/events`,
`server/codex/http.ts`) e entrega o histórico por `CodexHistory`. O `server/index.ts:91-117`
mostra o ponto exato de acoplamento onde um terceiro provedor se registraria.

### Recomendação (alinhada ao pedido "estender tem prioridade")

**Estender o Habblaud com um terceiro provedor `cursor`, não construir um produto novo.**
A maior parte do valor (modelo do escritório, arte pixel-art, SSE, terminal, timeline, "Meu dia",
tradutor de atividades PT-BR, mascaramento de segredos) é **agnóstica de provedor** e
reaproveitável 100%. O trabalho novo concentra-se em:

- uma fonte `CursorSource` (presença das sessões + parser de transcript + subagentes);
- uma rota de ingest de eventos de hook (`POST /api/cursor/events`), como a do Codex;
- um hook do Cursor (`mod/habblaud-cursor/hook.mjs`) que reencaminha eventos para o servidor;
- a reconstrução de "precisa de você" e de "responder pelo escritório", que no Claude Code usam
  o hook `PermissionRequest` — **que o Cursor NÃO suporta** (ver abaixo).

### Os três maiores riscos (a validar antes de implementar)

1. **O Cursor não tem o hook `PermissionRequest` nem o `Notification`.** A documentação oficial
   de *third-party hooks* lista `PermissionRequest` e `Notification` como **não suportados**
   (`cursor.com/docs/reference/third-party-hooks`). No Cursor, a permissão é expressa nos hooks
   `beforeShellExecution` / `beforeMCPExecution` / `beforeReadFile` / `preToolUse` / `subagentStart`,
   retornando `permission: allow|deny|ask`. Ou seja: o mecanismo "responder permissão pelo
   escritório" precisa ser **reconstruído** sobre esses hooks (um hook que *espera* a decisão do
   escritório e responde `allow`/`deny`), exatamente como o Habblaud já faz para o Codex.
2. **O formato e o local do `transcript_path` do Cursor não são documentados.** Os hooks recebem
   `transcript_path` ("null se transcripts desabilitados"), mas o Cursor não publica o schema
   desse arquivo. Sem ele não há: leitura de atividades, terminal, tarefas, estatísticas nem
   histórico no mesmo nível do Claude Code. Isso é **a primeira coisa a validar** (PoC P0).
3. **O suporte a hooks no CLI do Cursor (`cursor-agent`) é parcial e instável.** Relatos do fórum
   oficial indicam que o CLI dispara só `beforeShellExecution`/`afterShellExecution`, com casos de
   nenhum hook disparando no Linux. O plano deve assumir **IDE primeiro**, CLI como cenário
   secundário a validar.

## Ledger de evidências

Legenda: **F** = fato confirmado no código-fonte ou em documentação oficial; **H** = hipótese
(raciocínio sobre fatos, ainda não executado); **?** = não investigado / precisa de validação em
runtime.

### Habblaud (fonte: leitura do código local)

| # | Afirmação | Tipo | Evidência |
| --- | --- | --- | --- |
| H1 | Servidor Node sem dependências de runtime; TypeScript; cliente Vite/`<canvas>`; dados por SSE | F | `README.md:138-139,623-647`; `server/index.ts`; `package.json` |
| H2 | Uma fonte de agentes por ferramenta via `AgentSource`/`SourceSet` | F | `server/sources/source.ts:21-112` |
| H3 | Histórico do terminal por ferramenta via `HistoryProvider`/`HistorySet` | F | `server/sources/source.ts:138-183` |
| H4 | `Provider = 'claude' \| 'codex'`; ausência do campo = `claude` | F | `shared/types.ts:9-17` |
| H5 | Claude Code: presença via `sessions/<pid>.json`; atividades via `projects/<cwd>/<sessionId>.jsonl`; subagentes em `subagents/` | F | `server/sources/watcher.ts`; `server/sources/registry.ts:1-25`; `server/sources/transcript.ts`; `server/sources/subagents.ts` |
| H6 | Codex: presença via `thread-writer-locks/<thread>.lock` + rollout; eventos por `POST /api/codex/events` | F | `server/sources/codex/source.ts:279-376,788-870`; `server/codex/http.ts` |
| H7 | "Responder pelo escritório" = hook `PermissionRequest` que registra em `POST /api/permissions` e espera em long-poll | F | `mod/habblaud-permissoes/hooks/permission-hook.mjs:199-255`; `server/permissions/registry.ts:1-17` |
| H8 | "Mandar mensagens" = plugin que busca `POST /api/mod/inbox` e injeta com `$.prompt.submit({text, asUser:true})` | F | `mod/README.md:77-101`; `shared/types.ts:599-624` |
| H9 | Uso do plano (5h/semanal) = mod que lê `$.session.usage().rateLimits` e grava `~/.habblaud/usage/<conta>.json`; Codex lê `rate_limits` dos rollouts | F | `mod/README.md:15-24`; `shared/types.ts:268-300`; `server/sources/codex/source.ts:719-786` |
| H10 | Tradução de ferramenta→atividade em PT-BR + mascaramento de segredos é código puro compartilhado | F | `shared/activity.ts:1-90` |
| H11 | Timeline (timelapse) e "Meu dia" persistem no `dataDir`; dependem do `Office`/`Hub`, não da ferramenta | F | `server/index.ts:105-121`; `server/history/timeline.ts`; `server/history/daystats.ts` |
| H12 | Terminal ligado só com bind local; é a trava de terminal/permissões/mensagens | F | `server/config.ts:113-144`; `server/http/app.ts` |
| H13 | Cursor pode carregar hooks do Claude Code, com mapeamento de nomes, **exceto** `Notification` e `PermissionRequest` | F | `cursor.com/docs/reference/third-party-hooks` |
| H14 | Cursor expõe hooks `sessionStart/End`, `pre/postToolUse(±Failure)`, `subagentStart/Stop`, `before/afterShellExecution`, `before/afterMCPExecution`, `beforeReadFile/afterFileEdit`, `beforeSubmitPrompt`, `preCompact`, `stop`, `afterAgentResponse/Thought`, `workspaceOpen` | F | `cursor.com/docs/hooks` |
| H15 | Cursor define permissão por hook (`permission: allow\|deny\|ask`) em `beforeShellExecution`/`beforeMCPExecution`/`beforeReadFile`/`preToolUse`/`subagentStart` | F | `cursor.com/docs/hooks` |
| H16 | Cursor tem `stop`/`subagentStop` com `followup_message`/`decision: block` (análogo de "continuar/injetar") | F | `cursor.com/docs/hooks`; `cursor.com/docs/reference/third-party-hooks` |
| H17 | Hook do Cursor recebe `conversation_id`, `generation_id`, `model`/`model_id`/`model_params`, `hook_event_name`, `cursor_version`, `workspace_roots`, `user_email`, `transcript_path` | F | `cursor.com/docs/hooks` |
| H18 | CLI `cursor-agent` tem `--output-format json\|stream-json\|text`; NDJSON com `system`/`user`/`assistant`/`tool_call`/`result` e `session_id` | F | `cursor.com/docs/cli/reference/output-format` |
| H19 | Config de hooks: `~/.cursor/hooks.json`, `<projeto>/.cursor/hooks.json`, enterprise `/etc/cursor/hooks.json` | F | `cursor.com/docs/hooks` |
| H20 | Codex só decide `allow`/`deny` (sem sugestões, sem `updatedInput`, sem perguntas); Claude tem tudo isso | F | `mod/habblaud-codex/hook.mjs:145-155`; `mod/habblaud-permissoes/hooks/permission-hook.mjs:131-155` |

### Cursor (lacunas a validar em runtime)

| # | Pergunta | Tipo | Por que importa |
| --- | --- | --- | --- |
| C1 | Qual o formato e o local real do arquivo apontado por `transcript_path`? É JSONL? Tem papéis/tool calls/subagentes? | P0: 🟡 parcial | Sem isso não há atividades, terminal, tarefas nem histórico |
| C2 | O Cursor grava algum "registro de sessões abertas" (equivalente a `sessions/<pid>.json`)? Onde? | ? | Define a detecção de presença sem hooks |
| C3 | Quais eventos de hook realmente disparam na **IDE** (Cmd+K e Agent Chat) e no **CLI**? | P1: ✅ IDE (15 eventos); CLI: ? | Relatos: CLI dispara só shell hooks; IDE compatível com Claude hooks |
| C4 | Um hook `beforeShellExecution`/`preToolUse` pode **bloquear esperando** (long-poll) a decisão do escritório e devolver `allow`/`deny`? Qual o timeout máximo? | P2: ✅ (~20 s medido; config 35 s); ≥30 s por medir | É o coração do "responder pelo escritório" |
| C5 | Como o Cursor modela "esperando o usuário"? Existe evento específico ou é o próprio hook de permissão? | ? | Define o status `waiting` |
| C6 | Existe superfície de **uso/limites do plano** legível localmente (request counts, créditos)? | ? | O cartão de uso depende disso; pode não ter equivalente |
| C7 | É possível **injetar uma mensagem** numa sessão viva (equivalente a `$.prompt.submit`)? `followup_message` só vale em `stop`/`subagentStop`? | ? | Define se "Mandar mensagens" é viável |
| C8 | Como mapear `conversation_id` ↔ `session_id` ↔ transcript e o ciclo de vida de subagentes? | ? | Ids estáveis do `AgentInfo` |
| C9 | O Cursor carrega hooks de **plugin/marketplace** ("Customize") e de hooks de terceiros juntos? Há conflito? | ? | Distribuição/instalação do hook |
| C10 | Autenticação/uso: há API local para ler plano (Free/Pro/Business) e limites? | ? | Sem isso o cartão de uso fica vazio |
| C11 | O Cursor expõe MCPs e Tools no `preToolUse` com nomes estáveis (`Shell`, `Read`, `Write`, `Grep`, `Task`)? | P1: ✅ (`tool_name` estável) | Tradução de atividades e matchers |
| C12 | `transcript_path` é estável entre turns? Aponta para o transcript da conversa inteira ou por geração? | ? | Tail incremental (`FileTail`) depende disso |

> **Regra de trabalho**: nenhuma linha de `06-arquitetura-proposta.md` deve virar código sem que
> as lacunas `C1`, `C3`, `C4`, `C7` tenham sido respondidas por uma PoC (ver `07-poc-priorizados.md`).
>
> **Situação (2026-10-10, POC P0–P3)**: `C3` ✅ e `C4` ✅ respondidas; `C11` ✅; `C1` 🟡 **parcial**
> (JSONL legível, tool-calls por mapear); `C7` (injeção de mensagem) segue aberta (PoC P6). Relatório:
> `../relatorio-poc-cursor-2026-10-10.md`.
