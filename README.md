# api-docs-bpo-api

Coleção Bruno para a `bpo-api`, no mesmo padrão de `api-docs-upvendaspdv`/`api-docs-upvendascobranca`/`api-docs-upvendaslimite`.

## Como usar

1. Abra este diretório como uma coleção no Bruno (`Open Collection`).
2. Selecione o ambiente (`Desenvolvimento`, `Homologação` ou `Produção`) no seletor de ambiente.
3. Rode **AUTH > AUTENTICAÇÃO** informando `dominio`/`usuario`/`senha` reais — o script
   `after-response` da própria requisição captura o header `Authorization` da resposta e salva
   na variável de ambiente `token_auth`. As demais requisições autenticadas já usam
   `{{token_auth}}` automaticamente.
4. Se precisar trocar a empresa ativa do usuário logado, use **AUTH > TROCAR DE EMPRESA** — ela
   também atualiza `token_auth` com o novo token emitido.

> **Nota**: o header `Authorization` da resposta vem no formato `Bearer <token>`. Como a auth do
> tipo `bearer` do Bruno já adiciona o prefixo `Bearer ` sozinha ao montar o header de saída, o
> script de `after-response` salva em `token_auth` só o token limpo
> (`authHeader.replace(/^Bearer\s+/i, "")`) — diferente do `api-docs-upvendaspdv`, que salva o
> header cru. Sem essa limpeza o resultado sai como `Bearer Bearer <token>` (JWT com espaço no
> meio) e toda chamada autenticada falha com `Token invalido ou expirado: Compact JWT strings may
> not contain whitespace.` — foi exatamente esse erro que apareceu ao testar `TROCAR DE EMPRESA`
> antes dessa correção.

## Variáveis de ambiente

- `url_servidor`: usada por padrão em todas as requisições — muda conforme o ambiente
  selecionado (`ms.dev.upvendas.app`/`ms.hml.upvendas.app`/`ms.upvendas.app`).
- `url_local`: apontando para `http://127.0.0.1:8080/bpo-api` (porta padrão do `run-local.sh`) —
  troque manualmente o prefixo de uma requisição para `{{url_local}}` se quiser testar contra a
  API rodando na sua máquina em vez do servidor remoto.
- `token_auth`: secreta, populada automaticamente pelo script de AUTH — não precisa editar à mão.
- `openfinance_webhook_secret`: secreta, preencha manualmente com o mesmo valor configurado como
  `OPENFINANCE_BPO_WEBHOOK_SECRET` na API para o ambiente selecionado — usada em
  `OPEN FINANCE > CADASTRAR NOTIFICACAO` (campo `segredoWebhook`) e em
  `OPEN FINANCE > RECEBER WEBHOOK OPEN FINANCE` (header `x-openfinance-webhook-secret`, simulando o
  parceiro).

## Pastas

| Pasta | Controller | Observação |
| --- | --- | --- |
| `AUTH` | `LoginController` | Login e troca de empresa. |
| `CLIENTES` | `PessoaController`/`CidadeController` | Rotas `/pessoas`. `LISTAR CIDADES` é utilitário sem paginação/filtros — lista `idcidade`/`cidade`/`estado` do catálogo `cad_cidades` para popular o select de cidade ao cadastrar um endereço (dentro de `PessoaDto.enderecos`, usado tanto em `CLIENTES` quanto em `FORNECEDORES`). Não existem rotas separadas de ativar/inativar — o campo `ativo` é atualizado junto com o resto dos dados via `EDITAR CLIENTE`. |
| `FORNECEDORES` | `FornecedorController` | Mesma entidade de `CLIENTES` (`cad_pessoas`), rotas `/fornecedores`. Não existem rotas separadas de ativar/inativar — o campo `ativo` é atualizado junto com o resto dos dados via `EDITAR FORNECEDOR`. |
| `CONTAS BANCARIAS` | `ContaBancariaController`/`BancoController` | Contas bancárias da própria empresa (não tem relação com Open Finance). `LISTAR BANCOS` é utilitário sem paginação/filtros — lista `idbanco`/`nome` do catálogo `cad_bancos` para popular o select de banco na criação de conta bancária. |
| `CAIXA E BANCOS` | `CaixaController` | Tela de extrato bancário multi-contas. `RESUMO DAS CONTAS` (`/caixa/contas-resumo`) lista **todas** as contas bancárias do domínio com o saldo atual de cada uma — inclusive contas sem Open Finance (ex.: um caixa interno em espécie), cujo saldo é calculado somando os lançamentos de `movto_caixa` em vez do extrato do parceiro. `LISTAR MOVIMENTOS` (`/caixa/movimentos`) pagina os lançamentos individuais. `EXTRATO RESUMO` (`/caixa/extrato-resumo`) calcula os totais do período (saldo anterior, entradas, saídas, saldo do período, saldo final) a partir de `movto_caixa` — `data_de`/`data_ate` são obrigatórios (`AAAA-MM-DD`); `idcontabancaria` é opcional e aceita uma lista separada por vírgula (ex.: `1,2,3`) para restringir o cálculo às contas marcadas na tela — se omitido, soma todas as contas ativas do domínio. |
| `OPEN FINANCE` | `OpenFinanceController`/`OpenFinanceWebhookController`/`OpenFinanceConciliacaoController` | Integração com o parceiro Open Finance — pagadores, contas/consentimento, extratos/movimentos, cartões, o ciclo de webhook de saída (cadastrar/consultar/desativar inscrição + receber evento) e a conciliação bancária disparada por esse webhook: `LISTAR CONCILIACAO` (situações CONFERE/DIVERGENTE/SO_BANCO/SO_CAIXA — CONFERE e SO_BANCO são resolvidos automaticamente pelo próprio webhook, sem ação manual) e as ações dos botões da tela para os casos pendentes — `ATUALIZAR DIVERGENCIA` (banco vira fonte de verdade, id no **body**) e `REMOVER SO NO CAIXA` (soft-delete, id no path, DELETE). Ver `.claude/plans/2026-08-06-integracao-openfinance.md` e `.claude/plans/2026-08-18-conciliacao-openfinance-movimentos.md` no repositório da API. |
| `WEBHOOK` | `WebhookController` | Não exige Bearer. |
| `SISTEMA` | `PingController`/`OpenApiController` (SDK) | Health-check e spec OpenAPI. |
| `ITENS DE PAGAMENTO` | `ItemPagamentoController` | Categorias de pagar/receber usadas na conciliação de caixa (`cad_itens_pagamento`), rotas `/itens-pagamento`. Vínculo com plano de contas fica para uma etapa futura — por enquanto o cadastro cobre `descricao`/`tipo`/`pagar`/`receber`. `tipo` aceita só `"Fixa"` ou `"Variável"`. Não existem rotas separadas de ativar/inativar — o campo `ativo` é atualizado junto com o resto dos dados via `EDITAR ITEM DE PAGAMENTO`. |
| `CLASSIFICAÇÃO` | `ClassificacaoController` | Tela de classificação de movimentos de caixa, com **rateio**: um lançamento pode ser dividido em N linhas de plano financeiro. `LISTAR PENDENTES DE CLASSIFICACAO` (`/classificacao/movimentos`) traz os lançamentos de `movto_caixa` cujas linhas de rateio ainda não somam o valor do movimento e que não estão marcados como "não gerar financeiro" (mesmos filtros de `CAIXA E BANCOS > LISTAR MOVIMENTOS`), já com `idpessoa`/nome do cliente quando o lançamento já tiver esse vínculo (vincular pessoa não é feito por aqui) e `valorClassificado` (soma já lançada, para a UI mostrar "Distribuído: X de Y"). `LISTAR LINHAS DE CLASSIFICACAO` (`/classificacao/movimentos/:idmovtocaixa/linhas`, GET) retorna as linhas já salvas, para popular o modal ao reabrir um lançamento parcialmente classificado. `CLASSIFICAR MOVIMENTO` (`/classificacao/movimentos/:idmovtocaixa`, PUT) substitui **todas** as linhas do lançamento (delete+insert atômico) — body `{ naoGerarFinanceiro, linhas: [{idplanofinanceiro, iditempagamento (opcional), valor}, ...] }`; se `naoGerarFinanceiro=true` as `linhas` devem vir vazias/omitidas; senão a soma de `linhas[].valor` precisa fechar exatamente com o valor do movimento (a API valida e retorna 422 se não bater). |
| `PLANOS FINANCEIROS` | `PlanoFinanceiroController` | Leitura de `contabil_planofinanceiro` (plano de contas legado, fora do padrão `cad_*`, gerido por outro módulo da plataforma) para popular o seletor de plano financeiro em `CLASSIFICAÇÃO > CLASSIFICAR MOVIMENTO`. Só `LISTAR`/`DETALHAR` — sem cadastrar/editar, essa tabela não é gerida pela bpo-api. `analitica` filtra só as contas "folha" (as que fazem sentido atribuir a um lançamento, em vez dos grupos sintéticos). |

**`OPEN FINANCE > RECEBER WEBHOOK OPEN FINANCE`** não exige Bearer (`auth: none`) — quem chama essa
rota é o parceiro, não um usuário logado da bpo-api. É autenticada pelo header
`x-openfinance-webhook-secret`, comparado ao valor cadastrado em `CADASTRAR NOTIFICACAO`. O corpo
do arquivo traz o payload real confirmado pelo parceiro para `extrato.concluido` (sucesso, tipo
`BANK`) — o formato completo dos dois eventos (`conta.consentimento_atualizado` e
`extrato.concluido`, cada um com variação de sucesso/falha) está documentado no schema/exemplos
OpenAPI de `OpenFinanceWebhookController` (`OpenFinanceWebhookEventoDto`) no repositório da API.
Para simular os outros 3 casos, troque o corpo por:

- `conta.consentimento_atualizado` (sucesso/autorizado): `{"evento": "conta.consentimento_atualizado", "idConta": 3, "idPagador": 7, "banco": "341", "nomeBanco": "Itaú Unibanco", "agencia": "1234", "agenciaDigito": "5", "numero": "987654", "numeroDigito": "1", "linkConsentimento": "https://consentimento.tecnospeed.com.br/abc123", "idConsentimentoFornecedor": "cons_abc123", "statusConsentimento": "autorizado"}` — `statusConsentimento` também pode vir `revogado`, `pendente` ou `erro` (não é booleano).
- `extrato.concluido` (falha): mesmo shape do corpo do arquivo, com `"status": "falha"`, `"motivoFalha"` preenchido e `totalMovimentos`/saldos/`dataConclusao` como `null`.
- `extrato.concluido` com `tipoStatement: "CREDIT_CARD"`: `cartaoUltimosDigitos`/`creditCardLimiteDisponivelAtual`/`creditCardLimiteTotalAtual` vêm preenchidos e `saldoInicial`/`saldoFinal` ficam `null`.

Ver `.claude/plans/2026-08-06-integracao-openfinance.md` e
`.claude/plans/2026-08-17-alinhar-webhook-openfinance.md` no repositório da API para o contrato
completo (o antigo `codigo/INTEGRACAO_OPENFINANCE.md` foi migrado para o sistema de planos).

Os endpoints de listagem (`LISTAR CLIENTES`, `LISTAR FORNECEDORES`, `LISTAR CONTAS BANCARIAS`,
`LISTAR ITENS DE PAGAMENTO`) aceitam query params opcionais que não foram fixados no arquivo
(adicione manualmente na aba Params do Bruno quando precisar filtrar/paginar):

- **CLIENTES/FORNECEDORES**: `pagina`, `limite_pagina`, `ordenacao`, `tipo_ordenacao`,
  `nome_pessoa`, `documento`, `email`, `situacao`.
- **CONTAS BANCARIAS**: `pagina`, `limite_pagina`, `ordenacao`, `tipo_ordenacao`, `nome_banco`.
- **ITENS DE PAGAMENTO**: `pagina`, `limite_pagina`, `ordenacao` (`iditempagamento`/`descricao`/
  `tipo`), `tipo_ordenacao`, `descricao`, `tipo` (`Fixa`/`Variável`), `pagar`, `receber`,
  `situacao`.
- **OPEN FINANCE > LISTAR MOVIMENTOS**: `page`, `pageLimit` (nomes ainda não confirmados
  oficialmente pelo parceiro — ver "Passo a passo" em
  `.claude/plans/2026-08-06-integracao-openfinance.md`).

## Não incluído

As rotas `ExemploController`/`PlaygroundController` (scaffolding de demonstração do template do
`serverless-sdk`, não fazem parte da superfície real de negócio da API) não foram documentadas
aqui. Peça se precisar delas também.
