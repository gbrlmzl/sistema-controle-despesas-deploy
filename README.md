# sistema-controle-despesas-deploy

Repositório orquestrador do pipeline de entrega do [CRONOS](https://github.com/gbrlmzl/sistema-controle-despesas-front). Não contém código de aplicação nem specs de teste. Tem só o `docker-compose.yml` que sobe o sistema completo (Postgres + API + front) e o workflow que roda os testes Cypress contra essa stack e, se passarem, **promove** o artefato, **espelha** no ECR e faz o **deploy** no ECS, com aprovação manual.

Os specs em si continuam vivendo no repo do front ([`sistema-controle-despesas-front/cypress`](https://github.com/gbrlmzl/sistema-controle-despesas-front/tree/main/cypress)) — colocados junto do código que exercitam. Este repositório só os invoca contra o ambiente que ele monta.

> Como os três pipelines funcionam juntos, job a job: [`docs/pipeline-ci-cd.md`](docs/pipeline-ci-cd.md). Como a automação até o ECS foi montada, com o status de cada etapa: [`docs/pipeline-aws.md`](docs/pipeline-aws.md).

## Por que um repositório separado

- **Responsabilidade correta por repo**: o front testa o front (unitário/lint/build) na própria pipeline; a API testa a API na dela; aqui é o único lugar que testa o sistema como um todo, integrado.
- **Não acopla a pipeline de nenhum dos dois repos ao outro**: uma mudança na API não derruba o CI do front por causa de um teste e2e, e vice-versa.
- **Testa os artefatos reais**: API e front rodam a partir das imagens publicadas no GHCR pelos próprios CIs. Nada é rebuildado aqui, e a imagem que passa no e2e é exatamente a que vai para produção.

## Arquitetura

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────┐
│      front       │────▶│       api        │────▶│ postgres │
│ (imagem do GHCR, │     │ (imagem do GHCR, │     │          │
│  multi-arch)     │     │  multi-arch)     │     │          │
│      :3000       │     │      :8080       │     │          │
└──────────────────┘     └──────────────────┘     └──────────┘
```

**As duas aplicações rodam a imagem publicada**, na tag resolvida pelo tipo de disparo (ver [Formas de disparo](#formas-de-disparo)). Nenhuma delas tem endereço embutido no build. A API lê tudo do ambiente em runtime, e o front resolve `API_URL` a cada requisição no Route Handler `src/app/api/[...path]/route.ts`. Aqui, `API_URL=http://api:8080` é só uma variável do serviço `front` no compose.

As duas imagens publicam **manifest multi-arch** (`linux/amd64` + `linux/arm64`). O arm64 é o que vai para a instância Graviton; o amd64 é o que o runner `ubuntu-latest` do e2e executa nativamente, sem QEMU.

> **Histórico:** até 11/09/2026 o front era buildado **a partir do código-fonte** neste compose, porque o rewrite `/api/*` do Next era resolvido em build-time e a imagem publicada carregava o `API_URL` do momento do build. Isso quebrou produção em 20/08/2026 (ver [`problema-rewrite-api-build-time.md`](../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md)). Com o Route Handler em runtime e a imagem do front publicada multi-arch, o build do fonte saiu. Desde então o front também segue o "build once, promote everywhere", e o artefato que o Cypress testa é o mesmo que o `promote` marca como `:stable`.

## Rodando localmente

```bash
cp .env.example .env   # preencha JWT_SECRET (ver instrução no arquivo)
docker compose up -d --wait
```

Isso sobe a stack completa com as imagens publicadas no GHCR (API e front), nas tags resolvidas por
`API_IMAGE_TAG`/`FRONT_IMAGE_TAG` do `.env`.

Para iterar no front sem publicar imagem, suba só o Postgres e a API aqui e rode o front direto do
código-fonte, no repositório dele:

```bash
docker compose up -d postgres api
```

```bash
cd ../sistema-controle-despesas-front
npm run dev
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

## CI/CD

O workflow [`.github/workflows/ci.yml`](.github/workflows/ci.yml) segue o padrão **build once, promote everywhere**: nunca builda artefato de produção aqui — o front e a API já se buildam e se publicam sozinhos (GHCR) nos próprios CIs. Este repo valida a combinação via e2e e, se passar, leva **exatamente o artefato testado** até o ECS.

```
e2e ──▶ promote ──▶ mirror ──▶ deploy (environment: production · aprovação manual)
        (GHCR :stable)  (ECR :<sha> + :stable)  (task definition nova · migration · update-service)
```

`promote`, `mirror` e `deploy` só rodam em `repository_dispatch`. Em `push`, `pull_request` e `workflow_dispatch` o pipeline para no `e2e`.

### Job `e2e`

1. Resolve os parâmetros do teste (`front_ref`, `front_image_tag` e `api_image_tag`) a partir do tipo de disparo — ver tabela abaixo.
2. Faz checkout deste repositório e do front no `front_ref` resolvido (subpasta `front/`), só para ter os specs.
3. Gera um `.env` efêmero com `JWT_SECRET` aleatório e as duas tags de imagem.
4. Faz login no GHCR — o pacote `sistema-controle-despesas-api` é **privado**, então sem login o `docker compose up` falha ao puxar a imagem (`unauthorized`). Usa o `GHCR_PROMOTE_TOKEN`.
5. Sobe a stack com `docker compose up -d --wait` (Postgres → migrate → API → front, esperando os healthchecks).
6. Instala as dependências do front e roda `npx cypress run` contra `http://localhost:3000`.
7. Se algo falhar, publica os logs dos containers e o artifact `cypress-failure-evidence` (screenshots e vídeos, 7 dias). Derruba a stack ao final (`always()`).

### Job `promote`

Roda só quando o disparo foi um `repository_dispatch` (ou seja, quando existe um artefato específico recém-publicado a validar) e o `e2e` passou. Re-taggeia (`docker buildx imagetools create`, sem rebuild) a imagem testada como `:stable` no GHCR — `api` **e** `api-migrate` num `api-published`, `front` num `front-published`. É o sinal de "este SHA passou no e2e do sistema completo". Não é o sinal de "este SHA está em produção": o `deploy` ainda pode ser rejeitado ou falhar.

### Job `mirror`

Assume a role `github-actions-mirror` por OIDC (só escrita no ECR, sem chave estática) e copia a imagem `:stable` do GHCR para o ECR com duas tags: a **imutável** (`<sha12>` na API, `sha-<sha>` no front), que é a que a task definition usa, e `:stable`, só como rótulo. A cópia é registry → registry e preserva o manifest multi-arch.

### Job `deploy`

Declara `environment: production`, então para em **Waiting for review** até alguém aprovar em *Review deployments*. Só depois disso a role `github-actions-deploy` (só ECS) pode ser assumida. Com a aprovação:

1. Baixa a task definition ativa da família (`cronos-app` ou `cronos-front`), troca só a imagem do container e registra uma revisão nova.
2. **Só na API:** deriva dessa revisão uma task `cronos-migrate`, com a mesma imagem e os mesmos secrets, mas sem `portMappings` e sem health check. Roda `npx prisma migrate deploy` com `run-task` e só segue com `exitCode` 0. A família separada existe por causa da porta: a `cronos-app` publica `hostPort: 8080`, que a API antiga ainda ocupa (ver pendência 9).
3. `update-service` + `wait services-stable`. Os services têm circuit breaker com rollback automático.

Um deploy por vez (`concurrency: deploy-production`, sem cancelar o que está em andamento). Rollback e diagnóstico: [`docs/pipeline-ci-cd.md`](docs/pipeline-ci-cd.md) §9.

O passo final continua **deliberadamente dependente de uma decisão humana**. Isso não mudou; mudou o mecanismo, que antes era um `update-service` digitado no terminal e agora é um botão de aprovação no GitHub.

### Secrets e configuração

| Nome | Tipo | Usado em |
|---|---|---|
| `GHCR_PROMOTE_TOKEN` | Repository secret — PAT com `write:packages` sobre os pacotes de `sistema-controle-despesas-front` **e** `sistema-controle-despesas-api` (o escopo de escrita já cobre leitura; o `GITHUB_TOKEN` padrão só enxerga pacotes deste repo) | `e2e` (pull da imagem privada da API), `promote` (push de `:stable`), `mirror` (leitura) |
| `AWS_ACCOUNT_ID` | Repository secret | `mirror` e `deploy`, para montar o ARN das roles sem expor o ID num repositório público |
| Environment `production` | Settings → Environments, com *required reviewers* e política de branch `main` | Portão do `deploy` |
| Provedor OIDC + roles `github-actions-mirror` / `github-actions-deploy` | IAM, na AWS | Credenciais temporárias dos jobs `mirror` e `deploy`. Policies em [`docs/pipeline-aws.md`](docs/pipeline-aws.md) Etapa 4 |

### Formas de disparo

| Disparo | `front_ref` (specs) | `front_image_tag` | `api_image_tag` | Promove e faz deploy? | Quando usar |
|---|---|---|---|:-:|---|
| `push`/`pull_request` neste repo | `main` | `stable` | `stable` | — | Mudou algo aqui (ex.: `docker-compose.yml`) |
| `repository_dispatch: front-published` | SHA recém-publicado (`client_payload.sha`) | `client_payload.image_tag` | `stable` | ✅ | CI do front acabou de publicar uma imagem |
| `repository_dispatch: api-published` | `main` | `stable` | `client_payload.image_tag` | ✅ | CI da API acabou de publicar uma imagem |
| `workflow_dispatch` manual | input (default `main`) | input (default `stable`) | input (default `stable`) | — | Reproduzir/depurar um cenário específico |

A regra é testar o artefato novo contra o **último artefato aprovado (`:stable`) do outro lado**, nunca contra um `:latest` que ninguém validou.

Os CIs do front e da API disparam um `repository_dispatch` contra este repo (`gbrlmzl/sistema-controle-despesas-deploy`) logo depois de publicar no GHCR, enviando o payload esperado (`sha` + `image_tag` para o front; `image_tag` para a API). Isso exige um secret `DEPLOY_DISPATCH_TOKEN` (PAT com escopo `repo` sobre este repositório) configurado nos repos do front e da API.

> **Re-executar um run usa o workflow daquele run.** Um *Re-run* roda com o `ci.yml` do commit original, então uma correção feita aqui depois não vale para ele. Para reaplicar o pipeline a uma imagem já publicada, dispare de novo o `repository_dispatch` com a mesma tag. O comando está em [`docs/pipeline-ci-cd.md`](docs/pipeline-ci-cd.md) §9.2.

## Variáveis de ambiente

| Variável | Descrição |
|---|---|
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | Credenciais do Postgres efêmero da stack. Default: `postgres`/`postgres`/`sistema_despesas_e2e`. |
| `POSTGRES_PORT` | Porta exposta no host (default `5432` — mude se já tiver um Postgres local rodando nela). |
| `JWT_SECRET` | **Obrigatória.** Mínimo 32 caracteres — a API não sobe sem ela. |
| `API_IMAGE_TAG` | Tag da imagem da API a puxar do GHCR (default `latest`; o CI usa `stable` ou a tag recém-publicada). |
| `FRONT_IMAGE_TAG` | Tag da imagem do front a puxar do GHCR (default `stable`). |

## Documentação da infra AWS

| Documento | Conteúdo |
|---|---|
| [`docs/arquitetura-aws.md`](docs/arquitetura-aws.md) | Decisões de arquitetura, custos, cenários e o **roteiro de fases** |
| [`docs/banco-de-dados-aws.md`](docs/banco-de-dados-aws.md) | Fase 3 — Postgres em container + volume EBS |
| [`docs/api-aws.md`](docs/api-aws.md) | Fase 4 — task definition da API, ECR, secrets no SSM |
| [`docs/ingresso-aws.md`](docs/ingresso-aws.md) | Fase 5 — Caddy, CloudFront, TLS, **e as pendências abertas** |
| [`docs/front-aws.md`](docs/front-aws.md) | Fase 6 — task definition do front, `extraHosts` |
| [`docs/separacao-de-tasks-front-api.md`](docs/separacao-de-tasks-front-api.md) | Por que front e API rodam em tasks separadas |
| [`docs/borda-cloudflare.md`](docs/borda-cloudflare.md) | Troca da borda de CloudFront para Cloudflare e domínio próprio |
| [`docs/pipeline-aws.md`](docs/pipeline-aws.md) | Fase 8 — automação GHCR → ECR → ECS, com o status de cada etapa |
| [`docs/pipeline-ci-cd.md`](docs/pipeline-ci-cd.md) | Como os pipelines dos três repositórios funcionam hoje, job a job, com diagrama, mapa de erros e rollback |
| [`docs/docker-e-arquitetura.md`](docs/docker-e-arquitetura.md) | O estado do Docker nos três repositórios (a parte de CI, §4.3, é anterior à Fase 8) |

## Pendências

Estado em 14/09/2026: **Fases 0 a 6 concluídas**, borda na Cloudflare e **Fase 8 em execução**. O
sistema está no ar em `https://cronos.gabrielmizael.com`, e o caminho antigo
`https://d3c5d6t3539m1d.cloudfront.net` continua respondendo em paralelo. Deploy automatizado do front
funcionando desde 12/09; o da API ainda não fechou (item 10). O que falta, em ordem de importância:

1. **Google OAuth e SMTP não têm variáveis configuradas em produção.** O código dos dois está pronto,
   mas inerte: o botão de login com Google não funciona, e a recuperação de senha **completa o fluxo
   sem nunca enviar o email**. O `env.ts` da API valida cada grupo como "tudo ou nada" — preencher
   pela metade **impede a API de subir**. Ambas dependem do domínio da Fase 7. Ver
   [`ingresso-aws.md`](docs/ingresso-aws.md) §11.8.
2. ~~**O rewrite `/api/*` do front continua resolvido em build-time.**~~ **Resolvido.** O front usa
   um Route Handler que lê `API_URL` em runtime, publica uma imagem só para qualquer ambiente, e é essa
   imagem que o e2e testa e o ECS roda (Etapa 1 de [`pipeline-aws.md`](docs/pipeline-aws.md)). Ver
   [`ingresso-aws.md`](docs/ingresso-aws.md) §11.9.
3. **Fase 7 — domínio próprio.** O domínio e a borda já existem: `cronos.gabrielmizael.com`, com DNS e
   proxy na Cloudflare e certificado Origin CA no Caddy (ver [`borda-cloudflare.md`](docs/borda-cloudflare.md)).
   Falta o que dependia do domínio: `FRONTEND_URL` deixar de ser placeholder e o item 1. Vale agrupar
   tudo numa revisão de task definition só.
4. ~~**Fase 8 — automatizar o espelhamento no ECR.**~~ **Automatizado em 11/09/2026.** Os jobs `mirror`
   e `deploy` levam API e front do GHCR ao ECS (ver [CI/CD](#cicd)). Só o Caddy continua espelhado à
   mão, sob demanda. O passo final continua exigindo decisão humana, agora como aprovação no environment
   `production` em vez de comando no terminal.
5. ~~**O espelhamento manual usa `:latest`, não `:stable`.**~~ **Resolvido com a Fase 8.** O `mirror`
   copia só o `:stable`, que o `promote` cria depois do e2e verde, e o e2e também passou a usar `:stable`
   do lado oposto em vez de `:latest`.
6. ~~**Padronizar a porta da API em `8080`.**~~ **Concluído em 21/08/2026** — código e infra AWS
   (Security Group, `cronos-app:2`, `cronos-front:3`) já usam `8080`; verificado ponta a ponta via SSM.
   Falta só a Repository Variable `API_URL` do repo do front, sem efeito no que está no ar. Ver
   [`ingresso-aws.md`](docs/ingresso-aws.md) §11.10.
7. **Sem WAF e sem rate limiting na borda** — decisão consciente de orçamento, registrada em
   [`ingresso-aws.md`](docs/ingresso-aws.md) §11.6. O CloudFront traz Shield Standard de graça (L3/L4),
   e o teto fixo de compute protege a fatura; nada protege contra sobrecarga da instância.
8. **O `Caddyfile` não sobrevive à substituição da instância** — criado à mão, não versionado. Mesma
   classe do mount do EBS; resolver as duas juntas num user-data. Ver
   [`ingresso-aws.md`](docs/ingresso-aws.md) §11.1.
9. **Todo deploy derruba a API (e o front) por alguns segundos.** A causa é a porta fixa no host:
   `cronos-app` publica `hostPort: 8080` (e `cronos-front`, `3000`), então a task nova não sobe enquanto
   a antiga segura a porta — por isso os services rodam com `minimumHealthyPercent: 0` /
   `maximumPercent: 100` ("derruba e sobe"). Memória não é o obstáculo: em 14/09/2026 a instância tinha
   630 MB livres, suficiente para uma segunda API (448 MiB) durante a troca. Foi a mesma porta fixa que
   obrigou a migration do pipeline a rodar numa família própria (`cronos-migrate`, sem `portMappings`).

   **Solução escolhida: Cloud Map + Caddy como roteador interno** (~US$ 0,50/mês da zona privada do
   Route 53). Descartados: ALB (~US$ 18/mês, desproporcional) e ECS Service Connect (um sidecar Envoy
   por task, sem memória sobrando na `t4g.small`).

   - [ ] **Porta dinâmica:** `hostPort: 0` na `cronos-app` e na `cronos-front`.
   - [ ] **Cloud Map:** namespace DNS privado (ex.: `cronos.local`) e service registry com registro
     **SRV** (obrigatório em `bridge`, carrega IP + porta) em cada service — `api.cronos.local` e
     `front.cronos.local`, com TTL baixo (~10s).
   - [ ] **Caddy:** assumir a `:8080` fixa e rotear para a API com `reverse_proxy { dynamic srv
     api.cronos.local { refresh 5s } lb_try_duration 10s fail_duration 30s }`; trocar o upstream
     `:3000` do front por `dynamic srv front.cronos.local`. O front continua chamando
     `http://172.17.0.1:8080` e a regra de Security Group da `8080` não muda. Aproveitar para versionar
     o `Caddyfile` (item 8).
   - [ ] **Deployment configuration:** `minimumHealthyPercent: 100` / `maximumPercent: 200` nos dois
     services — sobe a nova, espera o health check, só então drena a antiga.
   - [ ] **Graceful shutdown na API:** no `SIGTERM`, parar de aceitar conexões e concluir as em
     andamento (`app.close()` do Fastify), com `stopTimeout` do container compatível.
   - [ ] **Migrations expand/contract:** a migration roda antes do `update-service`, então a API antiga
     convive alguns segundos com o schema novo. Renomear/remover coluna vira dois deploys (adiciona →
     depois remove). Documentar a regra no repo da API.
   - [ ] **Workflow:** nada muda no fluxo (`update-service` + `wait services-stable` já é o certo). A
     `cronos-migrate` deixa de ser necessária pela porta, mas continua útil (sem health check, comando
     próprio).

   Isso resolve queda **no deploy**, não queda da instância — que continua sendo ponto único de falha.
10. **O primeiro deploy automatizado da API não fechou, e `api:stable` está à frente de produção.** Em
    14/09/2026 o `api-published` do merge do Biome (`8b9f7045fc37`) passou por `e2e`, `promote` e `mirror`,
    mas o `deploy` falhou em "Aplicar migrations" pela porta 8080. O `update-service` não rodou, então a
    API no ar continua na versão anterior, enquanto `api:stable` (GHCR e ECR) já aponta para a nova. Todo
    e2e disparado pelo front, até isso ser resolvido, valida contra uma API que não está em produção.
    O passo com a `cronos-migrate` entrou em `965871c`, e o merge do PR #18 na API (`03adfffca316`)
    dispara o primeiro deploy da API que já o usa. *Re-run* do run antigo não serviria, porque usaria o
    workflow anterior. **Para fechar:** aprovar esse deploy e confirmar o `services-stable`. Ver
    [`pipeline-ci-cd.md`](docs/pipeline-ci-cd.md) §10.1 e §9.2.
