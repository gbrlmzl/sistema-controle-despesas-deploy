# sistema-controle-despesas-deploy

Repositório orquestrador dos testes end-to-end do [CRONOS](https://github.com/gbrlmzl/sistema-controle-despesas-front). Não contém código de aplicação nem specs de teste — só o `docker-compose.yml` que sobe o sistema completo (Postgres + API + front) e o workflow que roda os testes Cypress contra essa stack.

Os specs em si continuam vivendo no repo do front ([`sistema-controle-despesas-front/cypress`](https://github.com/gbrlmzl/sistema-controle-despesas-front/tree/main/cypress)) — colocados junto do código que exercitam. Este repositório só os invoca contra o ambiente que ele monta.

## Por que um repositório separado

- **Responsabilidade correta por repo**: o front testa o front (unitário/lint/build) na própria pipeline; a API testa a API na dela; aqui é o único lugar que testa o sistema como um todo, integrado.
- **Não acopla a pipeline de nenhum dos dois repos ao outro**: uma mudança na API não derruba o CI do front por causa de um teste e2e, e vice-versa.
- **Testa os artefatos reais**: a API roda a partir da imagem publicada no GHCR pelo próprio CI dela — não é rebuildada aqui a partir do código-fonte.

## Arquitetura

```
┌──────────┐     ┌─────────────────────┐     ┌──────────┐
│   front  │────▶│         api         │────▶│ postgres │
│ (buildado│     │ (imagem do GHCR,    │     │          │
│ do fonte)│     │  já testada no CI   │     │          │
│  :3000   │     │  do próprio repo)   │     │          │
└──────────┘     │        :3001        │     └──────────┘
                  └─────────────────────┘
```

**Front é buildado a partir do código-fonte, não da imagem publicada.** O Next.js resolve o destino do rewrite `/api/*` em build-time (ver comentário no `Dockerfile` do front) — a imagem publicada no GHCR carrega um placeholder (`http://localhost:8080`) que não aponta pro serviço `api` desta rede. Buildar aqui, com `API_URL=http://api:3001`, é a única forma de o proxy funcionar de verdade dentro do compose.

**API roda a imagem publicada.** Ela não tem esse problema — não tem nada baked-in no build, só lê variáveis de ambiente em runtime.

## Rodando localmente

Pré-requisito: este repositório precisa estar clonado como **irmão** de `sistema-controle-despesas-front` (mesmo diretório pai) — é assim que o `docker-compose.yml` encontra o código-fonte do front pra buildar:

```
Projetos/
├── sistema-controle-despesas-front/
└── sistema-controle-despesas-deploy/   (este repo)
```

```bash
cp .env.example .env   # preencha JWT_SECRET (ver instrução no arquivo)
docker compose up -d --build --wait
```

Depois, rode os specs a partir do repo do front, apontando pro front orquestrado:

```bash
cd ../sistema-controle-despesas-front
npm ci
npx cypress run --config baseUrl=http://localhost:3000
```

Pra derrubar tudo:

```bash
cd ../sistema-controle-despesas-deploy
docker compose down -v
```

## CI

O workflow [`.github/workflows/ci.yml`](.github/workflows/ci.yml) segue o padrão **build once, promote everywhere**: nunca builda artefato de produção aqui — o front e a API já se buildam e se publicam sozinhos (GHCR) nos próprios CIs. Este repo só valida a combinação via e2e e, se passar, promove exatamente o artefato testado.

### Job `e2e`

1. Resolve os parâmetros do teste (`front_ref` e `api_image_tag`) a partir do tipo de disparo — ver tabela abaixo.
2. Faz checkout deste repositório e do front no ref resolvido (subpasta `front/`).
3. Faz login no GHCR — o pacote `sistema-controle-despesas-api` é **privado**, então sem login o `docker compose up` abaixo falha ao puxar a imagem (`unauthorized`). Usa o mesmo `GHCR_PROMOTE_TOKEN` do job `promote`.
4. Gera um `JWT_SECRET` efêmero e sobe a stack (`docker compose up -d --build --wait`), com `FRONT_CONTEXT=./front` e `API_IMAGE_TAG` resolvido.
5. Instala as dependências do front e roda `npx cypress run` contra `http://localhost:3000`.
6. Derruba a stack ao final (`always()`), publicando os logs dos containers se algo falhar.

### Job `promote`

Roda só quando o disparo foi um `repository_dispatch` (ou seja, quando existe um artefato específico recém-publicado a validar) e o `e2e` passou. Re-taggeia (`docker buildx imagetools create`, sem rebuild) a imagem testada como `:stable` no GHCR do repo correspondente (front ou API) — é o sinal de "este SHA passou no e2e do sistema completo".

### Secret `GHCR_PROMOTE_TOKEN`

Usado nos jobs `e2e` (pull da imagem privada da API) e `promote` (push da tag `:stable`). Precisa ser um PAT com `write:packages` sobre os pacotes de `sistema-controle-despesas-front` **e** `sistema-controle-despesas-api` (o escopo de escrita já cobre leitura — o `GITHUB_TOKEN` padrão não serve porque só tem permissão sobre pacotes deste repo).

### Formas de disparo

| Disparo | `front_ref` | `api_image_tag` | Quando usar |
|---|---|---|---|
| `push`/`pull_request` neste repo | `main` | `latest` | Mudou algo aqui (ex.: `docker-compose.yml`) |
| `repository_dispatch: front-published` | SHA recém-publicado (`client_payload.sha`) | `latest` | CI do front acabou de publicar uma imagem |
| `repository_dispatch: api-published` | `main` | tag recém-publicada (`client_payload.image_tag`) | CI da API acabou de publicar uma imagem |
| `workflow_dispatch` manual | input `front_ref` | input `api_image_tag` | Reproduzir/depurar um cenário específico |

Os CIs do front e da API disparam um `repository_dispatch` contra este repo (`gbrlmzl/sistema-controle-despesas-deploy`) logo depois de publicar no GHCR, enviando o payload esperado (`sha` + `image_tag` para o front; `image_tag` para a API). Isso exige um secret `DEPLOY_DISPATCH_TOKEN` (PAT com escopo `repo` sobre este repositório) configurado nos repos do front e da API.

## Variáveis de ambiente

| Variável | Descrição |
|---|---|
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | Credenciais do Postgres efêmero da stack. Default: `postgres`/`postgres`/`sistema_despesas_e2e`. |
| `POSTGRES_PORT` | Porta exposta no host (default `5432` — mude se já tiver um Postgres local rodando nela). |
| `JWT_SECRET` | **Obrigatória.** Mínimo 32 caracteres — a API não sobe sem ela. |
| `API_IMAGE_TAG` | Tag da imagem da API a puxar do GHCR (default `latest`). |
| `FRONT_CONTEXT` | Caminho do código-fonte do front pro build (default `../sistema-controle-despesas-front`; no CI vira `./front`). |
