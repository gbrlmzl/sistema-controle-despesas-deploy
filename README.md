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
└──────────┘     │        :8080        │     └──────────┘
                  └─────────────────────┘
```

**Front é buildado a partir do código-fonte, não da imagem publicada.** O Next.js resolve o destino do rewrite `/api/*` em build-time (ver comentário no `Dockerfile` do front) — a imagem publicada no GHCR carrega o `API_URL` do momento em que foi buildada, e não há garantia de que ele aponte pro serviço `api` desta rede. Buildar aqui, com `API_URL=http://api:8080`, é a única forma de o proxy funcionar de verdade dentro do compose **independentemente de como o CI do front foi configurado**.

> **Nota de precisão (21/08/2026):** até 20/08, a imagem publicada carregava literalmente o placeholder `http://localhost:8080`, porque a Repository Variable `API_URL` do repo do front não existia. Ela foi configurada como `http://api:3001` — que na época coincidia com o valor que este compose usava, já que o serviço aqui também se chama `api`. **Não coincide mais**: a API padronizou sua porta para `8080` e este compose foi atualizado junto, enquanto a Repository Variable do repo do front continua em `http://api:3001` até aquele repositório também mudar. **É a quebra silenciosa que este parágrafo já alertava** — nomes coincidindo, não desenho. Continue buildando do fonte até o Route Handler existir.

**API roda a imagem publicada.** Ela não tem esse problema — não tem nada baked-in no build, só lê variáveis de ambiente em runtime.

> ⚠️ **Essa assimetria é uma dívida conhecida, não uma característica desejável.** O front é a única peça do sistema que **não** segue o "build once, promote everywhere": a imagem `:stable` dele carrega o `API_URL` do ambiente em que foi buildada. Em 20/08/2026 isso quebrou produção — a imagem foi publicada com o placeholder e toda chamada feita pelo navegador falhou, enquanto tudo que rodava no servidor continuava funcionando. Ver [`problema-rewrite-api-build-time.md`](../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md).
>
> A correção é um **Route Handler** no front que resolva `process.env.API_URL` em runtime. Quando ela entrar, o build do fonte deste `docker-compose.yml` deixa de ser necessário e o front passa a rodar a imagem publicada, como a API. Pendente — ver [`ingresso-aws.md`](./docs/ingresso-aws.md) §11.9.

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

## Documentação da infra AWS

| Documento | Conteúdo |
|---|---|
| [`docs/arquitetura-aws.md`](docs/arquitetura-aws.md) | Decisões de arquitetura, custos, cenários e o **roteiro de fases** |
| [`docs/banco-de-dados-aws.md`](docs/banco-de-dados-aws.md) | Fase 3 — Postgres em container + volume EBS |
| [`docs/api-aws.md`](docs/api-aws.md) | Fase 4 — task definition da API, ECR, secrets no SSM |
| [`docs/ingresso-aws.md`](docs/ingresso-aws.md) | Fase 5 — Caddy, CloudFront, TLS, **e as pendências abertas** |
| [`docs/front-aws.md`](docs/front-aws.md) | Fase 6 — task definition do front, `extraHosts` |
| [`docs/separacao-de-tasks-front-api.md`](docs/separacao-de-tasks-front-api.md) | Por que front e API rodam em tasks separadas |
| [`docs/docker-e-arquitetura.md`](docs/docker-e-arquitetura.md) | O estado do Docker e do CI nos três repositórios |

## Pendências

Estado em 21/08/2026: **Fases 0 a 6 concluídas** — o sistema está no ar em
`https://d3c5d6t3539m1d.cloudfront.net`. O que falta, em ordem de importância:

1. **Google OAuth e SMTP não têm variáveis configuradas em produção.** O código dos dois está pronto,
   mas inerte: o botão de login com Google não funciona, e a recuperação de senha **completa o fluxo
   sem nunca enviar o email**. O `env.ts` da API valida cada grupo como "tudo ou nada" — preencher
   pela metade **impede a API de subir**. Ambas dependem do domínio da Fase 7. Ver
   [`ingresso-aws.md`](docs/ingresso-aws.md) §11.8.
2. **O rewrite `/api/*` do front continua resolvido em build-time.** Quebrou produção em 20/08/2026 e
   foi corrigido só no valor, não no mecanismo — a imagem do front segue carregando a topologia da
   rede. A Fase 7 muda o endereço público e reativaria o bug. A saída é um **Route Handler** em
   runtime; quando ele entrar, o build do fonte deste compose deixa de ser necessário. Ver
   [`ingresso-aws.md`](docs/ingresso-aws.md) §11.9.
3. **Fase 7 — domínio próprio.** `.com.br` no Registro.br, DNS na Cloudflare, ACM, Caddy com Let's
   Encrypt na origem. É o gatilho natural dos dois itens acima e de `FRONTEND_URL` deixar de ser
   placeholder — vale agrupar tudo numa revisão de task definition só.
4. **Fase 8 — automatizar o espelhamento no ECR.** Hoje são **três** imagens (API, front, Caddy)
   espelhadas à mão. Estender o job `promote` é o caminho. Note que o passo final (`update-service`)
   é deliberadamente manual e deve continuar assim.
5. **O espelhamento manual usa `:latest`, não `:stable` — e isso anula o propósito do job `promote`.**
   O `:latest` é publicado pelos CIs do front e da API **antes** de o e2e rodar; o `:stable` só existe
   **depois** que ele passa. Espelhar `:latest` para o ECR significa mandar para produção um artefato
   que a validação de ponta a ponta ainda não aprovou, enquanto a tag validada fica ali sem uso. Na
   prática costuma ser o mesmo conteúdo (o e2e passa logo em seguida), mas basta um e2e vermelho para
   a diferença aparecer no pior momento possível. **Correção:** trocar `:latest` por `:stable` nos
   comandos de espelhamento — e já deixar assim quando a Fase 8 automatizar o passo.
6. **Padronizar a porta da API em `8080`.** Hoje `3001` difere do `3000` do front em um dígito, o que
   já se mostrou fonte de confusão. Sem ganho técnico, só legibilidade — ~~6 arquivos de código~~ (já
   feito: 5 no repo da API, `docker-compose.yml` deste) e 3 mudanças de infra, ainda pendentes. Barato
   se feito junto com o item 2, que já exige rebuild dos dois lados. Ver
   [`ingresso-aws.md`](docs/ingresso-aws.md) §11.10.
7. **Sem WAF e sem rate limiting na borda** — decisão consciente de orçamento, registrada em
   [`ingresso-aws.md`](docs/ingresso-aws.md) §11.6. O CloudFront traz Shield Standard de graça (L3/L4),
   e o teto fixo de compute protege a fatura; nada protege contra sobrecarga da instância.
8. **O `Caddyfile` não sobrevive à substituição da instância** — criado à mão, não versionado. Mesma
   classe do mount do EBS; resolver as duas juntas num user-data. Ver
   [`ingresso-aws.md`](docs/ingresso-aws.md) §11.1.
