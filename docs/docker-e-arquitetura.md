# Docker e Arquitetura — CRONOS

Documento de referência sobre os arquivos Docker dos três repositórios do sistema de controle de despesas (CRONOS), o que cada um faz linha a linha, e como eles se integram em runtime, em build-time e via CI/CD.

> **Atualizado em 19/08/2026.** Desde a primeira redação, o `Dockerfile` de produção do front foi
> reescrito (single-stage → multi-stage com `output: 'standalone'`) e as **duas** imagens passaram a
> ser publicadas como `linux/arm64` puro. As seções 2.1, 3.1, 4.3 e 6 refletem esse estado; o
> `Dockerfile` da API não mudou. A mudança do front ainda está **sem commit**, no working tree do
> branch `dev/gbrlmzl` — o que está no GHCR hoje é o artefato anterior.

| Repositório | Papel | Arquivos Docker |
|---|---|---|
| `sistema-controle-despesas-api` | Back-end (Express 5 + Prisma + Postgres) | `Dockerfile` (multi-stage), `docker-compose.yml` (perfis `dev`/`prod`), `.dockerignore` |
| `sistema-controle-despesas-front` | Front-end (Next.js 16 + React 19) | `Dockerfile` (produção), `Dockerfile.dev` (desenvolvimento), `docker-compose.yml`, `.dockerignore` |
| `sistema-controle-despesas-deploy` | Orquestrador dos testes E2E | `docker-compose.yml` (só orquestra — não tem código nem Dockerfile) |

![Arquitetura Docker do CRONOS](./arquitetura-docker.svg)

---

## 1. `sistema-controle-despesas-api`

### 1.1 `Dockerfile` — multi-stage com 4 estágios

O objetivo do multi-stage aqui é separar **o que é preciso para compilar** do **que é preciso para rodar**: o TypeScript, o `tsc`, os tipos e as devDependencies ficam todos em estágios intermediários e são descartados; a imagem final leva apenas `dist/`, o `node_modules` de produção e os artefatos do Prisma.

```dockerfile
ARG NODE_VERSION=24-bookworm-slim
```

**Debian slim, não Alpine.** Escolha deliberada: `bcrypt` é um módulo nativo e seus prebuilds são confiáveis em glibc (Debian), mas não em musl (Alpine). Usar Alpine forçaria a instalar toolchain de compilação (`python3`, `make`, `g++`) na imagem só para compilar o `bcrypt` do zero.

#### Estágio `deps`

```dockerfile
FROM node:${NODE_VERSION} AS deps
WORKDIR /app
RUN apt-get update -y && apt-get install -y --no-install-recommends openssl && rm -rf /var/lib/apt/lists/*
COPY package.json package-lock.json ./
RUN npm ci
```

- **`openssl`**: o *schema-engine* do Prisma (o binário nativo que executa `prisma migrate deploy`) depende de `libssl`. A imagem `slim` não traz OpenSSL — sem isso, o `prisma generate` emite warning e o `migrate` pode falhar de verdade em runtime.
- **`rm -rf /var/lib/apt/lists/*`** no mesmo `RUN`: apaga o cache do apt dentro da mesma camada, evitando que ele fique gravado na imagem.
- **`COPY package*.json` antes do resto do código**: garante que a camada cara (`npm ci`, que baixa centenas de pacotes) só seja invalidada quando as dependências mudarem, não a cada alteração no código-fonte.
- `npm ci` (não `npm install`): instalação determinística a partir do `package-lock.json`, com `node_modules` limpo.

Este estágio instala **dev + prod** porque quem herda dele precisa do `typescript`, do `tsx` e dos `@types/*`.

#### Estágio `build`

```dockerfile
FROM deps AS build
COPY tsconfig.json tsconfig.build.json prisma.config.ts ./
COPY prisma ./prisma
COPY src ./src
ENV DATABASE_URL="postgresql://placeholder:placeholder@localhost:5432/placeholder"
RUN npm run build
```

`npm run build` = `prisma generate && tsc -p tsconfig.build.json`, ou seja: gera o Prisma Client tipado a partir do `schema.prisma` e compila `src/` → `dist/`.

**Por que o `DATABASE_URL` placeholder?** O `prisma generate` não conecta no banco — ele só lê o schema. Mas o `prisma.config.ts` carrega essa variável de ambiente no momento do import, e quebraria se ela não existisse. O valor é irrelevante; só precisa estar presente. Nenhuma conexão real é feita durante o build.

> Este estágio não é apenas intermediário: o `docker-compose.yml` da API o usa **como alvo final** nos serviços `migrate` e `api-dev` (`target: build`), porque ambos precisam de ferramentas de desenvolvimento (`prisma` CLI, `tsx watch`).

#### Estágio `prod-deps`

```dockerfile
FROM node:${NODE_VERSION} AS prod-deps
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
```

Ramo **paralelo** (parte de `node:24-bookworm-slim` limpo, não de `deps`). Instala só as dependências de produção, produzindo um `node_modules` bem menor que o de `deps`. É esse `node_modules` que vai para a imagem final.

#### Estágio `runtime` — a imagem que sobe em produção

```dockerfile
FROM node:${NODE_VERSION} AS runtime
ENV NODE_ENV=production
RUN apt-get update -y && apt-get install -y --no-install-recommends openssl && ...
COPY --chown=node:node --from=prod-deps /app/node_modules ./node_modules
COPY --chown=node:node --from=build     /app/dist          ./dist
COPY --chown=node:node package.json ./
COPY --chown=node:node prisma.config.ts ./
COPY --chown=node:node prisma ./prisma
USER node
EXPOSE 8080
HEALTHCHECK ... CMD node -e "fetch('http://localhost:'+(process.env.PORT||8080)+'/health')..."
CMD ["node", "dist/server.js"]
```

Pontos importantes:

- **`openssl` de novo.** Este estágio parte de uma imagem base limpa; ele não herda o `apt-get` do estágio `deps`. E precisa mesmo: esta é a imagem a partir da qual o serviço `migrate` roda no repo de deploy.
- **`--chown=node:node`.** Os estágios anteriores rodam como `root`. Na primeira execução do `migrate deploy`, o Prisma baixa/escreve os binários da engine em `node_modules/@prisma/engines`. Sem o `chown`, o container (que roda como `node`) não teria permissão de escrita e o migrate falharia com `Can't write to .../@prisma/engines`.
- **`prisma/` e `prisma.config.ts` em runtime.** O `migrate deploy` lê `schema.prisma` e o diretório `migrations/` diretamente do disco — ele não usa o Prisma Client já gerado dentro de `dist/`. Sem esses arquivos, migrar a partir desta imagem seria impossível.
- **`USER node`.** A API não roda como root. (Note o contraste com o Dockerfile do front, que roda como root.)
- **`HEALTHCHECK` sem `curl`.** Usa o `fetch` global do Node 24 para bater no próprio `GET /health` — evita instalar `curl` na imagem só para isso. Respeita `PORT` se ela estiver definida.
- **`CMD node dist/server.js`** — executa o JavaScript já compilado; nada de `tsx`/`ts-node` em produção.
- O `prisma` CLI está em `dependencies` (não em `devDependencies`), então `npx prisma migrate deploy` funciona nesta imagem apesar do `--omit=dev`. É isso que permite ao repo de deploy usar a mesma imagem para o serviço `migrate`.

### 1.2 `.dockerignore`

Exclui do contexto de build: `node_modules`, `dist`, `src/generated`, `coverage`, `.git`, `.github`, `.env*`, `tests`, `docs`, `*.md`.

Dois efeitos: o contexto enviado ao daemon fica pequeno (build mais rápido) e — mais importante — **`.env` nunca entra na imagem**. Toda a configuração chega por variável de ambiente em runtime. Excluir `node_modules` também garante que o `npm ci` de dentro do container não seja contaminado por binários nativos compilados para o SO do host.

### 1.3 `docker-compose.yml` — perfis `dev` e `prod`

Este compose usa **profiles** para servir a dois cenários distintos com um só arquivo. Nada sobe sem `--profile`.

| Serviço | Perfis | O que é |
|---|---|---|
| `postgres` | `dev`, `prod` | Banco, em ambos os cenários |
| `migrate` | `dev`, `prod` | Job one-shot de migração |
| `api-dev` | `dev` | API com hot-reload e código montado por volume |
| `api` | `prod` | A imagem final enxuta (estágio `runtime`) |

#### `postgres`

```yaml
image: postgres:17-alpine
ports: ["${POSTGRES_PORT:-5432}:5432"]
volumes: [postgres_data:/var/lib/postgresql/data]
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ... -d ..."]
  interval: 5s / timeout: 5s / retries: 10
```

Volume **nomeado** (`postgres_data`) → os dados sobrevivem a `docker compose down`. O `healthcheck` com `pg_isready` é o que permite aos outros serviços esperarem o banco estar realmente pronto (não apenas "container iniciado"). A sintaxe `${VAR:-default}` deixa tudo funcionar sem `.env`, com defaults de desenvolvimento.

#### `migrate` — migração fora do processo que serve tráfego

```yaml
build: { context: ., target: build }
command: ["npx", "prisma", "migrate", "deploy"]
depends_on:
  postgres: { condition: service_healthy }
```

Roda uma vez e sai. **Nem `api` nem `api-dev` migram sozinhos ao subir** — decisão arquitetural deliberada: se a API migrasse no boot, N réplicas tentariam migrar em paralelo, e um deploy com migration quebrada derrubaria o serviço em vez de falhar num passo isolado e visível.

Usa `target: build` (não `runtime`) porque o estágio `build` já tem tudo instalado. Recebe apenas `DATABASE_URL`, montada a partir das mesmas variáveis do serviço `postgres`, apontando para o **hostname `postgres`** — o nome do serviço, resolvido pelo DNS interno da rede do compose.

#### `api-dev` (perfil `dev`)

```yaml
build: { context: ., target: build }
command: ["npm", "run", "dev"]     # tsx watch src/server.ts
env_file: [.env]
volumes:
  - ./src:/app/src
  - ./prisma:/app/prisma
depends_on:
  migrate: { condition: service_completed_successfully }
```

Hot-reload: `src/` e `prisma/` do host são montados por cima dos do container, e o `tsx watch` reinicia o processo a cada alteração. `node_modules` **não** é montado — permanece o do container, evitando conflito de binários nativos entre host e container.

`service_completed_successfully` significa: só inicia depois que o `migrate` sair com código 0. Se a migração falhar, a API não sobe.

`env_file: .env` traz o resto da configuração (JWT, Google OAuth, `FRONTEND_URL`), mas o `DATABASE_URL` do `.env` é sobrescrito pelo bloco `environment:` — porque no host o banco é `localhost`, e dentro da rede do compose é `postgres`.

#### `api` (perfil `prod`)

```yaml
build: { context: ., target: runtime }
image: ${API_IMAGE:-sistema-controle-despesas-api}:${IMAGE_TAG:-local}
restart: unless-stopped
environment: { NODE_ENV: production, DATABASE_URL: ... }
```

A imagem final — a **mesma** que o CI builda e publica no GHCR. O campo `image:` parametrizado (`${API_IMAGE}` / `${IMAGE_TAG}`) é o que permite ao workflow de CI usar `docker compose --profile prod build` e obter uma imagem já nomeada como `ghcr.io/gbrlmzl/sistema-controle-despesas-api:<sha12>`, pronta para `docker push`. Sem `.env`, o default é `sistema-controle-despesas-api:local`.

`restart: unless-stopped` reinicia o container se ele morrer, mas respeita uma parada manual.

---

## 2. `sistema-controle-despesas-front`

### 2.1 `Dockerfile` (produção)

Multi-stage em três estágios, sobre `node:24-bookworm-slim` (a base completa traz git/python/gcc que
nada aqui usa; o `slim` é Debian com glibc, exigida pelos binários prebuilt do SWC):

| Estágio | O que faz |
|---|---|
| `deps` | `npm ci` a partir do `package-lock.json` — determinístico, e falha se o lock estiver fora de sincronia. É a mesma árvore de dependências que a CI testou |
| `build` | copia o `node_modules` do estágio anterior, recebe `ARG API_URL` e roda `npm run build` |
| `runtime` | imagem final: copia só `.next/standalone`, `.next/static` e `public`; roda como `USER node`; tem `HEALTHCHECK`; `CMD ["node", "server.js"]` |

**O `output: 'standalone'` é o que dá forma a esse desenho.** Configurado no `next.config.ts`, ele faz
o `next build` rastrear e emitir, em `.next/standalone`, o `server.js` gerado mais **só o subconjunto
podado de `node_modules`** que o app realmente carrega em runtime. Isso dispensa o estágio `prod-deps`
que a API ainda usa (um `npm ci --omit=dev` à parte) e derruba o payload da aplicação de **>1 GB para
~47 MB** — medido: 43 MB de standalone + 2 MB de `.next/static` + 2 MB de `public`. A imagem final
fecha em **~390 MB**, porque a base `node:24-bookworm-slim` já contribui uns 250 MB sozinha.

Dois detalhes que só mordem depois:

- **`.next/static` e `public` são copiados à mão.** O standalone não os traz — leva só o servidor e o
  `node_modules` podado. Sem essas duas cópias a aplicação **sobe normalmente** e serve páginas sem
  CSS, sem JS de cliente e sem imagens: um sintoma que não aponta para a causa.
- **O entrypoint é `node server.js`, não `npm start`.** O `server.js` é gerado pelo próprio Next
  dentro do standalone, com o config já resolvido embutido (rewrites inclusos). O `next start` do
  `npm start` exigiria o `node_modules` de produção completo — exatamente o que o standalone
  eliminou.

**O ponto central deste arquivo continua sendo o `API_URL`.** O `next.config.ts` do front define um rewrite:

```ts
async rewrites() {
  return [{ source: "/api/:path*", destination: `${API_URL}/:path*` }];
}
```

Esse rewrite é um **proxy same-origin**: o navegador nunca fala com a API diretamente, só com a origem do próprio front, em `/api/*`. Isso elimina CORS no browser e — mais importante — faz os cookies de sessão (JWT + refresh token) pertencerem ao domínio do front, que é o que permite ao `proxy.ts` (guarda de rota) enxergá-los.

O Next resolve o **destino** desse rewrite em *build-time*, gravando-o no `routes-manifest.json`. Consequência prática: **trocar a API alvo exige rebuild da imagem**, não basta mudar a variável de ambiente. É exatamente essa característica que molda a arquitetura do repo de deploy (seção 3).

Detalhe de cache bem posicionado: o `ARG API_URL` aparece **depois** do `COPY . .`, dentro do estágio `build`. Como um `ARG` invalida o cache apenas das camadas seguintes, mudar o `API_URL` só refaz o `npm run build` — o `npm ci` do estágio `deps` sequer é tocado.

E há um segundo consumo de `API_URL` que o build **não** cobre: o `src/proxy.ts` (a guarda de rota do
Next 16) chama `POST /auth/refresh` a cada request e lê `process.env.API_URL` no boot do processo. Ou
seja, a variável precisa existir **também no ambiente do container**, não só como `--build-arg`. Sem
ela o processo sobe e **toda página responde 500**; como o `HEALTHCHECK` bate na raiz e checa `r.ok`,
o container nunca fica `healthy` — no ECS isso vira um service em loop de restart, com a causa real
escrita no log da aplicação (`Error: Variável de ambiente API_URL não configurada.`), não no evento do
ECS. Ver seção 3.1 e o README do front.

O contraste com o Dockerfile da API deixou de existir: os dois são multi-stage, usam `npm ci`, rodam
como `USER node` e declaram `HEALTHCHECK`. A única diferença estrutural que sobra é o estágio
`prod-deps` da API, que o front dispensa por causa do standalone.

**Plataforma:** a imagem publicada é `linux/arm64` pura (ver seção 4.3). O `Dockerfile` em si não
fixa arquitetura nenhuma — quem escolhe é o `platforms:` do `build-push-action`; buildar localmente
continua produzindo a arquitetura do host.

### 2.2 `Dockerfile.dev` (desenvolvimento)

```dockerfile
FROM node:24
WORKDIR /app-node
COPY package*.json ./
RUN npm install
EXPOSE 3000
```

Não faz build de produção e **não define `ENTRYPOINT`/`CMD`** — o comando vem do `docker-compose.yml`. O `npm install` aqui existe só para deixar uma camada de dependências já cacheada na imagem; em runtime o compose reexecuta `npm install` depois de montar o volume, mantendo o `node_modules` coerente.

Note que ele também não precisa de `API_URL`: em modo `next dev` não há build de produção congelando o rewrite; o `next.config.ts` é lido a cada boot do servidor de desenvolvimento.

### 2.3 `.dockerignore`

Exclui `node_modules`, `.next`, `.git`, `coverage`, `cypress`, `docs`, `prototipos`, `.github`, `*.tsbuildinfo`, `.env`, `.env.local`, `README.md` e os próprios arquivos Docker.

Excluir `.next` é essencial: um build local do host não deve vazar para a imagem e ser confundido com o build feito lá dentro (que é o que carrega o `API_URL` correto).

### 2.4 `docker-compose.yml`

```yaml
services:
  app:
    build: { context: ., dockerfile: Dockerfile.dev }
    environment:
      NODE_ENV: development
      API_URL: ${API_URL:-http://host.docker.internal:8080}
    ports: ["3000:3000"]
    volumes:
      - .:/app-node
      - /app-node/node_modules
      - /app-node/.next
    extra_hosts: ["host.docker.internal:host-gateway"]
    entrypoint: ["sh", "-c"]
    command: ["npm install && npm run dev"]
```

Sobe **apenas o front**, em modo desenvolvimento. A API e o Postgres ficam fora deste compose.

- **`API_URL: host.docker.internal:8080`** — como não há serviço `api` nesta rede, o front precisa alcançar o host. `host.docker.internal` é o nome que o Docker Desktop resolve para a máquina hospedeira; o `extra_hosts: host-gateway` faz o mesmo funcionar no Docker Engine em Linux, onde esse nome não existe nativamente.
- **Os três volumes.** `.:/app-node` monta o projeto inteiro (hot-reload). Os dois seguintes, `/app-node/node_modules` e `/app-node/.next`, são *volumes anônimos* usados como **máscara**: eles têm precedência sobre o bind mount naquele caminho, impedindo que o `node_modules` e o `.next` do host sobrescrevam os do container. Sem eles, um `node_modules` do Windows quebraria o container Linux.
- **`entrypoint: sh -c` + `command`** — reescreve o entrypoint para poder encadear dois comandos; o `npm install` roda depois do volume montado, sincronizando as dependências com o `package.json` atual.

> **Nota de manutenção:** o comentário no topo desse arquivo diz que o repositório da API está "hoje sem Dockerfile/compose próprio". Isso não é mais verdade — a API tem ambos. O comportamento do compose continua correto (ele realmente sobe só o front); apenas o comentário ficou desatualizado.

---

## 3. `sistema-controle-despesas-deploy`

Este repositório **não tem Dockerfile** — e isso é proposital. Ele não contém código de aplicação nem specs de teste; só monta o ambiente onde o sistema completo é exercitado.

### 3.1 `docker-compose.yml`

Quatro serviços, encadeados por condições de dependência:

```
postgres (healthy) → migrate (exited 0) → api (healthy) → front
```

#### `postgres`

Igual ao da API, com duas diferenças significativas:

- `POSTGRES_DB` default é `sistema_despesas_e2e` (banco separado, não o de desenvolvimento);
- **não há volume** — os dados são efêmeros de propósito. Cada execução do E2E parte de um banco vazio, o que torna os testes determinísticos e independentes de execuções anteriores.

#### `migrate`

```yaml
image: ${API_IMAGE:-ghcr.io/gbrlmzl/sistema-controle-despesas-api}:${API_IMAGE_TAG:-latest}
command: ["npx", "prisma", "migrate", "deploy"]
depends_on: { postgres: { condition: service_healthy } }
```

Roda a partir da **mesma imagem da API** que será testada — não de um build local. Isso garante que as migrations aplicadas correspondam exatamente à versão do código que vai atender às requisições. (É o motivo pelo qual o estágio `runtime` do Dockerfile da API precisa carregar `prisma/`, `prisma.config.ts` e o OpenSSL.)

#### `api`

```yaml
image: ${API_IMAGE:-ghcr.io/gbrlmzl/sistema-controle-despesas-api}:${API_IMAGE_TAG:-latest}
environment:
  NODE_ENV: production
  PORT: 8080
  DATABASE_URL: postgresql://...@postgres:5432/...
  FRONTEND_URL: http://localhost:3000
  JWT_SECRET: ${JWT_SECRET:?defina JWT_SECRET no .env (mínimo 32 caracteres) — ver .env.example}
  JWT_EXPIRES_IN: 15m
  REFRESH_TOKEN_EXPIRES_IN: 7d
depends_on: { migrate: { condition: service_completed_successfully } }
healthcheck: [node -e "fetch('http://localhost:8080/health')..."]
```

Consome a **imagem publicada no GHCR**, não builda do código-fonte. Como a API só lê variáveis de ambiente em runtime — nada é congelado no build dela —, a mesma imagem serve para qualquer ambiente.

A sintaxe **`${JWT_SECRET:?mensagem}`** é uma variável obrigatória: se estiver vazia ou ausente, o `docker compose` aborta imediatamente com essa mensagem, em vez de subir uma API com um segredo em branco.

`FRONTEND_URL: http://localhost:3000` é a origem que a API aceita em CORS e para onde redireciona após o login Google. É `localhost` porque é a URL vista pelo **navegador** (ou pelo Cypress), não a vista de dentro da rede do compose.

O `healthcheck` no nível do compose replica o que já existe como `HEALTHCHECK` no Dockerfile, sobrescrevendo-o com intervalos mais agressivos (5s) adequados a CI — é ele que o `docker compose up --wait` observa.

#### `front`

```yaml
build:
  context: ${FRONT_CONTEXT:-../sistema-controle-despesas-front}
  args: { API_URL: http://api:8080 }
environment:
  NODE_ENV: production
  API_URL: http://api:8080
depends_on: { api: { condition: service_healthy } }
healthcheck: [node -e "fetch('http://localhost:3000')..."]
```

**Este é o serviço mais interessante da stack, e o único que quebra o padrão "consuma o artefato publicado".**

O front é buildado a partir do código-fonte porque, como visto na seção 2.1, o Next congela o destino do rewrite `/api/*` em build-time. A imagem publicada no GHCR carrega o `API_URL` que valia no momento do build dela, sem garantia de apontar para o serviço `api` desta rede. Buildar aqui com `--build-arg API_URL=http://api:8080` é a única forma de o proxy funcionar de fato dentro do compose, **seja qual for a configuração do CI do front**.

> **Nota de precisão (21/08/2026):** até 20/08 a imagem publicada carregava literalmente o placeholder `http://localhost:8080` — a Repository Variable `API_URL` do repo do front não existia, e o CI caía no fallback. Isso quebrou produção (ver [`problema-rewrite-api-build-time.md`](../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md)) e a variable foi configurada como `http://api:3001`, que na época coincidia com a porta usada por este compose. **Não coincide mais**: a API padronizou sua porta para `8080` (§1) e este compose foi atualizado junto (§3), enquanto a Repository Variable do repo do front continua em `http://api:3001` até aquele repositório também mudar — exatamente a quebra silenciosa que este parágrafo alertava. Mantenha o build do fonte enquanto o rewrite for resolvido em build-time.

O `API_URL` aparece **duas vezes**, e as duas são necessárias:
- em `args:` — para congelar o destino do rewrite durante `npm run build`;
- em `environment:` — porque o `src/proxy.ts` (guarda de rota do Next 16) lê `process.env.API_URL` no
  boot para chamar `POST /auth/refresh` a cada request. Sem a variável em runtime, ele lança
  `Error: Variável de ambiente API_URL não configurada.` e **toda página responde 500**, mesmo com o
  rewrite já compilado na imagem. (Antes do build `standalone`, o motivo alegado aqui era o `throw`
  do `next.config.ts` reavaliado pelo `next start`; com o config embutido no `server.js`, o
  requisito continua o mesmo, mas quem o impõe é o `proxy.ts`.)

`${FRONT_CONTEXT}` aponta, por padrão, para o repositório irmão no disco (`../sistema-controle-despesas-front`); no CI, o workflow faz checkout do front em `./front` e sobrescreve essa variável.

### 3.2 Variáveis de ambiente

| Variável | Papel | Default |
|---|---|---|
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | Credenciais do Postgres efêmero | `postgres` / `postgres` / `sistema_despesas_e2e` |
| `POSTGRES_PORT` | Porta exposta no host | `5432` |
| `JWT_SECRET` | **Obrigatória** — mínimo 32 caracteres | — (aborta se ausente) |
| `API_IMAGE` | Repositório da imagem da API no registry | `ghcr.io/gbrlmzl/sistema-controle-despesas-api` |
| `API_IMAGE_TAG` | Tag da imagem da API a testar | `latest` |
| `FRONT_CONTEXT` | Caminho do código-fonte do front | `../sistema-controle-despesas-front` (`./front` no CI) |

---

## 4. Como os três se integram

### 4.1 Integração em runtime — a rede e o caminho de uma requisição

Dentro do compose do repo de deploy, todos os serviços compartilham a rede default e se enxergam pelo **nome do serviço** via DNS interno do Docker. O caminho completo de uma requisição autenticada:

```
navegador
   │  GET http://localhost:3000/despesas
   ▼
front (:3000)  ── Next.js, next start
   │  rewrite /api/:path* → http://api:8080/:path*
   │  (server-side; o browser nunca vê essa URL)
   ▼
api (:8080)  ── Express: valida JWT do cookie, autoriza
   │  DATABASE_URL → postgres:5432
   ▼
postgres (:5432)
```

Três decisões arquiteturais que só fazem sentido juntas:

1. **Proxy same-origin.** O navegador só conversa com `localhost:3000`. O salto para a API acontece server-side, dentro da rede do Docker.
2. **Cookies no domínio do front.** Como todo o tráfego passa pela origem do front, os cookies `JWT`/`refreshToken` pertencem a ela — o middleware de rota do Next consegue lê-los para proteger páginas.
3. **Sem CORS no browser.** A configuração `FRONTEND_URL` da API existe para os casos em que o front a chama diretamente (desenvolvimento) e para o redirect final do login Google.

**Portas expostas no host** (todas as três, porque o Cypress roda de fora dos containers):

| Serviço | Porta host | Quem usa |
|---|---|---|
| `front` | 3000 | navegador / Cypress (`baseUrl`) |
| `api` | 8080 | debug manual, `curl /health` |
| `postgres` | 5432 | inspeção manual — pode conflitar com um Postgres local; use `POSTGRES_PORT` |

### 4.2 Integração em build-time — a assimetria fundamental

| | API | Front |
|---|---|---|
| Configuração | 100% em runtime (env vars) | `API_URL` congelado no build |
| Uma imagem serve vários ambientes? | Sim | **Não** — uma imagem por API alvo |
| No repo de deploy | `image:` (pull do GHCR) | `build:` (do código-fonte) |

Essa assimetria é a razão de existir do parâmetro `FRONT_CONTEXT` e do requisito de os repositórios estarem clonados como irmãos:

```
Projetos/
├── sistema-controle-despesas-front/
└── sistema-controle-despesas-deploy/
```

### 4.3 Integração via CI/CD — *build once, promote everywhere*

Cada repositório testa a si mesmo na própria pipeline; o repo de deploy é o único lugar que testa a **combinação**.

**CI da API** (`push`/`pull_request` em `main`):

1. `test` — Postgres como *service container*, `prisma migrate deploy`, `npm run build`, `npm test` (unit + integração). O `JWT_SECRET` é gerado efêmero com `openssl rand -hex 32`.
2. `build` — `docker compose --profile prod build`, com um `.env` mínimo contendo `API_IMAGE`/`IMAGE_TAG` para que as imagens já saiam nomeadas corretamente. Salva `api` e `api-migrate` como artifact (`docker save`).
3. `smoke-test` — `docker load`, `.env` efêmero, `docker compose --profile prod up -d` e loop de `curl /health`. Ou seja: **valida a própria imagem subindo a stack de verdade**, não só que o build compilou.
4. `publish` (só em `main`) — push para GHCR como `:<sha12>` e `:latest`.
5. `dispatch` — `POST /repos/gbrlmzl/sistema-controle-despesas-deploy/dispatches` com `event_type: api-published` e `client_payload.image_tag`.

**CI do front** (`push` em `main`, `pull_request` em `main`, com `concurrency` + `cancel-in-progress`):

1. `lint` — eslint.
2. `test` — jest com coverage (`--ci`). Roda em paralelo com o `lint`.
3. `build` — `next build` (com `API_URL` do ambiente do step, já que fora de `NODE_ENV=test` o Next não carrega `.env.test` sozinho). Depende dos dois anteriores.
4. `docker-publish` (só em push na `main`) — `setup-qemu-action` + `setup-buildx-action` + `build-push-action` com **`platforms: linux/arm64`**, `--build-arg API_URL=${{ vars.API_URL || placeholder }}`, cache GHA e tags `:latest`, `:sha-<short>` e `:sha-<completo>`.
5. `dispatch` — job **separado** do `docker-publish` de propósito: se o `curl` falhar (secret ausente, por exemplo), só ele fica vermelho, e a imagem já publicada continua contando como sucesso. Envia `repository_dispatch: front-published` com `sha` e `image_tag: sha-<completo>`.

**CI do deploy** — job `e2e`:

1. Resolve `front_ref` e `api_image_tag` conforme o gatilho:

   | Disparo | `front_ref` | `api_image_tag` |
   |---|---|---|
   | `push`/`pull_request` aqui | `main` | `latest` |
   | `repository_dispatch: front-published` | `client_payload.sha` | `latest` |
   | `repository_dispatch: api-published` | `main` | `client_payload.image_tag` |
   | `workflow_dispatch` | input | input |

   Os valores do `client_payload` chegam por `env:`, não interpolados direto no `run:` — evita que um payload malformado seja interpretado como shell.
2. Checkout do front no ref resolvido, em `./front` (os specs do Cypress vivem lá).
3. Login no GHCR — o pacote da API é **privado**; sem login o `compose up` falha com `unauthorized`.
4. `.env` efêmero com `JWT_SECRET` aleatório e a `API_IMAGE_TAG` resolvida.
5. `docker compose up -d --build --wait` com `FRONT_CONTEXT=./front`. O `--wait` bloqueia até todos os healthchecks passarem — é aí que os healthchecks de `postgres`, `api` e `front` ganham função concreta.
6. `npx cypress run --config baseUrl=http://localhost:3000`, a partir de `front/`.
7. `docker compose logs` em caso de falha; `down -v` sempre.

**Job `promote`** — roda só quando o gatilho foi `repository_dispatch` (só aí existe um artefato específico recém-publicado a validar). Usa `docker buildx imagetools create` para re-taggear como `:stable` o manifesto **já publicado**, sem rebuild e sem baixar/reenviar camadas. É a garantia de que `:stable` é bit a bit o mesmo artefato que passou no E2E.

Isso fecha o princípio: **o artefato é buildado uma vez, no repo que o possui; testado como parte do sistema no repo orquestrador; e promovido — nunca reconstruído.**

**Tokens envolvidos:**

| Secret | Onde vive | Para quê |
|---|---|---|
| `GITHUB_TOKEN` | automático | push das imagens no GHCR do próprio repo |
| `DEPLOY_DISPATCH_TOKEN` | repos front e API | PAT com escopo `repo` para disparar o `repository_dispatch` |
| `GHCR_PROMOTE_TOKEN` | repo deploy | PAT com `write:packages` sobre os pacotes do front **e** da API — usado tanto para o pull da imagem privada quanto para o push da tag `:stable` |

### 4.4 O mesmo padrão, três vezes

Vale notar como as mesmas decisões aparecem consistentemente nos três repositórios:

- **Migração fora do processo que serve tráfego** — serviço `migrate` one-shot, tanto no compose da API quanto no do deploy; nunca no boot da aplicação.
- **Healthcheck real, não `sleep`** — `pg_isready` no banco, `GET /health` na API, HTTP no front; e `depends_on` com `condition:` em vez de espera cega.
- **Segredos só em runtime** — `.dockerignore` exclui `.env`; o CI gera `JWT_SECRET` efêmero a cada job; nenhum segredo entra em camada de imagem.
- **Defaults de desenvolvimento embutidos** — `${VAR:-default}` em quase tudo, exceto onde um default seria perigoso (`JWT_SECRET`, que usa `${VAR:?erro}`).

---

## 5. Comandos práticos

**API — desenvolvimento com hot-reload:**

```bash
docker compose --profile dev up
```

**API — validar a imagem de produção localmente:**

```bash
docker compose --profile prod up --build
```

**Front sozinho, apontando para uma API que roda no host:**

```bash
docker compose up --build
```

**Sistema completo + E2E (a partir do repo de deploy):**

```bash
cp .env.example .env
```

```bash
docker compose up -d --build --wait
```

```bash
cd ../sistema-controle-despesas-front && npm ci && npx cypress run --config baseUrl=http://localhost:3000
```

**Derrubar tudo, incluindo volumes:**

```bash
docker compose down -v
```

---

## 6. Pontos de atenção

- **`.env` do repo de deploy não é opcional.** Sem `JWT_SECRET` preenchido, o `compose up` aborta antes de subir qualquer container (comportamento intencional).
- **Conflito na porta 5432.** O compose de deploy publica o Postgres no host. Se já houver um Postgres local, defina `POSTGRES_PORT` no `.env`.
- **Repositórios irmãos.** O default de `FRONT_CONTEXT` assume `../sistema-controle-despesas-front`. Fora dessa estrutura, é preciso passar o caminho explicitamente.
- **A imagem do front publicada no GHCR não serve para a stack de deploy.** Ela existe para ambientes onde a variável `API_URL` do repositório (Repository Variable no GitHub) aponta para a API real; se essa variável não estiver configurada, a imagem publicada carrega o placeholder `http://localhost:8080`.
- **A assimetria de hardening entre as duas imagens acabou.** Era o ponto mais antigo desta lista: a
  imagem da API era multi-stage e não-root, a do front era single-stage e rodava como root. O
  `Dockerfile` do front foi reescrito exatamente no tratamento que este documento recomendava —
  multi-stage com `output: 'standalone'`, `npm ci`, `USER node` e `HEALTHCHECK` (seção 2.1).
- **As duas imagens publicadas são `linux/arm64` puro.** O único destino de deploy é uma instância
  Graviton (`t4g.micro`), então os dois CIs fixam a plataforma em ARM e usam QEMU no runner x86.
  Consequência prática: **rodar** qualquer uma dessas imagens numa máquina x86 depende de QEMU
  registrado; **buildar** localmente, não — o `docker build` continua produzindo a arquitetura do
  host.
- ~~**O E2E deste repositório roda a imagem ARM da API num runner x86 — e isso precisa ser conferido no
  próximo run.**~~ **Confirmado e corrigido em 21/08/2026.** O job `e2e` executa em `ubuntu-latest`
  (amd64) e faz `docker compose up` da imagem publicada da API, que é arm64 — e quebrava exatamente
  como previsto. O sintoma real não foi `exec format error` visível: o compose avisava
  `The requested image's platform (linux/arm64) does not match the detected host platform
  (linux/amd64/v3)` para `api` e `migrate`, e o run morria em
  `service "migrate" didn't complete successfully: exit 255` — antes de qualquer spec rodar. A
  correção foi a prevista: um step `docker/setup-qemu-action@v3` antes do `compose up`. O front não
  tem esse problema aqui, porque é buildado do código-fonte, na arquitetura do runner.
- **Comentário desatualizado** no `docker-compose.yml` do front, mencionado na seção 2.4.
