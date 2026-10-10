# Investigar o Habblaud para criar uma alternativa integrada ao Cursor

Utilize o REA para conduzir uma investigação técnica do Habblaud, disponível em https://github.com/marmottajr/habblaud e em ./projects/habblaud-main

## Objetivo

Entender como o aplicativo funciona internamente para projetar um subproduto próprio que ofereça uma experiência semelhante de visualização e ou adicionar suporte para acompanhamento de agentes de programação, mas voltada ao Cursor.

Se for possível extender este já implementado para conseguir usa-lo com o cursor, deve ser prioridade. Mas a principio iremos apenas estudar.

## Etapas da investigação

1. Analise a estrutura do repositório e identifique as principais tecnologias, módulos, processos e responsabilidades.
2. Investigue como o aplicativo detecta projetos e sessões de agentes.
3. Trace o fluxo de dados desde a origem dos eventos até a atualização da interface.
4. Identifique como são modelados os estados dos agentes, as atividades, as solicitações de intervenção e o histórico.
5. Investigue os mecanismos de atualização em tempo real, persistência, monitoramento de arquivos e integração com processos externos.
6. Identifique como o frontend representa visualmente os agentes e sincroniza as mudanças de estado.
7. Investigue as dependências de funcionalidades específicas do Claude Code e as abstrações que poderiam permitir outros provedores.
8. Compare cada mecanismo relevante com as interfaces e possibilidades de integração disponíveis no Cursor.

## Regras

- Prefira evidências do código-fonte e dos resultados produzidos pelo REA.
- Utilize análise estática e investigação de comportamento quando necessário.
- Não presuma que uma funcionalidade do Claude Code exista no Cursor.
- Diferencie fatos comprovados, hipóteses e pontos ainda não investigados.
- Não comece a implementar o produto antes de concluir o diagnóstico.

## Entregáveis

Produza:
- Um mapa da arquitetura do Habblaud.
- Um diagrama dos principais fluxos de dados.
- Um inventário dos módulos relevantes, com referências aos arquivos.
- Uma análise das integrações específicas do Claude Code.
- Uma matriz de compatibilidade com o Cursor, indicando o que precisa ser investigado ou validado.
- Uma proposta de arquitetura para o novo subproduto.
- Uma lista priorizada de provas de conceito para validar a viabilidade técnica.

Ao terminar, apresente as evidências e as incertezas antes de recomendar a implementação.