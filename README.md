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

## Pastas

| Pasta | Controller | Observação |
| --- | --- | --- |
| `AUTH` | `LoginController` | Login e troca de empresa. |
| `CLIENTES` | `PessoaController` | Rotas `/pessoas`. |
| `FORNECEDORES` | `FornecedorController` | Mesma entidade de `CLIENTES` (`cad_pessoas`), rotas `/fornecedores`. |
| `CONTAS BANCARIAS` | `ContaBancariaController` | Contas bancárias da própria empresa (não tem relação com Open Finance). |
| `OPEN FINANCE` | `OpenFinanceController` | Integração com o parceiro Open Finance — ver `codigo/INTEGRACAO_OPENFINANCE.md` no repositório da API. |
| `WEBHOOK` | `WebhookController` | Não exige Bearer. |
| `SISTEMA` | `PingController`/`OpenApiController` (SDK) | Health-check e spec OpenAPI. |

Os endpoints de listagem (`LISTAR CLIENTES`, `LISTAR FORNECEDORES`, `LISTAR CONTAS BANCARIAS`)
aceitam query params opcionais que não foram fixados no arquivo (adicione manualmente na aba
Params do Bruno quando precisar filtrar/paginar):

- **CLIENTES/FORNECEDORES**: `pagina`, `limite_pagina`, `ordenacao`, `tipo_ordenacao`,
  `nome_pessoa`, `documento`, `email`, `situacao`.
- **CONTAS BANCARIAS**: `pagina`, `limite_pagina`, `ordenacao`, `tipo_ordenacao`, `nome_banco`.
- **OPEN FINANCE > LISTAR MOVIMENTOS**: `page`, `pageLimit` (nomes ainda não confirmados
  oficialmente pelo parceiro — ver "Em aberto" em `INTEGRACAO_OPENFINANCE.md`).

## Não incluído

As rotas `ExemploController`/`PlaygroundController` (scaffolding de demonstração do template do
`serverless-sdk`, não fazem parte da superfície real de negócio da API) não foram documentadas
aqui. Peça se precisar delas também.
