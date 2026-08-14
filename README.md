# sistema-controle-despesas-e2e

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
└── sistema-controle-despesas-e2e/   (este repo)
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
cd ../sistema-controle-despesas-e2e
docker compose down -v
```

## CI

O workflow [`.github/workflows/e2e.yml`](.github/workflows/e2e.yml):

1. Faz checkout deste repositório e do front (numa subpasta `front/`).
2. Gera um `JWT_SECRET` efêmero e sobe a stack (`docker compose up -d --build --wait`), com `FRONT_CONTEXT=./front`.
3. Instala as dependências do front e roda `npx cypress run` contra `http://localhost:3000`.
4. Derruba a stack ao final (`always()`), publicando os logs dos containers se algo falhar.

Disparado em push/PR para `main` e manualmente via `workflow_dispatch`.

## Variáveis de ambiente

| Variável | Descrição |
|---|---|
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | Credenciais do Postgres efêmero da stack. Default: `postgres`/`postgres`/`sistema_despesas_e2e`. |
| `POSTGRES_PORT` | Porta exposta no host (default `5432` — mude se já tiver um Postgres local rodando nela). |
| `JWT_SECRET` | **Obrigatória.** Mínimo 32 caracteres — a API não sobe sem ela. |
| `API_IMAGE_TAG` | Tag da imagem da API a puxar do GHCR (default `latest`). |
| `FRONT_CONTEXT` | Caminho do código-fonte do front pro build (default `../sistema-controle-despesas-front`; no CI vira `./front`). |
