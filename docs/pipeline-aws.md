# Pipeline de deploy automatizado — Fase 8

Sequência de execução para fechar o último elo do pipeline. Até 11/09/2026 o CI ia do commit até a imagem
`:stable` no GHCR, e o trecho GHCR → ECR → ECS era feito à mão, com AWS CLI no terminal. Com este
documento executado, um merge na `main` chega até produção com **um clique de aprovação** no GitHub e
nenhum comando local.

> **Estado em 14/09/2026:** **em execução.** O deploy automatizado do **front** já funciona de ponta a
> ponta (12/09 e 13/09). O primeiro deploy automatizado da **API** (14/09) passou por `e2e`, `promote` e
> `mirror`, mas falhou no passo de migration. A correção entrou em `965871c` (Etapa 5.2), e o merge do PR #18
> da API, às 10h43, dispara o primeiro deploy que já a usa (em andamento no fechamento desta revisão).
> Corresponde à **Fase 8** do roteiro de [`arquitetura-aws.md`](./arquitetura-aws.md) §13. Como o pipeline
> funciona hoje, job a job: [`pipeline-ci-cd.md`](./pipeline-ci-cd.md).
> **Região:** `us-east-2`. **Cluster:** `ec2-sistema-despesas`. **Instância:** `t4g.small` (2 GB, ARM64).
> **Convenção:** ARNs usam `<conta>` no lugar do ID real, porque este repositório é público.

### Status por etapa

| Etapa | Status | Evidência |
|---|---|---|
| 1 — Front multi-arch e e2e com a imagem publicada | ✅ Concluída em 11/09 | Commits `f220415` (front) e `7a42d2f` (deploy) |
| 2 — `:stable` dos dois lados, promote do migrate | ✅ Concluída em 11/09 | Commits `819349c` e `065f8ad` |
| 3 — Levantamento na AWS | ✅ Concluída | Nota de 11/09 e medição de memória de 14/09 (630 MB livres) |
| 4 — OIDC e roles IAM | ✅ Concluída, com ajustes | O `mirror` assume a role desde a 3ª tentativa de 12/09 e o `deploy` desde a 4ª. Ver os ajustes em 4.2 |
| 5 — Jobs `mirror` e `deploy` | ✅ Na `main` desde 11/09 (`50a2e47`), migration corrigida em 14/09 (`965871c`) | A 1ª versão de "Aplicar migrations" falhou em 14/09. Ver 5.2 |
| 6 — Environment `production` | ✅ Concluída em 12/09 | *Required reviewers* + política de branch, com aprovações registradas nos runs |
| 7 — Primeiro deploy assistido | ⚠️ Front ✅ em 12/09 · API ⏳ (1ª tentativa ❌ em 14/09, 2ª em andamento) | Ver a Etapa 7 |
| 8 — Fechamento | 🔶 Parcial | README e rollback atualizados em 14/09. Lifecycle policy do ECR não verificada |

---

## 0. O que mudou, em uma tabela

| Peça | Antes (até 11/09) | Agora |
|---|---|---|
| Espelhamento GHCR → ECR | `docker pull` + `tag` + `push` manual, 3 imagens | Job `mirror` no CI (API, migrate e front; o Caddy continua manual) |
| Tag espelhada | `:latest` — **não validada pelo e2e** | `:stable` + tag imutável de SHA |
| Registro da task definition | Console ou CLI, à mão | Job `deploy`, via `describe` + patch |
| Migration | `run-task` manual | Passo do job `deploy`, numa task `cronos-migrate` derivada, antes do `update-service` |
| `update-service` | Manual, deliberadamente | **Continua exigindo uma decisão humana**, mas agora é um botão de aprovação no environment `production` |
| Credencial AWS no CI | Não existe | Duas roles assumidas por OIDC, sem chave estática |
| Imagem do front validada | Buildada do fonte no e2e, **≠ artefato publicado** | O artefato publicado é o que roda no e2e |

A ordem abaixo não é arbitrária. As Etapas 1 e 2 corrigiram os dois defeitos do pipeline anterior e
**precisavam vir antes** da automação: automatizar em cima deles seria consolidar o erro em código.

---

## Etapa 1 — Fechar "build once, promote everywhere" no front ✅

**O defeito.** O `docker-compose.yml` deste repositório builda o front a partir do código-fonte
(`build: context: ${FRONT_CONTEXT}`), enquanto o job `promote` marca como `:stable` a imagem que o CI
do front publicou no GHCR. São dois binários diferentes do mesmo commit. O Cypress nunca executou o
artefato que vai para produção. Para a API isso já está certo — o compose consome
`ghcr.io/.../sistema-controle-despesas-api` direto.

**A causa.** A imagem do front é `linux/arm64` pura, porque o único destino de deploy é Graviton. O
runner do e2e é `ubuntu-latest`, amd64. Consumir a imagem publicada ali rodaria o Next inteiro sob
QEMU — o comentário no compose registra que isso "só trocaria a emulação de lugar". A API resolveu o
mesmo problema publicando manifest multi-arch.

### 1.1 Publicar o front multi-arch

No repositório **`sistema-controle-despesas-front`**, em `.github/workflows/ci.yml`, job
`docker-publish`, passo `docker/build-push-action@v6`:

```yaml
          platforms: linux/amd64,linux/arm64
```

O `docker/setup-qemu-action@v3` já está no job e continua necessário para o leg arm64.

> **Custo:** o build amd64 é nativo no runner e some no ruído. O arm64 continua emulado, como hoje.
> Os dois legs rodam em paralelo dentro do buildx, e o `cache-from/cache-to: type=gha` já configurado
> cobre as camadas de `npm ci`.

**Verificação:** depois do merge,
`docker buildx imagetools inspect ghcr.io/gbrlmzl/sistema-controle-despesas-front:latest` deve listar
**duas** entradas em `Manifests`, `linux/amd64` e `linux/arm64`.

### 1.2 Fazer o e2e consumir a imagem publicada

Neste repositório, no `docker-compose.yml`, trocar o serviço `front`:

```yaml
  front:
    image: ${FRONT_IMAGE:-ghcr.io/gbrlmzl/sistema-controle-despesas-front}:${FRONT_IMAGE_TAG:-stable}
    environment:
      NODE_ENV: production
      API_URL: http://api:8080
    ports:
      - "3000:3000"
```

Só o bloco `build:` sai. `ports`, `depends_on` e `healthcheck` ficam como estão — em especial o
`ports`, porque o Cypress roda **no runner**, fora da rede do compose, e alcança o front por
`http://localhost:3000`. Sem o mapeamento, a suíte inteira falha por conexão recusada, e o sintoma
não aponta para o compose.

O `args: API_URL` sai junto. O `API_URL` continua em `environment`, que é o que o Route Handler lê
em runtime — era exatamente essa a condição registrada no comentário antigo do compose para poder
fazer a troca.

`FRONT_CONTEXT` deixa de existir. O checkout do repositório do front no job `e2e` **continua
necessário**: os specs do Cypress moram lá, não aqui.

### 1.3 Atualizar o `.env.example` e o README

Trocar a linha de `FRONT_CONTEXT` por `FRONT_IMAGE_TAG` e atualizar a tabela de variáveis de ambiente
do README. `FRONT_CONTEXT` some da seção "Rodando localmente" também — para iterar no front sem
publicar imagem, o caminho passa a ser `docker compose up -d postgres api` aqui e `npm run dev` no
repo do front.

### 1.4 Atualizar a seção "Arquitetura" e a pendência 2 do README ✅

**Feito em 14/09/2026.** A seção "Arquitetura" do README passou a mostrar o front rodando a imagem
publicada, como a API, e a pendência 2 foi riscada: o Route Handler está em produção, e o front publica a
mesma imagem para qualquer ambiente. O texto abaixo fica como registro do que motivou a mudança.

A 1.1-1.3 fecharam o defeito, mas o README deste repositório ainda descrevia o cenário antigo em dois
lugares:

- A seção **Arquitetura**: o diagrama rotula o front como "(buildado do fonte)", o parágrafo logo
  abaixo explica por que ele precisa ser buildado aqui, e a nota de precisão de 21/08/2026 e o bloco
  "⚠️ Essa assimetria é uma dívida conhecida" descrevem um front que ainda não segue "build once,
  promote everywhere". Isso deixou de ser verdade a partir da 1.2.
- A **pendência 2** da lista de Pendências ("o rewrite `/api/*` do front continua resolvido em
  build-time... a saída é um Route Handler em runtime") — o comentário do job `mirror` na Etapa 5.1
  já assume que esse Route Handler existe no repo do front. Vale conferir se a pendência foi resolvida
  também em produção (não só aqui, no compose do e2e) antes de riscá-la.

Quando esta etapa for retomada: reescrever o diagrama e os dois parágrafos da Arquitetura para
refletir que o front roda a imagem publicada como a API, e decidir o destino da pendência 2 — riscá-la
ou reformulá-la, conforme o que a checagem acima encontrar.

**Checkpoint da Etapa 1:** abra um PR neste repositório só com a mudança do compose. O e2e do PR sobe
a stack consumindo a imagem publicada e as 17 specs passam. Se passarem, o artefato testado e o
artefato promovido viraram o mesmo objeto.

---

## Etapa 2 — Parar de promover e espelhar o que o e2e não aprovou ✅

**O defeito.** O `:latest` é publicado pelos CIs do front e da API **antes** de o e2e rodar. O
`:stable` só existe depois que ele passa. O espelhamento manual de hoje usa `:latest` — ou seja,
manda para produção o artefato não validado, enquanto a tag validada fica parada no GHCR. Está
registrado como pendência 5 do README.

Há um segundo caso do mesmo defeito, mais discreto, dentro do próprio e2e: quando o gatilho é
`api-published`, o lado do front é fixado em `latest`; quando é `front-published`, o lado da API é
fixado em `latest`. Os dois deveriam ser `stable` — o contrato do e2e é validar **o artefato novo
contra o último bom conhecido do outro lado**, não contra um artefato que ninguém aprovou.

### 2.1 Trocar o lado oposto para `:stable` no job `e2e`

No `ci.yml` deste repositório, no passo `Resolver parâmetros do trigger`, o bloco passa a resolver
**duas** tags de imagem em vez de uma tag e um ref:

```bash
          case "${{ github.event_name }}" in
            repository_dispatch)
              if [ "${{ github.event.action }}" = "front-published" ]; then
                echo "front_ref=$DISPATCH_SHA"             >> "$GITHUB_OUTPUT"
                echo "front_image_tag=$DISPATCH_IMAGE_TAG" >> "$GITHUB_OUTPUT"
                echo "api_image_tag=stable"                >> "$GITHUB_OUTPUT"
              else
                echo "front_ref=main"                      >> "$GITHUB_OUTPUT"
                echo "front_image_tag=stable"              >> "$GITHUB_OUTPUT"
                echo "api_image_tag=$DISPATCH_IMAGE_TAG"   >> "$GITHUB_OUTPUT"
              fi
              ;;
```

`front_ref` continua existindo e continua sendo um ref de git — ele governa **só o checkout dos
specs**. `front_image_tag` é o que vai para o compose. Os dois vêm do mesmo commit quando o gatilho é
`front-published`, e é essa coincidência que dá sentido ao par.

Nos outros dois gatilhos (`push`/`pull_request` e `workflow_dispatch`), os defaults passam a ser
`stable` nas duas imagens, e os `inputs` do `workflow_dispatch` ganham um `front_image_tag`.

### 2.2 Semear o `:stable` antes do primeiro run

`:stable` não existe até o primeiro `promote` rodar. Antes de mergear a Etapa 2, crie as duas tags
uma vez, apontando para o que está em produção hoje:

```bash
docker buildx imagetools create --tag ghcr.io/gbrlmzl/sistema-controle-despesas-api:stable ghcr.io/gbrlmzl/sistema-controle-despesas-api:latest
```

```bash
docker buildx imagetools create --tag ghcr.io/gbrlmzl/sistema-controle-despesas-front:stable ghcr.io/gbrlmzl/sistema-controle-despesas-front:latest
```

São os penúltimos comandos manuais deste documento.

### 2.3 Promover também a imagem de migrate

O job `promote` hoje promove `sistema-controle-despesas-api` mas **não**
`sistema-controle-despesas-api-migrate`, que o CI da API publica junto. Acrescentar no passo de
promoção da API:

```bash
          docker buildx imagetools create \
            --tag ghcr.io/gbrlmzl/sistema-controle-despesas-api-migrate:stable \
            "ghcr.io/gbrlmzl/sistema-controle-despesas-api-migrate:$IMAGE_TAG"
```

**Checkpoint da Etapa 2:** com as Etapas 1 e 2 na `main`, um merge qualquer no repo do front deve
produzir, ao fim, um `:stable` novo no GHCR do front e um e2e que rodou contra a imagem publicada do
front e o `:stable` da API. A partir daqui, `:stable` é a única tag que significa "aprovado".

---

## Etapa 3 — Levantamento na AWS (o último uso da CLI local) ✅

Colete os valores que os jobs vão usar. Rode uma vez, guarde a saída.

```bash
aws sts get-caller-identity --query Account --output text
```

```bash
aws ecr describe-repositories --region us-east-2 --query "repositories[].repositoryName" --output table
```

Esperado: `sistema-despesas-api`, `sistema-despesas-front`, `caddy`. Se `sistema-despesas-api-migrate`
não existir, crie:

```bash
aws ecr create-repository --region us-east-2 --repository-name sistema-despesas-api-migrate --image-scanning-configuration scanOnPush=true
```

> **Nota (verificado em 11/09/2026, revisto em 14/09/2026):** este repositório existe (e precisa
> continuar sendo espelhado — ver 2.3 e 5.1) só para o `mirror` não quebrar e para manter paridade com o
> espelhamento manual antigo, que sempre incluiu as três imagens. `cronos-app` tem **um único
> container**, chamado `api`, e o `migrate` deste `docker-compose.yml` usa a **mesma imagem da API**
> (`${API_IMAGE}`), só troca o `command`. A Etapa 5.2 segue a mesma ideia: a task `cronos-migrate` que ela
> registra é **derivada da `cronos-app`**, com o container `api` e a imagem da API. Ou seja, nada do que
> este documento automatiza **consome de fato** a imagem `-migrate` hoje; ela só fica pronta no ECR.
>
> O plano original previa usar `containerOverrides` na própria `cronos-app` e só criar uma `cronos-migrate`
> por falta de memória. Na prática, o motivo foi a **porta**: ver 3.1 e 5.2.

```bash
aws ecs describe-services --region us-east-2 --cluster ec2-sistema-despesas --services cronos-app cronos-front --query "services[].{Servico:serviceName,TaskDef:taskDefinition,Desejado:desiredCount}" --output table
```

```bash
aws ecs describe-task-definition --region us-east-2 --task-definition cronos-app --query "taskDefinition.containerDefinitions[].name" --output text
```

Anote os nomes dos containers. O override da migration e o patch da imagem selecionam **por nome**,
não por índice — a Etapa 5.2 depende disso.

### 3.1 Conferir a folga de memória

```bash
aws ecs describe-container-instances --region us-east-2 --cluster ec2-sistema-despesas --container-instances $(aws ecs list-container-instances --region us-east-2 --cluster ec2-sistema-despesas --query "containerInstanceArns[]" --output text) --query "containerInstances[].{Registrada:registeredResources[?name=='MEMORY'].integerValue|[0],Livre:remainingResources[?name=='MEMORY'].integerValue|[0]}" --output table
```

**Por que isso importa aqui e não antes.** O passo de migration é um `run-task` de uma revisão
`cronos-migrate` derivada da `cronos-app`, que herda os **448 MiB rígidos** do container `api`. Ele roda
com a API antiga ainda de pé, porque o `update-service` vem depois. Havia ~694 MB livres na `t4g.small` em
20/08/2026 e 630 MB em 14/09/2026, então cabe.

Se este comando devolver menos de ~500 MB livres, o `run-task` não aloca a task. Ele **não fica preso em
`PROVISIONING`**: responde na hora com `tasks: []` e o motivo em `failures` (`RESOURCE:MEMORY`), e o passo
da Etapa 5.2 imprime isso e falha. A saída nesse caso é baixar o `memory` do container na derivação da
`cronos-migrate` (o `jq` da 5.2). A possibilidade já está antecipada em [`api-aws.md`](./api-aws.md) §11.

**O que o levantamento não pegou: a porta.** A `cronos-app` publica `hostPort: 8080` em `networkMode:
bridge`. Um `run-task` dela, com a API antiga ocupando a 8080, recebe `RESOURCE:PORTS`, com qualquer
quantidade de memória livre, e o `containerOverrides` não consegue remover `portMappings`. Foi o que
derrubou o primeiro deploy automatizado da API (14/09). A mesma porta fixa causa a queda de alguns segundos
a cada deploy (pendência 9 do [`README.md`](../README.md)).

---

## Etapa 4 — OIDC e as duas roles IAM ✅

> **O que a execução ensinou (12 a 14/09).** Todas as tentativas usaram o mesmo workflow, então cada
> falha que sumiu entre uma tentativa e a seguinte foi resolvida na AWS:
>
> - **`mirror`, 1ª e 2ª tentativas de 12/09:** falha em `configure-aws-credentials`, ou seja, o provedor
>   OIDC ou a trust policy da `github-actions-mirror` ainda não batia.
> - **`deploy`, 3ª tentativa de 12/09:** falha em `amazon-ecr-login`. O job faz login no ECR só para obter o
>   endereço do registry, e a policy original da `github-actions-deploy` não tinha
>   `ecr:GetAuthorizationToken`. A policy abaixo já inclui essa permissão.
> - **`deploy`, 1ª e 2ª tentativas de 14/09:** falha em "Registrar revisão nova" para `cronos-app`, sendo que
>   `cronos-front` sempre registrou sem erro. O que só a família da API tem é a task role. Se isso voltar a
>   acontecer, confira se o `iam:PassRole` abaixo cobre o **nome real** dessa role.

Isto substitui o terminal. Uma vez feito, o GitHub pega credencial temporária sozinho e nenhuma chave
estática existe em lugar nenhum.

### 4.1 Criar o provedor de identidade

Console IAM → Identity providers → Add provider → OpenID Connect.

| Campo | Valor |
|---|---|
| Provider URL | `https://token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |

### 4.2 Duas roles, não uma

A separação não é cerimônia: as duas metades do pipeline têm poder muito diferente, e a condição de
confiança de cada uma é o que garante que a metade perigosa **só pode ser assumida por um job
aprovado**.

| Role | Assumida por | Pode |
|---|---|---|
| `github-actions-mirror` | qualquer job na `main` deste repo | escrever nos repositórios ECR |
| `github-actions-deploy` | **só** jobs com `environment: production` | mexer no ECS |

Empurrar imagem para o ECR não muda nada em produção — por isso a metade `mirror` roda antes do
portão de aprovação, e por isso ela não precisa de nenhuma permissão de ECS.

**Trust policy da `github-actions-mirror`:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<conta>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:gbrlmzl/sistema-controle-despesas-deploy:ref:refs/heads/main"
      }
    }
  }]
}
```

**Trust policy da `github-actions-deploy`** — idêntica, trocando só o `sub`:

```json
        "token.actions.githubusercontent.com:sub": "repo:gbrlmzl/sistema-controle-despesas-deploy:environment:production"
```

Esse `sub` é emitido **apenas** para um job que declara `environment: production`. Como o environment
tem required reviewers (Etapa 6), a role fica inacessível sem aprovação humana. É o cadeado real; o
botão do GitHub sozinho seria só uma convenção.

**Permissões da `github-actions-mirror`** (policy inline):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:DescribeImages"
      ],
      "Resource": [
        "arn:aws:ecr:us-east-2:<conta>:repository/sistema-despesas-api",
        "arn:aws:ecr:us-east-2:<conta>:repository/sistema-despesas-api-migrate",
        "arn:aws:ecr:us-east-2:<conta>:repository/sistema-despesas-front"
      ]
    }
  ]
}
```

**Permissões da `github-actions-deploy`:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition",
        "ecs:DescribeTasks",
        "ecs:ListTasks"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["ecs:UpdateService", "ecs:DescribeServices"],
      "Resource": [
        "arn:aws:ecs:us-east-2:<conta>:service/ec2-sistema-despesas/cronos-app",
        "arn:aws:ecs:us-east-2:<conta>:service/ec2-sistema-despesas/cronos-front"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "ecs:RunTask",
      "Resource": "arn:aws:ecs:us-east-2:<conta>:task-definition/cronos-migrate:*",
      "Condition": {
        "ArnEquals": { "ecs:cluster": "arn:aws:ecs:us-east-2:<conta>:cluster/ec2-sistema-despesas" }
      }
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::<conta>:role/ecsTaskExecutionRole",
        "arn:aws:iam::<conta>:role/cronosAppTaskRole"
      ],
      "Condition": {
        "StringEquals": { "iam:PassedToService": "ecs-tasks.amazonaws.com" }
      }
    }
  ]
}
```

`ecs:RegisterTaskDefinition` não aceita permissão por recurso — o `*` é limitação da API, não
descuido. O `iam:PassRole` com a condição `PassedToService` é o que impede que essa role seja usada
para anexar a `ecsTaskExecutionRole` a qualquer outra coisa.

O `ecs:RunTask` restrito a `task-definition/cronos-migrate:*` é exatamente o que a Etapa 5.2 usa: a
migration roda numa revisão `cronos-migrate`, nunca na `cronos-app`. Com esta policy ao pé da letra, uma
migration por `containerOverrides` na `cronos-app`, como previa a versão anterior da 5.2, seria negada. Se a
role foi ampliada na AWS para testar esse caminho, dá para voltar a restringi-la a `cronos-migrate:*`.

---

## Etapa 5 — Os jobs `mirror` e `deploy` ✅

Acrescente ao `ci.yml` deste repositório, depois do job `promote`.

> **Status:** os dois jobs estão na `main` desde 11/09 (`50a2e47`). O `mirror` e o `deploy` do front rodam
> verdes. O passo "Aplicar migrations" da 5.2 foi reescrito depois da falha de 14/09, e o código abaixo já
> é a versão nova, commitada em `965871c`. O primeiro deploy da API com ela é o do merge do PR #18 da API.

### 5.1 `mirror` — GHCR para ECR

```yaml
  mirror:
    name: Espelhar imagem aprovada no ECR
    needs: promote
    if: github.event_name == 'repository_dispatch'
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    outputs:
      image_tag: ${{ steps.tag.outputs.image_tag }}
    steps:
      - name: Normalizar a tag imutável
        id: tag
        env:
          DISPATCH_IMAGE_TAG: ${{ github.event.client_payload.image_tag }}
        run: echo "image_tag=$DISPATCH_IMAGE_TAG" >> "$GITHUB_OUTPUT"

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GHCR_PROMOTE_TOKEN }}

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-mirror
          aws-region: us-east-2

      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      # imagetools copia manifesto e camadas entre registries diferentes sem
      # materializar a imagem no runner. Duas tags no destino: a de SHA, que e
      # imutavel e vai para a task definition, e :stable, que e so um rotulo
      # legivel de "o que esta aprovado agora".
      - name: Espelhar API (+ migrate)
        if: github.event.action == 'api-published'
        env:
          REG: ${{ steps.ecr.outputs.registry }}
          TAG: ${{ steps.tag.outputs.image_tag }}
        run: |
          for repo in sistema-controle-despesas-api sistema-controle-despesas-api-migrate; do
            dest="${repo/sistema-controle-despesas/sistema-despesas}"
            docker buildx imagetools create \
              --tag "$REG/$dest:$TAG" \
              --tag "$REG/$dest:stable" \
              "ghcr.io/gbrlmzl/$repo:stable"
          done

      - name: Espelhar front
        if: github.event.action == 'front-published'
        env:
          REG: ${{ steps.ecr.outputs.registry }}
          TAG: ${{ steps.tag.outputs.image_tag }}
        run: |
          docker buildx imagetools create \
            --tag "$REG/sistema-despesas-front:$TAG" \
            --tag "$REG/sistema-despesas-front:stable" \
            "ghcr.io/gbrlmzl/sistema-controle-despesas-front:stable"
```

**Por que duas tags no ECR, e por que a task definition usa a de SHA.** Se a task definition apontar
para `:stable`, cada `register-task-definition` gera uma revisão idêntica à anterior, o ECS não vê
motivo para puxar nada, e é preciso `--force-new-deployment` para forçar. Pior: olhando a task
definition não dá para saber qual build está rodando. Com a tag de SHA, a revisão nova é diferente da
anterior por construção, o rollback é registrar a revisão anterior, e o histórico do ECS vira o
histórico de releases.

### 5.2 `deploy` — o portão e o ECS

```yaml
  deploy:
    name: Deploy em producao
    needs: mirror
    if: github.event_name == 'repository_dispatch'
    runs-on: ubuntu-latest
    environment: production
    concurrency:
      group: deploy-production
      cancel-in-progress: false
    permissions:
      id-token: write
      contents: read
    env:
      AWS_REGION: us-east-2
      CLUSTER: ec2-sistema-despesas
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-deploy
          aws-region: us-east-2

      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      # A fonte da verdade da task definition continua sendo a AWS: baixa a
      # revisao ativa, troca so o campo "image" do container, registra. Os
      # campos removidos pelo "del" sao read-only — register-task-definition
      # rejeita o JSON se eles vierem junto.
      - name: Registrar revisao nova
        id: td
        env:
          REG: ${{ steps.ecr.outputs.registry }}
          TAG: ${{ needs.mirror.outputs.image_tag }}
          ACTION: ${{ github.event.action }}
        run: |
          if [ "$ACTION" = "api-published" ]; then
            FAMILY=cronos-app;   CONTAINER=api;   REPO=sistema-despesas-api
          else
            FAMILY=cronos-front; CONTAINER=front; REPO=sistema-despesas-front
          fi
          echo "family=$FAMILY" >> "$GITHUB_OUTPUT"

          aws ecs describe-task-definition --task-definition "$FAMILY" \
            --query taskDefinition > td.json

          jq --arg img "$REG/$REPO:$TAG" --arg c "$CONTAINER" '
            (.containerDefinitions[] | select(.name == $c) | .image) = $img
            | del(.taskDefinitionArn, .revision, .status, .requiresAttributes,
                  .compatibilities, .registeredAt, .registeredBy, .deregisteredAt)
          ' td.json > td-new.json

          ARN=$(aws ecs register-task-definition --cli-input-json file://td-new.json \
                  --query "taskDefinition.taskDefinitionArn" --output text)
          echo "arn=$ARN" >> "$GITHUB_OUTPUT"
          echo "Revisao registrada: $ARN"

      # Migration so quando a API mudou, e sempre ANTES do update-service — e a
      # traducao do depends_on: service_completed_successfully do compose.
      # Roda numa familia propria (cronos-migrate) derivada da revisao recem-registrada:
      # a cronos-app publica hostPort 8080, que a API antiga ainda ocupa, e o override
      # do run-task nao consegue tirar portMappings (o ECS devolveria RESOURCE:PORTS).
      - name: Aplicar migrations
        if: github.event.action == 'api-published'
        run: |
          jq '
            .family = "cronos-migrate"
            | (.containerDefinitions[] | select(.name == "api"))
              |= (del(.portMappings, .healthCheck) | .command = ["npx","prisma","migrate","deploy"])
          ' td-new.json > td-migrate.json

          TD=$(aws ecs register-task-definition --cli-input-json file://td-migrate.json \
                 --query "taskDefinition.taskDefinitionArn" --output text)
          echo "Migration com: $TD"

          OUT=$(aws ecs run-task --cluster "$CLUSTER" --task-definition "$TD" --launch-type EC2 \
            --output json)
          # run-task nao falha quando nao consegue alocar a task: devolve tasks=[] e o
          # motivo em failures (ex.: RESOURCE:MEMORY). Sem isso o wait recebe "None".
          TASK=$(echo "$OUT" | jq -r '.tasks[0].taskArn // empty')
          [ -n "$TASK" ] || { echo "run-task nao iniciou a migration:"; echo "$OUT" | jq '.failures'; exit 1; }
          aws ecs wait tasks-stopped --cluster "$CLUSTER" --tasks "$TASK"
          CODE=$(aws ecs describe-tasks --cluster "$CLUSTER" --tasks "$TASK" \
                   --query "tasks[0].containers[0].exitCode" --output text)
          echo "migrate exitCode=$CODE"
          [ "$CODE" = "0" ] || { aws ecs describe-tasks --cluster "$CLUSTER" --tasks "$TASK"; exit 1; }

      - name: Atualizar o service
        env:
          FAMILY: ${{ steps.td.outputs.family }}
          TD: ${{ steps.td.outputs.arn }}
        run: |
          aws ecs update-service --cluster "$CLUSTER" --service "$FAMILY" --task-definition "$TD"
          aws ecs wait services-stable --cluster "$CLUSTER" --services "$FAMILY"

      - name: Eventos do service (se algo falhar)
        if: failure()
        env:
          FAMILY: ${{ steps.td.outputs.family }}
        run: aws ecs describe-services --cluster "$CLUSTER" --services "$FAMILY" --query "services[].events[:15]"
```

**Por que a migration ganhou uma família própria.** A primeira versão deste passo rodava a revisão nova
da `cronos-app` com `containerOverrides` trocando o `command`. Isso esbarra na porta: a `cronos-app`
publica `hostPort: 8080`, a API antiga segura essa porta até o `update-service`, e override nenhum remove
`portMappings`. O `jq` monta, a partir do mesmo `td-new.json` que acabou de ser registrado, uma revisão
`cronos-migrate` com a mesma imagem, as mesmas roles e os mesmos secrets, mas **sem `portMappings`** (nada
para disputar) e **sem `healthCheck`** (a task sai assim que a migration termina, sem nunca ficar
*healthy*). O `run-task` também deixou de ser confiado cegamente: quando não consegue alocar, ele devolve
`tasks: []` com o motivo em `failures` e **não** falha. Por isso o passo lê o JSON e para com uma mensagem
legível, em vez de passar `None` ao `wait`.

Efeito colateral aceito: cada deploy da API registra duas revisões, uma de `cronos-app` e uma de
`cronos-migrate`.

`concurrency` com `cancel-in-progress: false` é obrigatório: front e API disparam o
`repository_dispatch` de forma independente, e dois merges próximos gerariam dois `update-service`
concorrentes no mesmo cluster. Cancelar um deploy pela metade seria pior que enfileirar.

O disjuntor de implantação já está ligado nos dois services, com `rollback: true` — uma revisão que
não estabiliza volta sozinha para a anterior, e o `wait services-stable` falha o job. Não há passo de
rollback automático a escrever.

---

## Etapa 6 — O botão ✅

> **Verificado em 14/09/2026:** o environment `production` existe desde 12/09, com *required reviewers*
> (`gbrlmzl`) e política de branch. Os runs de 12, 13 e 14/09 pararam em *Waiting for review* e registraram
> a aprovação. O repositório é público, então a proteção não depende de plano pago. `can_admins_bypass` está
> ligado, o que permite a um admin pular a revisão.

Repositório `sistema-controle-despesas-deploy` → Settings → Environments → New environment →
`production`.

| Proteção | Valor |
|---|---|
| Required reviewers | você |
| Deployment branches | `Selected branches` → `main` |
| Wait timer | 0 |

O job `deploy` para em **Waiting for review** e mostra "Review deployments → Approve and deploy" na
própria página do run. Sem aprovação, o OIDC não emite o `sub` de `environment:production` e a role
de deploy é inassumível — a Etapa 4.2 já garante isso do lado da AWS.

> **Atenção ao plano.** Protection rules em environment são gratuitas em repositório **público**. Em
> repositório privado exigem GitHub Pro, Team ou Enterprise. Se este repositório for privado e o
> plano for o gratuito, o environment é criado mas o `Required reviewers` não fica disponível, e o
> job passa direto — o portão some sem aviso. Confirme antes de mergear a Etapa 5.

### 6.1 Secret

| Nome | Onde | Valor |
|---|---|---|
| `AWS_ACCOUNT_ID` | Repository secret | ID da conta (Etapa 3) |

Não é segredo de verdade, mas mantém o ID fora de um repositório público sem precisar de outro
mecanismo. `GHCR_PROMOTE_TOKEN` e `DEPLOY_DISPATCH_TOKEN` já existem e não mudam.

---

## Etapa 7 — Primeiro deploy assistido ⚠️

> **Resultado.**
>
> - **Front ✅ — 12/09, 00h45** ([run](https://github.com/gbrlmzl/sistema-controle-despesas-deploy/actions/runs/34670092890)).
>   Ficou verde na 4ª tentativa, depois dos ajustes de IAM descritos na Etapa 4. Repetiu verde de primeira
>   em 13/09 ([run](https://github.com/gbrlmzl/sistema-controle-despesas-deploy/actions/runs/34798259756)).
> - **API, 1ª tentativa ❌ — 14/09, 00h28** ([run](https://github.com/gbrlmzl/sistema-controle-despesas-deploy/actions/runs/34802740567)).
>   O gatilho foi o merge do Biome (`8b9f7045fc37`), e não uma mudança inócua. `e2e`, `promote` e `mirror`
>   ficaram verdes. O `deploy` falhou duas vezes em "Registrar revisão nova" e, na 3ª tentativa, em
>   "Aplicar migrations", por causa da porta 8080 (Etapa 3.1). O `update-service` não rodou, então a API em
>   produção continua na versão anterior, enquanto `api:stable` já aponta para `8b9f7045fc37`.
>
> - **API, 2ª tentativa ⏳ — 14/09, a partir das 10h43.** A nova 5.2 entrou em `965871c`. Um *Re-run* do run
>   de 00h28 usaria o workflow antigo (`50a2e47`) e falharia igual, então o caminho é um dispatch novo. Foi o
>   que o merge do PR #18 da API (`03adfffca316`) gerou.
>
> **Para fechar a etapa:** aprovar esse deploy e confirmar, no log, `migrate exitCode=0` e o
> `services-stable` da `cronos-app`. Se precisar reaplicar o pipeline a uma tag já publicada, o comando está
> em [`pipeline-ci-cd.md`](./pipeline-ci-cd.md) §9.2.

Faça uma mudança inócua no repo da API (um comentário basta), mergeie na `main` e acompanhe:

1. CI da API: `lint` + `test` → `build` → `smoke-test` → `publish` → `dispatch`.
2. Deploy: `e2e` verde → `promote` cria `:stable` → `mirror` empurra para o ECR.
3. `deploy` para em **Waiting for review**. Confira, no log do `mirror`, que a tag espelhada é a do
   SHA que você acabou de mergear.
4. Aprove. O job registra `cronos-app:<N>` e `cronos-migrate:<M>`, roda a migration, atualiza o service e
   espera estabilizar.
5. Confirme:

```bash
curl -fsS https://cronos.gabrielmizael.com/api/health
```

Repita com uma mudança no front, para exercitar o outro ramo do `if`.

**Se a migration falhar com `run-task nao iniciou a migration`**, leia o `failures` impresso logo abaixo.
`RESOURCE:MEMORY` é memória: volte à Etapa 3.1 e reduza o `memory` da `cronos-migrate`. `RESOURCE:PORTS`
indica que a revisão ainda publica porta, ou seja, o `del(.portMappings)` não foi aplicado.

---

## Etapa 8 — Fechamento 🔶

1. **Lifecycle policy no ECR** ⏳ *(não verificada)*, nos repositórios espelhados: manter as 5 imagens mais
   recentes. Sem isso cada release acumula ~390 MB indefinidamente. Está recomendado em
   [`arquitetura-aws.md`](./arquitetura-aws.md) §12. Agora que o `mirror` empurra uma imagem por merge, a
   policy deixou de ser teórica.
2. **Atualizar as pendências** ✅ *(14/09, parcial)*. Os itens 4 e 5 do README deste repositório foram
   riscados, e os READMEs da API e do front deixaram de dizer que o deploy é manual. Falta o item 6 de
   [`api-aws.md`](./api-aws.md) §11: "três imagens espelhadas à mão" passa a ser zero, e o Caddy continua
   espelhado sob demanda, porque muda uma vez por ano.
3. **Registrar a mudança de contrato do `update-service`** ✅ *(14/09)*. O README afirmava que o passo final
   "é deliberadamente manual e deve continuar assim". A intenção continua valendo; o que mudou foi o
   mecanismo, de comando digitado para aprovação explícita. A frase foi reescrita, não apagada.
4. **Documentar o rollback** ✅ *(14/09, [`pipeline-ci-cd.md`](./pipeline-ci-cd.md) §9.3)*. Aprovar um deploy
   errado é agora mais rápido que antes, então o caminho de volta precisa estar escrito. Antes, confirme com
   `describe-services` qual revisão está em uso, porque um deploy que falha depois do registro deixa a
   revisão mais recente fora do ar:

```bash
aws ecs update-service --region us-east-2 --cluster ec2-sistema-despesas --service cronos-app --task-definition cronos-app:<revisao anterior>
```

Enquanto não houver um `workflow_dispatch` de rollback, esse é o único comando que ainda justifica
abrir o terminal.

---

## 9. O que este documento deliberadamente não faz

- **Não versiona as task definitions como JSON no repositório.** O job as lê da AWS e altera um campo.
  A alternativa — arquivos `aws/cronos-*.json` com `amazon-ecs-render-task-definition` — é melhor a
  longo prazo, mas duplicaria estado que a Fase 10 (Terraform) vai reescrever de qualquer forma, e
  introduziria drift entre o arquivo e o console durante meses. O ganho de infra-as-code vem inteiro
  com o Terraform; antecipá-lo pela metade custa mais do que rende.
- **Não toca no `cronos-edge` nem no `cronos-data`.** Caddy e Postgres não têm CI que publique imagem
  e não mudam por merge. Continuam como estão.
- **Não automatiza o espelhamento do Caddy.** Uma imagem de terceiro, atualizada manualmente algumas
  vezes por ano, não paga um job.
