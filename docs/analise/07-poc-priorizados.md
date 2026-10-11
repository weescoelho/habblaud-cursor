# 07 — Provas de conceito priorizadas

Cada PoC tem: **hipótese**, **método**, **critério de sucesso/falha**, **esforço** (P/M/G),
**dependências** e **decisão que destrava**. Ordem por risco/retorno.

> Regra: os PoCs são **descartáveis**. O objetivo é responder perguntas, não construir o produto.
> Nenhum código de produção deve começar antes de P0–P3.

---

## P0 — Descobrir o transcript do Cursor (`transcript_path`)  ★ bloqueante

- **Hipótese**: o Cursor grava um arquivo local (JSONL ou JSON) com a conversa, papéis, tool calls
  e possivelmente uso/subagentes, apontado por `transcript_path` nos hooks.
- **Método**: instalar um hook mínimo `afterAgentResponse`/`preToolUse` que **copia `stdin`** e o
  valor de `transcript_path`/`CURSOR_TRANSCRIPT_PATH` para `/tmp`; rodar uma conversa no Agent
  Chat com shell, edição, MCP e subagente (`Task`); inspecionar o arquivo (caminho, formato,
  campos, incremental, tamanho, quando é criado/rotacionado).
- **Sucesso**: existe arquivo legível, com delimitação de turnos/tool calls, estável entre turns;
  dá para derivar `Activity` sem hooks. **Falha**: arquivo inexistente/opaco·criptografado (ex.:
  SQLite `state.vscdb`) → dependemos 100% de hooks para atividades.
- **Esforço**: M. **Depende**: —. **Destrava**: Fase 3; e a estratégia geral (arquivos vs hooks).
- **Entrega**: um dump anotado do formato + um esboço de `server/sources/cursor/transcript.ts`.

## P1 — Cobertura real de eventos de hook na IDE  ★ bloqueante

- **Hipótese**: na IDE, disparam `sessionStart/End`, `pre/postToolUse`, `subagentStart/Stop`,
  `before/afterShellExecution`, `before/afterMCPExecution`, `beforeReadFile`, `afterFileEdit`,
  `beforeSubmitPrompt`, `preCompact`, `stop`, `afterAgentResponse/Thought`.
- **Método**: hook "log tudo" gravando `hook_event_name` + stdin; exercitar Cmd+K, Agent Chat,
  subagente (`Task`), shell, MCP, edição de arquivo, cancelamento, fim de turno. Registrar quais
  eventos realmente chegam e com que campos (`conversation_id`, `generation_id`, `cwd`?,
  `session_id`?).
- **Sucesso**: cobertura suficiente para alimentar `working/idle/shell/done` e `subagent*`.
  **Falha parcial**: se faltar algum, o status cai para heurística por hook disponível.
- **Esforço**: M. **Depende**: —. **Destrava**: Fase 1; confirma `05.3`.
- **Entrega**: matriz "evento → dispara? → campos" (complementa `05.1`).

## P2 — Hook de decisão consegue bloquear esperando o escritório?  ★ bloqueante

- **Hipótese**: `beforeShellExecution` (e `beforeMCPExecution`/`preToolUse`) pode rodar por vários
  segundos/minutos, consultar o Habblaud e devolver `permission: allow|deny`.
- **Método**: hook de `beforeShellExecution` que executa `sleep N` variável antes de responder;
  medir quando o Cursor desiste/expira; testar com a página aberta e fechada; testar long-poll
  real contra um `PermissionRegistry` local.
- **Sucesso**: N ≥ ~30 s confiável e a decisão aplicada. **Falha**: timeout curto → só observação
  (permissão fica 100% no terminal).
- **Esforço**: M. **Depende**: —. **Destrava**: Fase 2 (o "responder pelo escritório").
- **Entrega**: tempo máximo seguro + formato de saída validado.

## P3 — Ingest mínimo + agente vivo no escritório

- **Hipótese**: uma rota `POST /api/cursor/events` + `CursorSource` mínima fazem um agente Cursor
  aparecer e mudar de status no escritório, reusando `Office`/`Hub` sem tocar no cliente.
- **Método**: implementar `CursorLive`/`CursorSource` esqueleto (só `sessionStart`/`preToolUse`/
  `stop`/`sessionEnd`), registrar em `server/index.ts`, apontar o hook de P1 para ela, abrir o
  navegador. Verificar sala (via `workspace_roots`), nome, status, feed.
- **Sucesso**: personagem aparece, "trabalha" e sai, sem regressão em Claude/Codex.
  **Falha**: acoplamento escondido → identificar e isolar.
- **Esforço**: M/G. **Depende**: P1. **Destrava**: Fase 1; valida a tese "é extensão".
- **Entrega**: PR esqueleto + roteiro de demonstração.

## P4 — Subagentes e ciclo de turno

- **Hipótese**: `subagentStart`/`subagentStop` (com `subagent_id`/`subagent_type`/`task`) mapeiam
  para `addSub`/`completeSub`, e `conversation_id`/`parent_conversation_id` ligam pai↔filho.
- **Método**: disparar `Task` (explore/generalPurpose) e mapear ids; comparar com o modelo de
  subagente do Habblaud.
- **Sucesso**: subagente entra, trabalha e entrega. **Falha**: sem ids correlacionáveis → sub
  aparece sem pai (degradação).
- **Esforço**: M. **Depende**: P1/P3. **Destrava**: Fase 4.

## P5 — CLI headless (`cursor-agent`) como fonte alternativa

- **Hipótese**: no CLI, os hooks são parciais/instáveis, mas `-p --output-format stream-json`
  fornece um NDJSON rico (`system`/`user`/`assistant`/`tool_call`/`result`).
- **Método**: rodar `cursor-agent -p` com stream-json; medir eventos, `session_id`, riqueza de
  `tool_call`; comparar com hooks no mesmo ambiente; testar Linux vs macOS.
- **Sucesso**: dá para observar o CLI pelo stream (mesmo sem hooks). **Falha**: só a IDE é viável.
- **Esforço**: M. **Depende**: —. **Destrava**: escopo "CLI suportado?".
- **Evidência de risco**: fórum oficial relata hooks parciais no `cursor-agent` e nenhum hook no
  Linux em certas versões.

## P6 — Injeção de mensagem na sessão

- **Hipótese**: é possível injetar texto numa sessão viva (equivalente a
  `$.prompt.submit`) — talvez só via `stop.followup_message`.
- **Método**: hook `stop` retornando `followup_message`; verificar se o agente continua com o
  texto; investigar SDK/ACP (Agent Client Protocol) ou CLI como alternativa.
- **Sucesso**: mensagem "entra como usuário" ao fim do turno. **Falha**: v1 sem "Mandar mensagens".
- **Esforço**: M. **Depende**: P1. **Destrava**: Fase 5.

## P7 — Histórico e GitHub (dependem de P0)

- **Hipótese**: com o transcript (P0), dá para listar sessões recentes (`CursorHistory`) e detectar
  comandos `gh`/`git` para os efeitos de sala.
- **Método**: adaptar `CodexHistory`/`SessionHistory` ao formato descoberto; testar detecção de
  `git push`/`gh pr create`.
- **Sucesso**: histórico e efeitos funcionam. **Falha**: histórico limitado ao período observado.
- **Esforço**: M/G. **Depende**: P0. **Destrava**: Fase 4.

## P8 — Uso/limites do plano

- **Hipótese**: existe alguma fonte local (ou API/servidor do Cursor) com limites de uso.
- **Método**: procurar em `~/.cursor/` por arquivos de plano/estado; verificar headers/API do Cursor;
  confirmar que não há hook equivalente a `rate_limits`.
- **Sucesso**: dá para exibir algo. **Falha (provável)**: ocultar o cartão para Cursor.
- **Esforço**: P/M. **Depende**: —. **Destrava**: decisão de UI (cartão de uso).

---

## Resultados executados (P0–P3, 2026-10-10)

> Fonte: `docs/relatorio-poc-cursor-2026-10-10.md`. Validação **ao vivo** na IDE com assinatura
> ativa, via runbook `…/08-runbook-poc-cursor.md` (dentro do fork, branch `cursor-poc`).

| PoC | Resultado | Observações |
| --- | --------- | ----------- |
| **P0** transcript | **Sucesso** | `transcript_path` → **JSONL** legível. Schema mapeado (**C1 ✅**, amostra de 64 linhas): `{"role":…,"message":{"content":[{"type":"text"\|"tool_use","name","input"}]}}` + marcadores `{"type":"turn_ended","status":…}`. Ferramentas: `Shell`, `Read`, `Write`, `StrReplace`, `Glob`, `Grep`, `CallDynamicTool`. Parser ajustado (`transcript.ts` lê o `message` aninhado; `describeCursorTool` traduz os nomes) — o harness agora reporta `Tool-calls: sim`. |
| **P1** cobertura de hooks (IDE) | **Sucesso** | **15/15** eventos candidatos dispararam (`sim`) com `conversation_id`, `workspace_roots`, `cwd`, `model`, `user_email`, `transcript_path` etc. **`subagentStart`/`subagentStop` não observados** nesta rodada → ver P4. |
| **P2** bloqueio do hook de decisão | **Sucesso (ressalva)** | Formato nativo `{"permission":"allow"\|"deny"}` validado; bloqueio medido até **~20 s** (cap do stub 25 s); configuração real `decisionTimeoutS: 35`; o critério de **≥30 s ao vivo** ainda não foi medido. |
| **P3** ingest mínimo | **Sucesso** | `POST /api/cursor/events` → agente `provider:"cursor"` aparece na sala, muda de status (`working`/`idle`), some no `sessionEnd`; sem regressão em Claude/Codex. Casos negativos: 400 (body inválido), 403 (Host externo), `{ok:false}` (evento desconhecido). |

**Ajustes do P0 em relação ao critério original:**

- **C1** (formato do transcript): **respondida ✅** (amostra coletada pelo runbook
  `…/09-runbook-coleta-transcript-c1.md`, evidência `docs/resultado_test_c1.txt`). O arquivo é
  JSONL e os **tool-calls vêm aninhados em `message.content`** (blocos `tool_use`), **não** no topo
  da linha — era exatamente isso que o inspetor não reconhecia. O `server/sources/cursor/transcript.ts`
  passou a ler **os dois contêineres** (topo *e* `message`) e o `describeCursorTool` cobre os nomes do
  transcript (`Shell`, `StrReplace`, `CallDynamicTool`, `Glob` com `glob_pattern`).
- **C2** (registro de sessões): continua aberta — presença segue 100% por hooks (`sessionStart`/`End`
  + fonte com `STALE_PRESENCE_MS`).

---

## Resultados executados (P4–P8, 2026-10-11)

> Fonte: harness local (`scripts/cursor-poc.ts` + `mod/habblaud-cursor/followup-hook.mjs`, branch
> `cursor-poc`) e a leitura já feita sobre o export real do usuário. Runbook: `10-runbook-poc-cursor-p4-p8.md`
> (dentro do fork). **Privacidade**: a evidência abaixo só tem contagens/rótulos/nomes de ferramenta —
> caminhos/projetos reais do export não entram em commits.

| PoC | Resultado | Observações |
| --- | --------- | ----------- |
| **P4** subagentes | **Instrumentado (método corrigido); validação ao vivo pendente** | A rodada P0–P3 não instalava `subagentStart`/`subagentStop` — o "não observado" era **artefato do método**, não resposta. O `install` agora instala **17 eventos** e o `cursor:poc:subagents` correlaciona `Start`↔`Stop` por id. Sobre a captura real (~1.783 eventos, 0 linhas ruins, anterior à correção) segue "não observado", mas o transcript mostra uso de `Task` (`mcp__cursor__Task`) → há subagente sem hook correlacionável nessa amostra. **Re-testar pelo runbook `10`.** |
| **P5** CLI headless | **Instrumentado; validação ao vivo pendente** | `cursor:poc:cli` caracteriza o NDJSON (`session_id`, tipos, ferramentas, linhas ilegíveis). O CLI `cursor-agent` está instalado; falta só o NDJSON real (runbook `10`). |
| **P6** injeção de mensagem | **Forma validada; efeito real pendente** | O probe `followup-hook.mjs` devolve `{"followup_message": …}` no `stop` (validado por `cursor:poc:followup`, inclusive falha barulhenta para saída inválida). A **injeção real** exige instalar o probe à mão no `stop` (runbook `10`); até lá, **C7** segue aberta. |
| **P7** histórico/GitHub | **Sucesso (parcial)** | `cursor:poc:history` reconstruiu **2 sessões** reais (**25** e **9** turnos) com ferramentas (`Read`, `Glob`, `Grep`, `Write`, `StrReplace`, `Bash`, `GetDynamicTools`, `UpdateCurrentStep`, `AskQuestion`, `Task`), reusando o parser do transcript. Detecção de efeitos `git push`/`gh pr create`/`gh pr merge` **instrumentada** (0 na amostra). Histórico viável (Fase 4). |
| **P8** uso/limites do plano | **Falha (provável) confirmada localmente** | `cursor:poc:usage` **não** achou fonte de primeira parte; `cli-config.json`/`storage.json` só têm config/telemetria; `plans/` é **plan mode**; o lead vivo é `ai-tracking/` (não confirmado). → **ocultar o cartão** para o Cursor (como previsto). |

**Notas de método (P4/P8):**

- **P4**: a correção de método (instalar os hooks de subagente) muda a pergunta de "o Cursor emite?"
  para "o Cursor emite **quando instalado**?" — só o re-teste ao vivo responde.
- **P8**: a ausência de fonte local é **resultado**, não lacuna: nenhum arquivo de primeira parte
  casou no `usage`; os hits de "uso/plano" são arquivos de **projeto** (`plans/`, `planos.json`) e
  telemetria (`statsig`).

---

## Resumo de priorização

| Ordem | PoC | Risco que reduz | Bloqueia implementação? |
| --- | --- | --- | --- |
| 1 | **P0** transcript | Paridade de atividades/terminal/histórico | **Sim** |
| 2 | **P1** cobertura de hooks (IDE) | Status/atividades ao vivo | **Sim** |
| 3 | **P2** bloqueio do hook de decisão | "Responder pelo escritório" | **Sim** |
| 4 | **P3** ingest mínimo | Tese "é extensão" | **Sim** |
| 5 | P4 subagentes | Fase 4 | Não |
| 6 | P5 CLI | Escopo CLI | Não |
| 7 | P6 mensagens | Fase 5 ("Mandar mensagens") | Não |
| 8 | P7 histórico/GitHub | Fase 4 | Não |
| 9 | P8 uso do plano | UI (cartão de uso) | Não |

## Critério de "go" para implementar

Implementar de verdade (Fase 1) **somente se**:
- P1 confirmar cobertura suficiente de hooks na IDE; **e**
- P3 demonstrar um agente Cursor no escritório sem regressão; **e**
- P0 (ou, na sua falta, P2) mostrar caminho para atividades/permissões.

> **Veredicto de 2026-10-10: GO ✅** — as três condições foram atendidas ao vivo:
> P1 15/15 hooks, P3 agente no escritório sem regressão, e P0 confirmou o transcript JSONL
> **com os tool-calls mapeados** (**C1 ✅**, fechado no mesmo dia). Com o C1 fechado, a paridade de
> atividades/terminal/histórico deixa de depender de uma incógnita de schema e passa a ser trabalho
> de Fase 1/3 — o `transcript.ts` já lê o formato real.

Se **P0 falhar**, o subproduto ainda é viável em modo "hooks-only" (presença/status/subagentes/
permissões), porém **sem** terminal, tarefas, tokens, GitHub e histórico no nível do Claude Code.
Isso deve ser comunicado explicitamente antes de prometer paridade. Hoje (2026-10-10) o cenário é
**favorável**: transcript JSONL legível **com tool-calls mapeados** (**C1 ✅**) — a paridade passa a
depender do esforço de implementação da Fase 1, não de um schema desconhecido.
