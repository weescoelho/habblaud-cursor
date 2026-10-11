# Relatório de testes — POCs P4–P8 (2026-10-11)

Data (UTC): 2026-10-11
Repositório: habblaud (branch cursor-poc)
Runbook: docs/analise/10-runbook-poc-cursor-p4-p8.md
Ambiente: Linux 6.8.0-146-generic · Node v22.23.2

## Execução prática (validação automatizada + testes com CLI cursor-agent)

O cursor-agent foi localizado em /home/weesc/.local/bin/cursor-agent (v2026.10.01-e373342).

### P5 — CLI headless (stream-json)
Comando: cd /tmp && cursor-agent -p "responda apenas OK" --output-format stream-json --trust

Resultado (NDJSON capturado):
- Linhas: 7
- Session-id: 8293ae96-8626-4af6-a2df-3f10a908bc9d
- Tipos: assistant 1, result 1, system 1, thinking 3, user 1
- Ferramentas: 0 distintas, 0 chamadas (resposta direta OK)

Caracterização via harness: `cursor:poc:cli --path /tmp/cursor-stream.ndjson` → Session-id correto, eventos 7, ilegíveis 0. **Sucesso**.

### P4 — Subagentes
Após reinstall: `npm run cursor:poc:install` instala 17 eventos (incluindo subagentStart/subagentStop). Com `HABBLAUD_CURSOR_POC_DIR=/tmp/habblaud-capture-real`, `cursor:poc:subagents` reporta "não observado" (diretório vazio). Não houve disparo de subagentStart/subagentStop nesta sessão de teste. O harness está pronto para correlacionar quando esses eventos chegarem (teste automatizado cobre par casado/órfão/sem id). **Instrumentado, validação ao vivo pendente**.

### P6 — Injeção de mensagem (stop.followup_message)
- `followup --message "testando injeção"` → ✓ followup_message: testando injeção, exit 0
- `followup` (sem mensagem) → ✓ followup_message: Continue a tarefa seguindo as instruções anteriores, exit 0
Probe nunca lança; saída inválida falha barulhenta (testado nos testes automatizados). **Forma validada. Efeito real ao vivo pendente**.

### P7 — Histórico
Fixtures sintéticas e testes passam (2 sessões, efeitos git push/gh pr create/merge, ignora linha corrompida). Executado contra export real → 2 sessões reconstruídas (25 e 9 turnos), ferramentas variadas, efeitos 0 na amostra. **Sucesso (parcial)**.

### P8 — Uso/limites
Varredura com fixtures e contra export → nenhum arquivo candidato de primeira parte; ignora projects/; JSON mostra só chaves de topo; SQLite mostra tabelas+contagens. **Falha (provável) confirmada** (ocultar cartão).

## Gate automatizado
typecheck OK; vitest 1265 passed | 2 skipped; build OK. Validação independente: 23/23 ACs PASS, 5/5 mutações matadas, validate_state exit 0.

## Conclusão
POCs P4–P8 implementados e validados (automatizado). P5 validado com NDJSON real do cursor-agent (trust). P4/P6 precisam de exercício da IDE para gerar eventos ao vivo; harness pronto.
