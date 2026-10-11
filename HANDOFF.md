# Handoff — Cursor POCs P4–P8

**Data:** 2026-10-10 (sábado)  
**Estado:** Design escrita + Tasks escritas + Spec validada. **Aguardando reinício da execução.**

---

## O que já foi entregue (commitado)

| Arquivo | Conteúdo | Status |
|---------|----------|--------|
| `.specs/features/cursor-poc-p4-p8/spec.md` | 23 requisitos AC (CP48-01..23); 6 user stories (S1–S6); já validada: `0 error(s), 0 warning(s)` | ✅ |
| `.specs/features/cursor-poc-p4-p8/design.md` | Visão arquitetural + Evidência real (export do usuário) + Decisões técnicas; conforma com `AD-001` | ✅ |
| `.specs/features/cursor-poc-p4-p8/tasks.md` | 8 tasks (T1..T8) organizadas em 6 fases; dependências; gate check; diagram cross-check; 0 errors, 2 warnings (docs→matrix `none`) | ✅ |

---

## Descobertas-chave do export real (`docs/cursor-export/`)

| POC | O que encontramos | Impacto |
|-----|------------------|---------|
| **P4** | `p4-events.jsonl` (1784 eventos, 0 ruínas) tem `Task` (3×) mas **nenhum `subagentStart`/`subagentStop`**; `HARNESS_EVENTS` nunca os instalou | **Método incorreto** — corrigir: adicionar `subagentStart`/`subagentStop` a `HARNESS_EVENTS` e re-testar (AC CP48-05). |
| **P5** | CLI `cursor-agent` instalado em `~/.config/Cursor/User/globalStorage/anysphere.cursor-agent-worker/agent-cli/.local/share/cursor-agent/versions/` (várias versões) + `agent-cli-state.json` (388 B) | Runbook P5 pode apontar para o binário existente; falta só o NDJSON `stream-json`. |
| **P7** | 2 transcripts reais parseiam limpo: `roles user/assistant`, tools `Read/Glob/Write/Grep/Shell/StrReplace/GetDynamicTools/CallDynamicTool/AskQuestion/UpdateCurrentStep/List`; 5 sessões distintas em `projects/home-jcoelho3-habblaud/agent-transcripts/`; novas ferramentas sem tradução dedicada (`GetDynamicTools`, `AskQuestion`, `UpdateCurrentStep`, `List`) | Histograma/efeitos reconstruíveis; as novas ferramentas caem no nome cru (aceitável POC). |
| **P8** | Raiz `~/.cursor` tem `plans/` (26 *.plan.md* — modo plano, não billing), `ai-tracking/` (não listado), `agent-cli-state.json`, `statsig-cache.json` (850 KB), extensão `cursor-token-monitor` (terceiros); `storage.json` só telemetria; `state.vscdb` sem tabelas listadas | **Nenhuma fonte de primeira parte** de uso/plano encontrada → provável veredito "ocultar o cartão". Leads: `ai-tracking/`, `state.vscdb` tables, extensão token-monitor. |

> **Privacidade:** `docs/cursor-export/` **não** entra em commits (dados pessoais). Evidência nos docs usa apenas contagens/rótulos.

---

## O que falta (próximos passos)

### Phase 1 — Task T1 (P8 `usage` subcommand)
- Implementar `scanUsage`, `describeUsageFile`, `--path` no `scripts/cursor-poc.ts`.
- Testes unitários; gate `npx vitest run server/test/cursor-poc.test.ts`.
- **Aguardo comando para iniciar** (ou para pular para outra task).

### Phase 2 — Task T2 (P4 subagents + hooks)
- Adicionar `subagentStart`/`subagentStop` a `HARNESS_EVENTS`.
- Implementar `correlateSubagents` e o subcomando `cursor:poc:subagents`.
- Gate quick.

### Phase 3 — Task T3 (P5 CLI stream-json)
- Subcomando `cursor:poc:cli [--path]` que lê NDJSON.
- Gate quick.

### Phase 4 — Tasks T4/T5 (P6 followup probe + subcommand)
- Criar `mod/habblaud-cursor/followup-hook.mjs` (probe).
- Subcomando `cursor:poc:followup` que valida a forma.
- Gate full (spawn/integração).

### Phase 5 — Task T6 (P7 history + git/gh)
- Subcomando `cursor:poc:history [--projects <dir>]`.
- Gate quick.

### Phase 6 — Tasks T7/T8 (Docs + registro)
- T7: runbook `docs/analise/10-runbook-poc-cursor-p4-p8.md` (fork).
- T8: atualizar `07`/`00`/`05`/`relatorio` (pai) + `.gitignore` para `docs/cursor-export/`.

---

## Como prosseguir

1. **Confirmação:** Diga `pode seguir` para eu iniciar a execução das tasks (T1 primeiro, em diante), ou me diga qual task/POC priorizar.
2. **Dados extras:** Se tiver os arquivos que me pediu antes (`ai-tracking/` listing, `agent-cli-state.json`, extensão `cursor-token-monitor` `out/usageIntelligence.js`, `state.vscdb` tables), pode enviar em `/tmp/opencode/cursor-export/` e eu continuarei enriquecendo o design/implementação.
3. **Pausa a qualquer momento:** Eu salvo o estado e retorno ao handoff.

--- 

Próxima task agendada: **T1** (subcomando `usage` do harness).  
Se quiser pular para outra task ou mudar a ordem, avise.