# habblaud-cursor

Workspace das POCs da integração **Cursor × Habblaud**. Reúne a documentação de
análise e o fork do Habblaud **como submódulo**, para que um único clone traga tudo.

## Estrutura

- `docs/analise/` — análise `00`–`07` (arquitetura, fluxos de dados, modo Claude Code,
  compatibilidade com o Cursor e POCs priorizados).
- `projects/habblaud-main/` — **submódulo** do fork
  [`weescoelho/habblaud`](https://github.com/weescoelho/habblaud), na branch `cursor-poc`.
  É onde vive o código das POCs (provedor `cursor`, harnesses locais e o runbook `08`).
- `.opencode/`, `.agents/` — skills e configuração de agentes usados no desenvolvimento.

## Clonar

Como há um submódulo, clone com recursão:

```bash
git clone --recurse-submodules https://github.com/weescoelho/habblaud-cursor.git
cd habblaud-cursor
```

Se já clonou sem recursão:

```bash
git submodule update --init --recursive
```

O submódulo fica **fixado** no commit registrado (detached HEAD). Para acompanhar a
ponta da branch `cursor-poc`:

```bash
git submodule update --remote projects/habblaud-main
```

Para trabalhar dentro do submódulo:

```bash
cd projects/habblaud-main
git checkout cursor-poc
git pull
```

## Rodar as POCs

Depois de clonar, siga o runbook **dentro do submódulo**:

```bash
cd projects/habblaud-main
npm install
```

- Runbook: `projects/habblaud-main/docs/analise/08-runbook-poc-cursor.md`
- Ordem recomendada: **P1 → P0 → P2 → P3**.

Requisitos: **Node ≥ 22.12** (usa `node:sqlite`) e Cursor logado.

## Notas

- `.serena/` (estado local da tooling) e o `.specs/` da **raiz** (de outro projeto,
  `api-routes`) **não** são versionados aqui.
- O `.specs/` das POCs vive dentro do submódulo, em
  `projects/habblaud-main/.specs/features/cursor-poc/`.
- Repositório **privado**.
