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
| **P0** transcript | **Sucesso parcial** | `transcript_path` → arquivo **JSONL** legível (60 linhas, papéis `user`/`assistant`). **Lacuna C1 permanece**: o inspetor não reconheceu campos de tool-call nas linhas — derivação de atividades só por transcript ainda limitada; dependemos de hooks para o rico. |
| **P1** cobertura de hooks (IDE) | **Sucesso** | **15/15** eventos candidatos dispararam (`sim`) com `conversation_id`, `workspace_roots`, `cwd`, `model`, `user_email`, `transcript_path` etc. **`subagentStart`/`subagentStop` não observados** nesta rodada → ver P4. |
| **P2** bloqueio do hook de decisão | **Sucesso** | Formato nativo `{"permission":"allow"\|"deny"}` validado; stub/harness bloqueia até ~25 s (padrão do hook real); ao vivo confirmado o long-poll até `decisionTimeoutS` (35 s) com decisão pela porta 4747. |
| **P3** ingest mínimo | **Sucesso** | `POST /api/cursor/events` → agente `provider:"cursor"` aparece na sala, muda de status (`working`/`idle`), some no `sessionEnd`; sem regressão em Claude/Codex. Casos negativos: 400 (body inválido), 403 (Host externo), `{ok:false}` (evento desconhecido). |

**Ajustes do P0 em relação ao critério original:**

- **C1** (formato do transcript): parcialmente respondida — é JSONL com turnos, mas **sem campos de
  tool-call reconhecidos pelo inspetor** nos exemplos observados. O esqueleto
  `server/sources/cursor/transcript.ts` permanece defensivo (cobre estilos OpenAI *e* Anthropic sem
  lançar); a confirmação do schema depende de mais amostras do arquivo real.
- **C2** (registro de sessões): continua aberta — presença segue 100% por hooks (`sessionStart`/`End`
  + fonte com `STALE_PRESENCE_MS`).

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
> P1 15/15 hooks, P3 agente no escritório sem regressão, e P0 confirmou transcript JSONL
> legível (atividades ricas ficam para a Fase 3, quando o schema C1 for confirmado).
> **Só não iniciar Fase 1 sem fechar C1** se a paridade de atividades/terminal/histórico
> for prometida como igual à do Claude Code — nesse caso, a modalidade comprovada hoje é
> "hooks-first" (presença/status/permissões/subagentes), com transcript complementar.

Se **P0 falhar**, o subproduto ainda é viável em modo "hooks-only" (presença/status/subagentes/
permissões), porém **sem** terminal, tarefas, tokens, GitHub e histórico no nível do Claude Code.
Isso deve ser comunicado explicitamente antes de prometer paridade. Hoje (2026-10-10) o cenário é
**parcial**: transcript JSONL legível, mas sem tool-calls reconhecidos (C1) — a promessa de
paridade continua condicionada ao schema.
