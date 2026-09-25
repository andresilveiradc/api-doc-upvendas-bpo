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
| `CAIXA E BANCOS` | `CaixaController` | Tela de extrato bancário multi-contas. `RESUMO DAS CONTAS` (`/caixa/contas-resumo`) lista **todas** as contas bancárias do domínio com o saldo atual de cada uma — inclusive contas sem Open Finance (ex.: um caixa interno em espécie), cujo saldo é calculado somando os lançamentos de `movto_caixa` em vez do extrato do parceiro. `LISTAR MOVIMENTOS` (`/caixa/movimentos`) pagina os lançamentos individuais. `EXTRATO RESUMO` (`/caixa/extrato-resumo`) calcula os totais do período (saldo anterior, entradas, saídas, saldo do período, saldo final) a partir de `movto_caixa` — `data_de`/`data_ate` são obrigatórios (`AAAA-MM-DD`); `idcontabancaria` é opcional e aceita uma lista separada por vírgula (ex.: `1,2,3`) para restringir o cálculo às contas marcadas na tela — se omitido, soma todas as contas ativas do domínio. Existe uma rota única de exportação, `/caixa/movimentos/exportar`, que aceita os mesmos filtros de `LISTAR MOVIMENTOS` (sem paginação) mais um parâmetro `formato` (`csv`, `pdf` ou `ofx`) que decide o arquivo devolvido — documentada aqui em 3 requisições separadas (`EXPORTAR CSV`/`EXPORTAR PDF`/`EXPORTAR OFX`) só por conveniência, é a mesma URL com `formato` diferente. **Nenhuma delas devolve JSON**: a resposta vem com `Content-Type`/`Content-Disposition: attachment` do formato pedido (`text/csv`, `application/pdf`, `application/x-ofx`), então no Bruno abra a aba "Response" em modo raw/preview/download para ver o conteúdo — não vai aparecer formatado como JSON. Detalhes por formato: **`EXPORTAR CSV`** — `;` como separador (padrão Excel BR), vírgula como separador decimal, BOM UTF-8 no início (acentuação correta ao abrir no Excel), `idcontabancaria` opcional (pode misturar contas, como a tela normal). **`EXPORTAR PDF`** — tabela simples (uma linha por movimento, texto truncado por coluna, sem estilo/logo), `idcontabancaria` opcional. **`EXPORTAR OFX`** — formato OFX 1.02 SGML; **`idcontabancaria` é obrigatório só neste formato** (um extrato OFX representa uma única conta, diferente de CSV/PDF); sem `LEDGERBAL` (saldo), só a lista de transações; `FITID` reaproveita `movto_caixa.fitid` do lançamento original quando existe (vindo do Open Finance) ou usa o fallback `MC{idmovtocaixa}` para lançamentos manuais. |
| `OPEN FINANCE` | `OpenFinanceController`/`OpenFinanceWebhookController`/`OpenFinanceConciliacaoController` | Integração com o parceiro Open Finance — pagadores, contas/consentimento, extratos/movimentos, cartões, o ciclo de webhook de saída (cadastrar/consultar/desativar inscrição + receber evento) e a conciliação bancária disparada por esse webhook: `LISTAR CONCILIACAO` (situações CONFERE/DIVERGENTE/SO_BANCO/SO_CAIXA — CONFERE e SO_BANCO são resolvidos automaticamente pelo próprio webhook, sem ação manual) e as ações dos botões da tela para os casos pendentes — `ATUALIZAR DIVERGENCIA` (banco vira fonte de verdade, id no **body**) e `REMOVER SO NO CAIXA` (soft-delete, id no path, DELETE). Ver `.claude/plans/2026-08-06-integracao-openfinance.md` e `.claude/plans/2026-08-18-conciliacao-openfinance-movimentos.md` no repositório da API. |
| `WEBHOOK` | `WebhookController` | Não exige Bearer. |
| `SISTEMA` | `PingController`/`OpenApiController` (SDK) | Health-check e spec OpenAPI. |
| `ITENS DE PAGAMENTO` | `ItemPagamentoController` | Categorias de pagar/receber usadas na conciliação de caixa (`cad_itens_pagamento`), rotas `/itens-pagamento`. Vínculo com plano de contas fica para uma etapa futura — por enquanto o cadastro cobre `descricao`/`tipo`/`pagar`/`receber`. `tipo` aceita só `"Fixa"` ou `"Variável"`. Não existem rotas separadas de ativar/inativar — o campo `ativo` é atualizado junto com o resto dos dados via `EDITAR ITEM DE PAGAMENTO`. |
| `CLASSIFICAÇÃO` | `ClassificacaoController` | Tela de classificação de movimentos de caixa, com **rateio**: um lançamento pode ser dividido em N linhas de plano financeiro. `LISTAR PENDENTES DE CLASSIFICACAO` (`/classificacao/movimentos`) traz os lançamentos de `movto_caixa` cujas linhas de rateio ainda não somam o valor do movimento e que não estão marcados como "não gerar financeiro" (mesmos filtros de `CAIXA E BANCOS > LISTAR MOVIMENTOS`), já com `idpessoa`/nome do cliente quando o lançamento já tiver esse vínculo (vincular pessoa não é feito por aqui) `valorClassificado` (soma já lançada, para a UI mostrar "Distribuído: X de Y") e `iditempagamento`/`itemPagamento` (o item de pagamento legado já vinculado ao lançamento pelo fluxo automático do Open Finance, quando houver — a tela mostra esse valor pré-selecionado e o usuário pode trocar; é independente do item de pagamento de cada linha de rateio). `LISTAR LINHAS DE CLASSIFICACAO` (`/classificacao/movimentos/:idmovtocaixa/linhas`, GET) retorna as linhas já salvas, para popular o modal ao reabrir um lançamento parcialmente classificado. `CLASSIFICAR MOVIMENTO` (`/classificacao/movimentos/:idmovtocaixa`, PUT) substitui **todas** as linhas do lançamento (delete+insert atômico) — body `{ naoGerarFinanceiro, linhas: [{idplanofinanceiro, iditempagamento (opcional), valor}, ...] }`; se `naoGerarFinanceiro=true` as `linhas` devem vir vazias/omitidas; senão a soma de `linhas[].valor` precisa fechar exatamente com o valor do movimento (a API valida e retorna 422 se não bater). |
| `PLANOS FINANCEIROS` | `PlanoFinanceiroController` | Leitura de `contabil_planofinanceiro` (plano de contas legado, fora do padrão `cad_*`, gerido por outro módulo da plataforma) para popular o seletor de plano financeiro em `CLASSIFICAÇÃO > CLASSIFICAR MOVIMENTO`. Só `LISTAR`/`DETALHAR` — sem cadastrar/editar, essa tabela não é gerida pela bpo-api. `analitica` filtra só as contas "folha" (as que fazem sentido atribuir a um lançamento, em vez dos grupos sintéticos). |
| `PLANOS DE CONTAS FINANCEIROS` | `PlanoContasFinanceiroController` | Cadastro de `contabil_planofinanceiro_nome` — o "recipiente" que agrupa as contas de `contabil_planofinanceiro` pela coluna `idplanonome`. Uma empresa pode ter vários planos (ex.: "Plano 1", "Plano 2026"), e toda conta pertence a exatamente um. A listagem traz os planos da própria empresa **e** os planos `compartilhado = true` do mesmo domínio (inclusive de outras empresas), por isso o campo `empresa` vem em cada item. No cadastro, o primeiro plano da empresa nasce com `vigencia = true` e os demais com `false`; na edição, colocar um plano em vigência tira a vigência de todos os outros da empresa, na mesma transação — diferente do ERP, onde é possível ficar com dois planos vigentes ao mesmo tempo. A exclusão é bloqueada quando o plano já tem qualquer conta cadastrada (o ERP apaga as contas em cascata; aqui não). |
| `CONTAS DO PLANO FINANCEIRO` | `ContaPlanoFinanceiroController` | As contas de um plano (`contabil_planofinanceiro`), sob `/planos-contas-financeiros/:idplano/contas`. A listagem vem **sem paginação** e ordenada por `codigoredutor`, porque o front monta a árvore a partir de `idplanopai` — paginar quebraria a hierarquia. O filtro `busca` bate em código e descrição ao mesmo tempo (o campo único da tela). `codigoredutor` é **gerado pela API** a partir da conta pai, copiando dos irmãos quantos dígitos o último bloco usa (`1.1.3` → `1.1.4`, `1.1.1.01.001` → `1.1.1.01.002`), e **não é editável** — não vai no payload nem do POST nem do PUT. `limite` só é aceito em conta analítica. Não dá para criar conta abaixo de uma conta analítica (`analitica = true` não pode ter filhos). A exclusão é bloqueada em três casos: a conta tem filhos, tem relação contábil (`contabil_planofinanceiro_relacaocontabil`) ou tem movimentos classificados (`movto_caixa_classificacao` — essa última trava não existe no ERP). |
| `SELECTS DO PLANO FINANCEIRO` | `OpcoesPlanoFinanceiroController` | Os três combos da tela de conta. `LISTAR TIPOS DE CONTA` (`/tipos-conta`) não consulta banco — é a lista fixa Analítica/Sintética, e o que vai no payload da conta é o boolean `analitica`. `LISTAR TIPOS` (`/tipos-plano-financeiro`) lê `contabil_planofinanceiro_tipo` (Receita, Despesa, Transferência, Investimento). `LISTAR GRUPOS DE CONTAS DRE` (`/grupos-contas-dre`) lê `contabil_planofinanceiro_grupodecontas` ordenado por `ordem`, que é a sequência do relatório, não alfabética. As duas tabelas são globais dentro do banco do domínio — não têm `idempresa`/`iddominio`. Conta sem grupo de DRE simplesmente não entra no relatório. `LISTAR CONTAS FINANCEIRAS` (`/listar-contas-financeiras`) é a exceção da pasta — não recebe `:idplano` — e serve o combo de plano de contas da tela de **Classificação**, que não tem onde escolher plano: ela resolve sozinha o plano vigente da empresa (o de menor `idplano`, se houver mais de um vigente) e devolve só as contas `analitica = true`, que são as únicas que aceitam lançamento. Sem parâmetro nenhum, como `LISTAR CIDADES`/`LISTAR BANCOS` (`CLIENTES`/`CONTAS BANCARIAS`) — devolve sempre a lista completa, e o filtro é feito no front. O mesmo raciocínio vale para `LISTAR TIPOS DE CONTA` e `LISTAR TIPOS` desta pasta: também alimentam combos usados pela tela de Classificação, não só pelo cadastro de plano financeiro. Substitui o uso que a Classificação faz hoje de `PLANOS FINANCEIROS > LISTAR`. |

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
