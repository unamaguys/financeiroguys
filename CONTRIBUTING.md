# Como trabalhar no FinanceiroGuys

## Escolha uma issue

Consulte o Project e as dependências. Confirme o responsável real antes de iniciar. Cada história inclui interface, API, persistência, testes e documentação; não distribua o trabalho por camada.

O campo **Responsável sugerido** usa I1–I5. O campo nativo **Assignees** fica reservado às pessoas que assumirem a tarefa. Nenhum perfil é atribuído por suposição.

## Estados do quadro

- **Backlog:** trabalho planejado, ainda sem início.
- **Pronto:** escopo entendido e condições para começar atendidas.
- **Em andamento:** implementação em curso.
- **Em revisão:** PR aberto e pronto para avaliação.
- **Concluído:** critérios aceitos, revisão feita e mudança integrada.

Os vínculos **blocked by** representam dependências para concluir a integração. É possível começar telas e testes com contratos simulados. Antes de mover para Concluído, remova os mocks do fluxo entregue.

## Prioridade e marcos

**P0** libera a base e os primeiros registros financeiros. **P1** completa o MVP. A prioridade não representa autorização para cortar uma funcionalidade sem combinar com a equipe.

Os quatro marcos são Fundação, Desenvolvimento dos módulos, Integração e Qualidade e entrega. Eles não têm datas enquanto a equipe não confirmar disponibilidade e prazo.

## Branches e pull requests

Use branches curtas, como `feat/E2-02-registrar-lancamento`. Entregue uma história ou uma parte demonstrável por PR. Relacione a issue com `Closes #numero` quando o PR concluir todos os critérios.

Cada módulo tem um dono. Combine mudanças em contratos com os consumidores. Alterações no schema compartilhado e no registro central de rotas entram em PRs pequenos, com uma pessoa integradora por vez.

Não altere migrations já aplicadas pelos colegas. Crie uma migration corretiva e confira a sequência em banco vazio.

## Definição de pronto

- Critérios da issue atendidos no fluxo real.
- Autorização e validação feitas na API.
- Interface com carregamento, vazio e erro quando aplicável.
- Testes das regras e dos riscos afetados aprovados.
- Tipos, build e verificações de CI aprovados.
- OpenAPI e documentação do módulo atualizadas.
- Revisão de um colega e integração na branch principal.

Uma issue de planejamento só está concluída quando seus artefatos foram revisados e podem ser usados pela equipe. Um épico acompanha as histórias filhas e não deve duplicar suas estimativas.

## Revisão cruzada sugerida

I2 revisa I1; I3 revisa I2; I5 revisa I3; I1 revisa I4; I4 revisa I5. Se alguém estiver indisponível, outro colega pode revisar. O autor permanece responsável por corrigir e integrar sua entrega.
