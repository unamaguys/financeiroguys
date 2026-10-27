# Arquitetura e regras financeiras

## Stack e estrutura

**Proposta de referência:** React + TypeScript no frontend; NestJS + TypeScript no backend; PostgreSQL e Prisma para persistência; REST documentada com OpenAPI; Docker Compose para ambiente local. Fixar versões estáveis compatíveis e lockfile na fundação. Se a equipe já dominar outra stack, preservar os mesmos limites de domínio em vez de adicionar uma curva de aprendizado desnecessária.

Um monorepositório, uma aplicação de frontend, uma API e um banco. Organização por domínio, com controllers para HTTP, services para regras e acesso a dados encapsulado quando necessário. Não criar abstrações genéricas de repositório ou uma arquitetura distribuída para este MVP.

```text
apps/
  web/src/
    app/                  # rotas, sessão e composição
    modules/
      identity/
      accounts/
      transactions/
      budgets/
      goals/
      recurring/
      dashboard/
    shared/               # componentes e cliente HTTP comuns
  api/src/
    modules/
      identity/
      accounts/           # cadastro, referências e transferências
      balance-queries/    # composição de saldos/extratos, proprietário I1
      transactions/       # categorias, lançamentos e consultas, I2
      budgets/
      reports/            # proprietário I3
      goals/
      recurring/
      dashboard/          # composição, proprietário I5
    shared/               # banco, erros, guard e infraestrutura
packages/
  api-client/             # cliente/tipos gerados do OpenAPI
docs/
  decisions/
  contracts/
```

Providers exportados formam a interface pública de cada módulo no NestJS. Exportar serviços pequenos e explícitos, evitando acesso direto a repositórios de outro domínio. Referência: [módulos do NestJS](https://docs.nestjs.com/modules).

## Limites e direção das dependências

| Serviço público | Dono | Responsabilidade | Consumidores |
|---|---|---|---|
| Contexto de autenticação/guard | I1 | Identificar o usuário e exigir sessão válida | Todos |
| `AccountReferenceService` | I1 | Validar proprietário, estado e abertura da conta | I2 e I5 |
| `TransferQueryService` | I1 | Consultar transferências | Saldos/extratos de I1 |
| `CategoryReferenceService` | I2 | Validar categoria, tipo e estado | I3 e I5 |
| `TransactionCommandService` | I2 | Criar lançamento com regras únicas; aceitar transação de banco do chamador | I5 |
| `FinancialQueryService` | I2 | Listar e agregar receitas/despesas por usuário, período, conta e categoria | I1, I3 e I5 |
| `BalanceQueryService` | I1 | Compor saldo/extrato com contas, lançamentos e transferências | I5 e telas de contas |
| `BudgetQueryService` / `ReportQueryService` | I3 | Progresso dos limites e relatório por categoria | I5 |
| `GoalQueryService` | I4 | Resumo de metas e aportes | I5 |

O cadastro de contas não importa lançamentos. Lançamentos importam apenas o módulo de referências de contas. O módulo separado `balance-queries` importa contas e lançamentos para compor leituras. Assim, não existe ciclo contas → lançamentos → contas. Essa separação é interna ao trabalho de I1.

Dashboard apenas consome consultas. Recorrências chamam lançamentos; lançamentos não importam recorrências. A proteção contra exclusão de lançamento recorrente vem da chave estrangeira restritiva, traduzida para erro 409 por I2, sem dependência de código inversa.

Não há chamadas HTTP entre módulos do backend: são chamadas de serviços dentro do mesmo processo. Cada consumidor passa o usuário autenticado; nenhum serviço de consulta exposto pode omitir esse filtro.

Os serviços de consulta aceitam um contexto transacional opcional para que saldo, extrato e dashboard possam compartilhar o mesmo snapshot. O chamador de composição abre a transação de leitura e repassa esse contexto; cada serviço continua dono de suas consultas.

## Regras financeiras compartilhadas

1. **Dinheiro:** `NUMERIC(14,2)` no banco, strings decimais na API, por exemplo `"125.90"`. Rejeitar mais de duas casas e valores fora do limite antes da gravação. Usar operações decimais na aplicação; não usar ponto flutuante como fonte dos cálculos. O PostgreSQL recomenda `numeric` quando se exige precisão monetária: [tipos numéricos](https://www.postgresql.org/docs/current/datatype-numeric.html).
2. **Datas:** data financeira em `DATE`, no formato `YYYY-MM-DD`; mês de orçamento como primeiro dia do mês; auditoria em timestamps UTC. “Hoje” segue `America/Sao_Paulo` no MVP. Evitar converter datas financeiras para instantes UTC na interface.
3. **Saldo até D:** saldo inicial das contas abertas até D + receitas até D − despesas até D + transferências recebidas até D − transferências enviadas até D. Todos os movimentos respeitam a data de abertura da conta. Saldo inicial pode ser negativo.
4. **Fonte do saldo:** consultas sobre fatos persistidos; não manter uma coluna `current_balance` atualizada por vários módulos. Saldo negativo é permitido, pois o aplicativo registra a realidade informada pelo usuário.
5. **Resultado do período:** receitas − despesas. Saldo inicial, transferências e aportes em metas ficam fora dessa conta.
6. **Transferências:** uma linha com origem, destino, valor e data. Não criar duas transações de receita/despesa para representar a transferência. A consulta usa a mesma linha nos dois lados.
7. **Orçamento:** somar despesas efetivadas por categoria e mês; não bloquear o lançamento ao ultrapassar o limite.
8. **Histórico:** arquivar conta/categoria preserva referências e totais. Não permitir exclusão física desses cadastros no MVP.
9. **Metas:** aportes declaratórios, sem vínculo com dinheiro em contas. Acumulado é soma dos aportes; atingida é condição calculada. Não existe campo de saldo da meta atualizado separadamente.
10. **Recorrências:** confirmar ocorrência é a única ação que gera lançamento. A chave regra/data impede repetição; registro da ocorrência e lançamento são gravados na mesma transação de banco. Atomicidade significa que todas as gravações são confirmadas ou nenhuma é aplicada: [transações no PostgreSQL](https://www.postgresql.org/docs/current/tutorial-transactions.html).
11. **Leituras consolidadas:** consultas que compõem vários totais da mesma resposta devem compartilhar uma visão consistente do banco; usar agregação única ou transação de leitura com snapshot consistente. Evitar que edição simultânea produza cartões e gráficos contraditórios.

## Banco de dados integrado

| Entidade | Campos principais | Relacionamentos/restrições | Dono |
|---|---|---|---|
| `users` | id, name, email, password_hash, created_at | E-mail normalizado único | I1 |
| `sessions` | id, user_id, token_hash, expires_at | N sessões para 1 usuário; revogação no logout | I1 |
| `accounts` | id, user_id, name, opening_balance, opening_date, active | N contas para 1 usuário | I1 |
| `transfers` | id, user_id, source_account_id, destination_account_id, amount, date, idempotency_key, request_hash | Origem diferente do destino; valor positivo; chave única por usuário | I1 |
| `categories` | id, user_id, name, normalized_name, type, active | Único usuário/tipo/nome normalizado | I2 |
| `transactions` | id, user_id, account_id, category_id, type, amount, date, description | Valor positivo; referências obrigatórias | I2 |
| `budgets` | id, user_id, category_id, month, limit_amount | Único usuário/categoria/mês; limite positivo | I3 |
| `goals` | id, user_id, title, target_amount, deadline, archived | Alvo positivo; progresso calculado | I4 |
| `goal_contributions` | id, user_id, goal_id, amount, date, note | Valor positivo; meta do mesmo usuário | I4 |
| `recurring_rules` | id, user_id, account_id, category_id, type, amount, anchor_day, start_date, active, description | Dia entre 1 e 31; valor positivo | I5 |
| `recurring_occurrences` | id, user_id, rule_id, occurrence_date, transaction_id | Único regra/data; transaction_id único, não nulo e com exclusão restrita | I5 |

Incluir timestamps de criação/atualização nas entidades mutáveis. Índices devem acompanhar as consultas reais, começando por `(user_id, date)` em lançamentos e `(user_id, month)` em orçamentos.

Validar a propriedade na aplicação e usar chaves estrangeiras compostas `(user_id, referência)` para referências entre registros do usuário. As tabelas referenciadas expõem unicidade `(user_id, id)`. Isso evita vínculos cruzados mesmo se uma validação de serviço falhar. Tipo de categoria compatível é regra do serviço, não um CHECK que consulta outra tabela.

Cada dono modifica suas entidades/migrations. A equipe usa um único histórico de migrations; alterações no schema compartilhado entram por PRs curtos e sequenciais. Antes de integrar, atualizar a branch e validar a sequência em banco vazio. Não editar uma migration já aplicada por colegas; criar nova migration corretiva.

## Contratos de API

- Prefixo `/api/v1`; recursos no plural em inglês; JSON em camelCase; tabelas e colunas em snake_case.
- UUIDs; dinheiro como string decimal; datas e paginação com formato único.
- GET de lista retorna `{ "items": [], "page": 1, "pageSize": 20, "total": 0 }`; `pageSize` máximo 100. GET de detalhe retorna o objeto documentado; não há envelope adicional obrigatório.
- Filtros: `startDate`, `endDate`, `accountId`, `categoryId`, `type`. Período é inclusivo. Ordenação padrão de lançamentos: data decrescente e ID decrescente.
- Erro: `{ "code": "VALIDATION_ERROR", "message": "...", "fieldErrors": {} }`.
- 201 para criação, 200 para consultas/edições, 204 para exclusão/logout; 400 para entrada inválida, 401 para falta de sessão, 404 para recurso inexistente ou de outro usuário, 409 para duplicidade/conflito de negócio.
- A API deriva `userId` da sessão e não aceita proprietário no corpo da requisição.
- Somente cadastro e login são públicos; os demais endpoints de negócio exigem sessão. A proteção deve ser padrão global, com exceções explícitas para essas duas operações.
- Validar na API todos os dados e regras, independentemente da validação da interface.
- Sessão opaca armazenada no servidor; cookie HttpOnly, Secure em HTTPS e SameSite; validar origem e aceitar somente os formatos previstos nas requisições mutáveis. Frontend e API publicados sob a mesma origem para simplificar cookies e integração. Duração de sessão definida coletivamente e coberta por teste.
- Publicar OpenAPI com exemplos de sucesso e erro. O NestJS fornece integração para gerar essa documentação: [OpenAPI no NestJS](https://docs.nestjs.com/openapi/introduction).
- Gerar tipos/cliente a partir do OpenAPI; não importar entidades do ORM no frontend. I5 mantém o comando de geração, e cada dono atualiza seus contratos.

## Endpoints do MVP

Todos abaixo recebem o prefixo `/api/v1`. Arquivamento é atualização por PATCH; GETs de listagem não exigem endpoints separados para cada filtro.

| Dono | Endpoints |
|---|---|
| I1 — 12 | `POST /auth/register`; `POST /auth/login`; `POST /auth/logout`; `GET /me`; `PATCH /me`; `GET /accounts`; `POST /accounts`; `PATCH /accounts/:id`; `GET /accounts/:id/statement`; `GET /balances`; `GET /transfers`; `POST /transfers` |
| I2 — 7 | `GET /categories`; `POST /categories`; `PATCH /categories/:id`; `GET /transactions`; `POST /transactions`; `PATCH /transactions/:id`; `DELETE /transactions/:id` |
| I3 — 6 | `GET /budgets`; `POST /budgets`; `PATCH /budgets/:id`; `DELETE /budgets/:id`; `GET /budgets/progress`; `GET /reports/expenses-by-category` |
| I4 — 7 | `GET /goals`; `POST /goals`; `GET /goals/:id`; `PATCH /goals/:id`; `GET /goals/:id/contributions`; `POST /goals/:id/contributions`; `DELETE /goals/:id/contributions/:contributionId` |
| I5 — 6 | `GET /recurring-rules`; `POST /recurring-rules`; `PATCH /recurring-rules/:id`; `GET /recurring-occurrences`; `POST /recurring-rules/:id/occurrences/:date/confirm`; `GET /dashboard` |

## Responsabilidades coletivas antes de implementar

Na primeira semana, registrar um documento curto de decisões com:

1. Escopo obrigatório, adiamentos e capacidade semanal real.
2. Stack, versões, scripts, configuração local, seed e publicação.
3. Modelo de dados, propriedade das entidades e política de migrations.
4. Sessão, guard, isolamento por usuário e comportamento dos erros.
5. Regras de saldo, datas, moeda, arquivamento, metas e recorrências.
6. Contratos de serviço, DTOs, endpoints e exemplos de respostas.
7. Componentes visuais mínimos, navegação, formulários, acessibilidade básica e estados de interface.
8. Convenções de branches, revisão, testes e definição de pronto.

Não adiar essas decisões para cada integrante resolver isoladamente. Após essa base, cada dono decide detalhes internos que não alterem contratos públicos.

## Desenvolvimento paralelo, Git e prevenção de conflitos

- `main` deve permanecer demonstrável; branches curtas como `feat/E2-02-create-transaction`.
- Cada PR entrega uma história ou uma fatia demonstrável; evitar uma branch por integrante durante todo o semestre.
- Um colega revisa cada PR. Em contrato compartilhado, revisar também com o consumidor afetado.
- Donos editam seus módulos. Rotas centrais, configuração e schema compartilhado têm uma pessoa integradora por alteração; o papel roda entre integrantes.
- Na fundação, I1 coordena autenticação e navegação; I2, schema e seed; I3, CI e ambiente; I4, componentes comuns; I5, cliente OpenAPI e mocks. São tarefas temporárias de integração. Cada integrante continua responsável por um módulo fullstack.
- Publicar exemplos de contrato e mocks já na fundação. Usar dois usuários e contas/categorias de teste para desenvolvimento independente. Guard simulado fica somente em testes/desenvolvimento e não pode ser ativado no deploy.
- Integrar primeiro as consultas compartilhadas mínimas e depois os refinamentos visuais. Mudança incompatível de contrato exige combinar alteração e atualização dos consumidores no mesmo pacote de integração.
- CI: instalação reproduzível, formatação/lint, verificação de tipos, build e testes; incluir teste de migrations em PostgreSQL.
- Cada dono testa suas regras; o colega verifica o fluxo afetado. Para fechamento, executar testes HTTP com banco real e testes de navegador nos fluxos críticos.
