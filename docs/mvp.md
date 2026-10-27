# Escopo do MVP

Uma pessoa deve conseguir criar seu acesso, cadastrar contas, registrar movimentações e conferir saldos, orçamentos, metas e recorrências em um dashboard.

## Divisão fullstack

| Papel | Módulo | Entrega |
|---|---|---|
| I1 | Acesso e carteiras | Autenticação, perfil, contas, transferências, saldos e extratos |
| I2 | Lançamentos e categorias | Cadastros, receitas/despesas, correções, filtros e consultas financeiras |
| I3 | Orçamentos e relatórios | Limites, progresso e gastos por categoria; coordenação de CI e entrega |
| I4 | Metas | Cadastro, aportes, correções, progresso e arquivo |
| I5 | Recorrências e dashboard | Calendário mensal, confirmação sem duplicidade e visão consolidada |

Os papéis indicam a divisão sugerida; não substituem os responsáveis reais no GitHub. As tarefas de fundação e revisão são compartilhadas. Coordenar CI ou deploy não transforma um integrante em responsável exclusivo por uma camada.

## Limites da primeira entrega

Uma moeda (BRL), uso individual e lançamentos efetivados. Recorrências são mensais e só geram lançamento após confirmação. Aportes em metas acompanham valores declarados pelo usuário e não movimentam contas.

Comparação entre meses, CSV, IA, integração bancária, cartões/faturas, parcelamentos, múltiplas moedas, notificações e recuperação por e-mail ficam para depois do MVP. O relatório por categoria permanece porque também alimenta o dashboard.

## Organização do backlog

São cinco épicos, 21 histórias fullstack, cinco tarefas de fundação e três tarefas de integração/entrega. Contas e consulta de saldo foram separadas para que o cadastro desbloqueie lançamentos sem criar dependência circular.

As estimativas são referências iniciais em horas-pessoa e incluem testes e documentação. Reestimar após a primeira entrega vertical. Não há datas de calendário definidas; os marcos indicam sequência, não prazo acordado.

## Sequência de entrega

1. **Fundação:** estrutura executável, contratos, banco, componentes básicos e CI.
2. **Módulos:** cada dono entrega histórias completas e publica os serviços de consulta usados pelos demais.
3. **Integração:** dashboard real e conferência dos mesmos dados entre telas.
4. **Qualidade e entrega:** isolamento, concorrência, documentação e demonstração publicada.

Telas e regras podem avançar com dados simulados enquanto os contratos estabilizam. A conclusão de uma história exige integração real; o ambiente de demonstração não usa mocks.

## Roteiro de aceitação

Conta A com R$ 1.000 e B com R$ 0. Registrar receita de R$ 500 e despesa de R$ 200 em A; transferir R$ 300 para B. Esperado: A = R$ 1.000, B = R$ 300, saldo consolidado = R$ 1.300 e resultado do período = R$ 300.

Orçamento de R$ 500 com gasto de R$ 200 mostra 40% e R$ 300 restantes. Aporte declarado em meta não altera saldos. Confirmar duas vezes a mesma recorrência gera um único lançamento. Um segundo usuário não acessa esses registros.
