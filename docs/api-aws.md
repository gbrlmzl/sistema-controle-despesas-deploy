# API na AWS — CRONOS

Registro de **como a API foi colocada no ar na AWS**, o que cada peça faz e por que cada decisão foi
tomada dessa forma. Corresponde à **Fase 4** do roteiro de [`arquitetura-aws.md`](./arquitetura-aws.md) §13.

> **Estado:** provisionado e verificado em 19/08/2026. API rodando como service ECS, conectada ao
> Postgres da [Fase 3](./banco-de-dados-aws.md), com as migrations aplicadas.
> **Método:** task definition e service criados pelo **console**; o resto por AWS CLI. A Fase 3 foi
> toda por CLI — a troca foi deliberada, para conhecer as duas interfaces.
> **Região:** `us-east-2`. **Instância:** `t4g.micro` (ARM64, 1 GB) — ver `arquitetura-aws.md` §-1.
> **Convenção:** ARNs usam `<conta>` no lugar do ID real da conta AWS, porque o repositório é público.

---

## 0. O que ficou de pé

| Peça | Recurso | Identificação |
|---|---|---|
| Imagem | Espelhada no ECR | `sistema-despesas-api:latest` |
| Credenciais | 2 parâmetros SSM `SecureString` | `/cronos/api/{database-url,jwt-secret}` |
| Permissão | `ecsTaskExecutionRole` ampliada | inline policy agora cobre `/cronos/api/*` |
| Logs | CloudWatch, retenção 7 dias | `/ecs/cronos-app` |
| Definição | Task definition `cronos-app:1` | ARM64, `bridge`, 448 MiB rígidos |
| Schema | `prisma migrate deploy` via `run-task` | executado uma vez, `exitCode: 0` |
| Execução | Service ECS `cronos-app` | `desiredCount=1`, disjuntor de implantação ligado |

Custo marginal desta fase: **~US$ 0,10/mês** (armazenamento da imagem no ECR). O compute já estava
pago; nenhum recurso cobrado por hora foi criado.

O que **não** foi criado, e é a decisão de maior impacto financeiro da fase: **nenhum Application Load
Balancer** — ver §8.

---

## 1. A imagem: por que ECR e não GHCR

O CI da API publica no **GHCR**, e o pacote é privado. Isso é um problema para o ECS: puxar de um
registry privado de terceiro exige o campo `repositoryCredentials` na task definition, apontando para
um segredo no **Secrets Manager** (o SSM Parameter Store não serve para esse campo específico).

Três caminhos foram considerados:

| Opção | Custo | Veredito |
|---|---|---|
| **Espelhar no ECR** | ~US$ 0,10/GB-mês | ✅ **escolhida** |
| Tornar o pacote GHCR público | US$ 0 | Expõe a imagem sem necessidade e mantém a dependência de um registry de terceiro |
| `repositoryCredentials` + Secrets Manager | US$ 0,40/mês por segredo | Paga por um segredo e adiciona uma permissão só para manter o status quo |

**Por que o ECR ganha**, e não é só preço: é o destino que a §1.9 do documento de arquitetura já
recomendava ("pull mais rápido, sem credencial de terceiro no host, sem depender de rate limit
externo"). E tem uma vantagem operacional que só fica óbvia na hora: **a autenticação já está
resolvida**. A policy gerenciada `AmazonECSTaskExecutionRolePolicy`, que a `ecsTaskExecutionRole` já
carregava desde a Fase 3, inclui `ecr:GetAuthorizationToken` e `ecr:BatchGetImage`. O agente ECS
autentica sozinho — o campo "Autenticação de registro privado" do console fica **vazio**.

> **Pendência:** o espelhamento foi feito **à mão**. A automação disso é a Fase 8 — estender o job
> `promote` do CI para empurrar no ECR depois que o E2E passar. Enquanto isso, toda imagem nova exige
> um `docker pull` + `tag` + `push` manual.

---

## 2. As credenciais, e a armadilha do caractere especial

Dois parâmetros novos no SSM, ambos `SecureString`:

| Parâmetro | Conteúdo |
|---|---|
| `/cronos/api/database-url` | connection string completa do Postgres |
| `/cronos/api/jwt-secret` | 64 caracteres hex (`openssl rand -hex 32`) |

**Por que o `DATABASE_URL` é um parâmetro único**, e não composto a partir dos três da Fase 3
(`user`, `password`, `db`): o bloco `secrets` do ECS injeta **valores inteiros** em variáveis de
ambiente. Não existe interpolação — não há como montar `postgresql://${user}:${pass}@...` a partir de
partes. A connection string precisa existir pronta.

### O `/` na senha quebra a URI silenciosamente

A senha do Postgres foi gerada com `openssl rand -base64 24` na Fase 3. O alfabeto base64 inclui
`+`, `/` e `=` — e uma `/` no meio da senha faz o parser da URI lê-la como início do *path*,
apontando para um banco que não existe.

Não foi hipotético: a senha real continha os dois caracteres. O valor gravado ficou assim
(trecho, com o encode visível):

```
postgresql://postgres:f45%2BKx43lI%2FcdXFrJATHnD2oQn7Mlw4X@172.31.18.227:5432/sistema_controle_despesas
                          └─ + ─┘      └─ / ─┘
```

O encode foi feito com `urllib.parse.quote(..., safe='')` antes de montar a string.

> **Se isso passasse batido**, o sintoma seria um erro de conexão ou de "banco inexistente" na
> subida da API — e o instinto seria desconfiar do Security Group ou do Postgres, não de um caractere
> na senha.

### O host é o IP privado da instância — e isso é uma dívida

A API e o Postgres estão em **tasks diferentes**, então não se enxergam por nome (`links` só valem
dentro da mesma task, em modo `bridge`). O caminho é o **IP privado da instância** na porta 5432
publicada no host — que é o que a §10 do documento de arquitetura recomenda, e que a regra
*self-reference* do Security Group já autoriza.

> **Pendência conhecida:** esse IP (`172.31.18.227`) está **gravado dentro do parâmetro SSM**. Se a
> instância for substituída, ele muda, e a API para de achar o banco — com um erro que não menciona
> IP nenhum. A solução estrutural é ECS Service Connect / Cloud Map (nomes DNS internos), citada na
> §10 do documento de arquitetura como upgrade natural. A solução barata, enquanto isso, é lembrar de
> atualizar o parâmetro junto com qualquer troca de instância.

---

## 3. A permissão, e o preço recorrente do menor privilégio

A inline policy `ReadCronosPostgresParams` da `ecsTaskExecutionRole` estava restrita a
`parameter/cronos/postgres/*`. Os novos parâmetros vivem em `/cronos/api/*` — **a role não conseguia
lê-los**, e a task falharia ao subir.

A policy passou a listar os dois prefixos:

```json
"Resource": [
  "arn:aws:ssm:us-east-2:<conta>:parameter/cronos/postgres/*",
  "arn:aws:ssm:us-east-2:<conta>:parameter/cronos/api/*"
]
```

Detalhe do comando: `put-role-policy` com o mesmo `--policy-name` **substitui a policy inteira**, não
acrescenta. Por isso o JSON precisou repetir o bloco do Postgres e o `kms:Decrypt`.

> **Vale nomear o custo desta escolha:** toda vez que um serviço novo precisar de um segredo novo,
> essa policy terá que ser ampliada de novo. É trabalho recorrente — exatamente o que um
> `parameter/*` evitaria, ao preço de que um comprometimento dessa role abriria **todos** os segredos
> da conta. A troca continua valendo a pena, mas ela não é de graça.

---

## 4. O log group precisa existir antes do console

O log group `/ecs/cronos-app` foi criado por CLI, com retenção de 7 dias, **antes** de abrir o
formulário. O console consegue criá-lo sozinho — e é justamente isso que se quer evitar: um grupo
criado automaticamente nasce **sem retenção**, ou seja, guardando log para sempre. É a armadilha de
custo nº 4 da §3 do documento de arquitetura.

### A opção `awslogs-create-group` tinha que sair

O formulário do console pré-preencheu a lista de opções do driver com:

```
awslogs-create-group = true
```

Isso faz o driver **tentar criar o grupo** a cada subida do container, o que exige a permissão
`logs:CreateLogGroup`. A `AmazonECSTaskExecutionRolePolicy` concede `CreateLogStream` e `PutLogEvents`,
mas **não** `CreateLogGroup`.

> Deixar essa opção ligada faria a task falhar por permissão do CloudWatch — um erro que aponta para
> "log" quando o problema real é um campo tentando fazer algo desnecessário, já que o grupo existe.

A opção foi removida, e o `awslogs-stream-prefix` (que veio com o genérico `ecs`) passou a `api`,
para identificar a origem dos streams quando a task tiver mais containers na Fase 5.

---

## 5. A task definition, campo a campo

| Campo | Valor | Justificativa |
|---|---|---|
| SO/Arquitetura | **`Linux/ARM64`** | A instância é `t4g.micro` (Graviton). Task amd64 aqui morre com `exec format error` |
| `networkMode` | `bridge` | §10 do documento de arquitetura |
| CPU/memória **de tarefa** | **em branco** | Ver §6 |
| Task role | **vazia** | A API não chama nenhuma API da AWS em runtime — INFRA-05 |
| Execution role | `ecsTaskExecutionRole` | Puxa a imagem do ECR e lê os secrets |
| Autenticação de registro privado | **vazia** | Desnecessária com ECR — ver §1 |
| `portMappings` | `3001:3001` | Porta fixa, publicada no host |
| `readonlyRootFilesystem` | **`true`** | Ver §7 |
| `healthCheck` | `/health` | Ver §9 |

**Variáveis em texto plano:** `NODE_ENV=production`, `PORT=3001`,
`FRONTEND_URL=http://localhost:3000`.

**Variáveis como `secrets`:** `DATABASE_URL` e `JWT_SECRET`, apontando para os ARNs do SSM.

**Nada de Google OAuth nem de SMTP** — e isso é uma decisão, não um esquecimento. O
[`env.ts`](../../sistema-controle-despesas-api/src/config/env.ts) valida esses dois grupos como
"tudo ou nada": preencher uma variável do grupo e esquecer as outras faz a API **recusar-se a subir**.
Como o `googleAuthEnabled`/`mailEnabled` degradam graciosamente quando tudo está ausente, deixá-las
fora é o caminho seguro até haver domínio (Fase 7) e credencial de SMTP.

> **Pendência:** `FRONTEND_URL` é um placeholder. Ele alimenta o `Access-Control-Allow-Origin` com
> `credentials: true` e o redirect do OAuth — precisa virar a origem real de produção na Fase 7.
> Foi setado explicitamente, em vez de deixar o default do Zod, para que a pendência fique **visível**
> na task definition.

---

## 6. O orçamento de memória, e o desvio deliberado

O console exige memória em **GB**, enquanto a CLI aceita MiB. Os valores foram escolhidos para
converter redondo:

| Limite | MiB | GB (digitado no console) |
|---|---|---|
| Flexível (`memoryReservation`) | 256 | `0.25` |
| **Rígido (`memory`)** | **448** | `0.4375` |

**Por que os dois campos de tarefa ficaram em branco.** No launch type EC2, CPU e memória em nível de
task são opcionais — deixá-los vazios move o controle para o nível do container, que é onde ele deve
estar quando os containers têm perfis diferentes. O console avisa que é preciso especificar **um dos
dois** níveis; o limite rígido do container satisfaz a validação (o flexível sozinho **não**).

### O limite rígido contraria o documento — de propósito

A §6 do documento de arquitetura recomenda `memoryReservation` (flexível) em quase tudo, e `memory`
(rígido) **só** no Postgres. Aqui a API também levou limite rígido. O motivo é a instância ter metade
da RAM prevista:

```
t4g.micro — ~950 MB registrados pelo ECS
├── postgres   384 MB (rígido, Fase 3)
├── api        448 MB (rígido, esta fase)
└── ~118 MB    reservados para o caddy da Fase 5
```

> Sem limite rígido, um vazamento de memória na API cresce até o kernel escolher uma vítima — e o OOM
> killer tende a escolher o **maior** processo, que seria o Postgres. **Com o limite, uma API
> descontrolada morre sozinha em vez de derrubar o banco junto.** Numa instância de 2 GB o
> flexível bastaria; em 1 GB, não.

### O risco do heap do V8 (não mitigado)

O V8 dimensiona o heap com base na memória que enxerga, e dependendo de como lê os limites do cgroup
pode tentar crescer **além** dos 448 MiB antes de o GC reagir. O resultado não seria uma exceção
tratável, e sim `SIGKILL` — `OutOfMemoryError` no `describe-tasks`, sem stack trace.

A mitigação é uma variável de ambiente, ainda **não aplicada**:

```
NODE_OPTIONS=--max-old-space-size=320
```

320 MiB de heap deixa ~128 MiB de folga não-heap (buffers, stack, engine do Prisma) dentro do teto de
448. Transforma pressão de memória em GC agressivo — lento, mas visível e recuperável — em vez de
morte súbita. Fica registrado como pendência, a aplicar se aparecer o primeiro OOM.

---

## 7. Filesystem raiz somente leitura

Ativado. Antes de ligar, o código foi varrido em busca de escrita em disco em runtime (`fs.write*`,
`multer`, `os.tmpdir`, `createWriteStream`) — **nenhuma ocorrência**:

- os avatares são uma whitelist fixa, não upload de arquivo;
- os logs vão para `stdout`, capturados pelo driver `awslogs`;
- os binários da engine do Prisma são gerados no build e copiados para a imagem (é o que o
  `--chown=node:node` do `Dockerfile` da API garante) — nada é baixado na primeira query.

**O que isso compra:** se um dia existir uma RCE na aplicação ou em uma dependência, o invasor não
consegue gravar payload no container. Custo zero de performance.

> **Se um dia algo precisar escrever** (gerar um PDF antes de enviar por email, por exemplo), o
> sintoma será `EROFS: read-only file system`. A saída correta **não** é desligar a proteção: é
> montar um volume `tmpfs` só no diretório que precisa, mantendo o resto travado.

Este item não estava na revisão de segurança da API — é um ganho encontrado no formulário do console.

---

## 8. O que não foi criado: o Application Load Balancer

O assistente do console oferece "Balanceamento de carga" como parte do fluxo de criação do service, e
a criação de um ALB chegou a ser aberta antes de ser cancelada.

**Não criar foi a decisão de maior impacto financeiro desta fase.** O §3 do documento de arquitetura
já a antecipava:

> ALB custa **US$ 20–25/mês** — o orçamento do sistema **inteiro** foi desenhado em ~US$ 25/mês. Um
> ALB quase dobraria a conta para resolver um problema que este desenho não tem: o único problema que
> um load balancer resolveria aqui é IP mudando a cada deploy, que é característica do **Fargate**.
> Numa EC2 com Elastic IP, o endereço já é estável.

O caminho de entrada planejado continua sendo **CloudFront → Caddy na instância** (Fases 5 e 7), sem
load balancer nenhum.

Consequência que aparece no formulário: das quatro estratégias de implantação oferecidas
(cumulativa, azul/verde, canário, linear), **só a cumulativa funciona sem load balancer** — as outras
três dependem dele para dividir tráfego entre versões.

---

## 9. Health check: `/health`, e por que não `/ready`

A task definition aponta o health check do container para **`/health`** — a rota barata, que não toca
o banco.

**Isto contraria uma linha da revisão de segurança da API** (§4.5, SEC-17), que sugere apontar o
`healthCheck` da task definition para `/ready`. O motivo de divergir:

> O health check de container do ECS é uma sonda de **liveness**: quando falha, o ECS **mata e recria
> a task**. O `/ready` consulta o banco. Juntando os dois: Postgres cai → `/ready` falha → ECS
> reinicia a API → a API nova também não acha o banco → **loop de restart** por um problema que não é
> dela.
>
> É exatamente o cenário que a própria tabela liveness/readiness daquele documento descreve como
> errado.

A recomendação de lá pressupõe um **ALB**, onde o target group carrega a readiness e decide
roteamento (tirar da rotação, não reiniciar). Como este desenho não tem ALB (§8), a readiness não tem
onde morar. O `/ready` continua existindo e útil para diagnóstico manual — e vira health check de
target group no dia em que um load balancer entrar em cena.

---

## 10. A migration: `run-task`, não um service

O banco estava vazio: o Prisma nunca havia rodado contra ele. Subir a API antes disso significaria um
processo de pé falhando em toda query. A ordem correta (§1.7 do documento de arquitetura) é
**migration primeiro, service depois**.

```bash
aws ecs run-task --cluster ec2-sistema-despesas --task-definition cronos-app --launch-type EC2 \
  --overrides '{"containerOverrides":[{"name":"api","command":["npx","prisma","migrate","deploy"]}]}'
```

**Por que `run-task` e não um service:** `run-task` sobe a task **uma vez** e não a recria quando ela
termina. Um service faria o oposto — trataria a saída bem-sucedida da migration como falha e a
reiniciaria em loop.

**Por que reusar a `cronos-app` em vez de uma task definition dedicada:** ela já tem imagem,
`DATABASE_URL` e execution role — tudo que a migration precisa. O `containerOverrides` troca só o
comando. Isso funciona porque o `Dockerfile` da API foi construído para isso: `prisma` está em
`dependencies` (não `devDependencies`), `prisma/` e `prisma.config.ts` são copiados para o estágio
final, e o `openssl` está instalado para o schema-engine.

> Quando a task definition ganhar Caddy e front (Fases 5 e 6), esse override deixa de ser adequado —
> subiria os três containers só para rodar uma migration. Aí vale uma `cronos-migrate` dedicada.

**`prisma migrate deploy` não reseta nada.** Ele aplica só as migrations pendentes e registra cada uma
em `_prisma_migrations`. Quem reseta é `migrate reset`, que não foi usado. A tabela `teste_persistencia`
criada à mão na Fase 3 é invisível para o Prisma — ele só enxerga a própria pasta `migrations/` e a
própria tabela de controle.

### O erro do primeiro `run-task`, e o que ele ensina

A primeira execução falhou. O diagnóstico inicial não achou nada:

```json
{ "exitCode": null, "reason": null }
```

> **`exitCode: null` com a task `STOPPED` significa que o container nunca chegou a rodar.** A falha
> foi antes da execução — provisionamento, permissão ou posicionamento —, e nesse caso a mensagem
> **não está no nível do container**. Está no `stoppedReason` da **task**:
>
> ```
> Fetching secret data from SSM Parameter Store in us-east-2:
> invalid parameters: /cronos/api/jwt-secret
> ```

O `/cronos/api/jwt-secret` simplesmente não havia sido criado. `invalid parameters` é a mensagem que
o ECS dá para "parâmetro inexistente", o que não é óbvio — soa como valor malformado, não como
ausência.

Criado o parâmetro, o segundo `run-task` saiu com `exitCode: 0`.

> **A lição de diagnóstico**, que vale para qualquer task ECS que não sobe: consultar sempre o
> `stoppedReason` no nível da **task**, não só `exitCode`/`reason` no nível do container. São dois
> lugares diferentes, e o segundo fica vazio exatamente nos casos em que o primeiro tem a resposta.

---

## 11. O service, e as opções do console

```
desiredCount = 1
minimumHealthyPercent = 0
maximumPercent = 100
```

Mesmo raciocínio da `cronos-data`: uma instância, porta fixa (3001), então a task antiga precisa sair
antes de a nova subir. O padrão (`100`) travaria o deploy para sempre.

Diferente da Fase 3, o service foi criado pelo **console** — e o formulário trouxe quatro decisões que
a CLI não pergunta:

| Opção do console | Escolha | Por quê |
|---|---|---|
| **Tipo de inicialização** vs. **Estratégia de provedor de capacidade** | Tipo de inicialização → EC2 | Um capacity provider pode estar ligado a *managed scaling* do ASG e **criar instâncias sozinho**. É o INFRA-01 da revisão de segurança: teto de compute fixo é a proteção estrutural contra DDoS financeiro. `launch-type` só posiciona no que já existe, e falha de forma visível se não couber |
| **Rebalanceamento de zonas de disponibilidade** | **Desativado** | Veio ligado por padrão e **bloqueava o formulário**, exigindo `maximumPercent > 100`. Não faz sentido aqui: uma instância, uma AZ (`us-east-2b`, a mesma do volume EBS). Subir o máximo para satisfazer a validação seria pior — o ECS tentaria rodar 2 tasks e esbarraria na porta 3001 |
| **Service Auto Scaling** | Desativado | Mesmo INFRA-01 |
| **ECS Exec** | Desativado | Daria `docker exec` remoto, mas exigiria permissões novas na task role — que decidimos manter **vazia**. O mesmo acesso já existe via `ssm start-session` + `sudo docker exec` |
| **Métricas de alta resolução (20s)** | Não — padrão de 60s | A resolução fina é paga; 60s é grátis e suficiente para qualquer alarme que valha a pena hoje |

### Disjuntor de implantação

Ativado, com **reversão automática**. É o mecanismo que teria encurtado o crash loop da Fase 3 (mais
de dez ciclos de restart até o log revelar o `lost+found`): depois de N falhas consecutivas, o ECS
**para de tentar** e volta para a revisão anterior.

Detalhe do "Valor limite" que confunde: o campo aceita uma porcentagem (ficou em `50`), mas com
`desiredCount = 1` isso daria `0,5`. A opção **"porcentagem limitada"** impõe um **piso de 3**, como o
próprio formulário avisa — então o limite efetivo é **3 falhas**, independente do número digitado.
Trocar para "ilimitada" seria pior: `50% × 1` arredondaria para baixo e o disjuntor dispararia na
primeira falha isolada.

---

## 12. Verificação

O `steady state` prova que o container não morreu — não prova que ele fala com o banco. Os dois testes
que fecham isso, de dentro da instância:

| Comando | O que prova |
|---|---|
| `curl -s http://localhost:3001/health` | O processo Node está vivo (sem tocar o banco) |
| `curl -s http://localhost:3001/ready` | **Ponta a ponta**: API → Security Group → Postgres → resposta |
| `sudo docker ps` | Os dois containers lado a lado, ambos `(healthy)` |

> Atenção ao ler o `docker ps`: o `(healthy)` do container da API vem do health check da task
> definition, que chama **`/health`**. Ele fica verde mesmo com o banco fora do ar — por construção
> (§9). Só o `/ready` responde pela conexão.

---

## 13. Pendências conhecidas

Em ordem de importância. Nenhuma bloqueia a Fase 5.

1. **O `DATABASE_URL` grava o IP privado da instância** (§2). Trocar a instância quebra a API de forma
   silenciosa. Atualizar o parâmetro junto, ou migrar para Service Connect / Cloud Map.
2. **`NODE_OPTIONS=--max-old-space-size=320` não aplicado** (§6). Perdeu urgência: a instância virou
   `t4g.small` (2 GB) e o RSS medido da API foi de 123,2 MiB de 448 reservados
   ([`front-aws.md`](./front-aws.md) §10.1). Continua sendo boa higiene, não mais um risco de
   "container morto sem log".
3. ~~**Swap ainda não configurado**~~ — **configurado**, junto com o resize para `t4g.small`
   ([`front-aws.md`](./front-aws.md), pré-requisitos).
4. **A regra da porta 3001 no Security Group usa CIDR** (`172.31.0.0/16`) em vez de *self-reference* —
   a inconsistência já anotada em [`banco-de-dados-aws.md`](./banco-de-dados-aws.md) §6. O service
   `app` agora existe; dá para corrigir. **Se a padronização da porta (item 9) for feita, corrija as
   duas coisas na mesma mexida** — é a mesma regra.
5. **`FRONTEND_URL` é um placeholder** (§5). Vira a origem real na Fase 7, junto com o item 7.
6. **O espelhamento no ECR é manual** (§1). Automatizar no job `promote` é a Fase 8 — hoje são
   **três** imagens (API, front e Caddy).
7. **Google OAuth e SMTP continuam sem variáveis** (§5) — e a decisão de deixá-las fora, que era
   correta enquanto não havia domínio nem credencial, agora tem custo visível: o botão de login com
   Google não funciona, e **a recuperação de senha completa o fluxo sem enviar o email**. São 4
   variáveis para o Google (`GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL`,
   `COOKIE_SESSION_SECRET`) e 5 para o SMTP (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`,
   `MAIL_FROM`), com os segredos no SSM. Lembre que o `env.ts` valida cada grupo como "tudo ou nada":
   **preencher pela metade impede a API de subir**. Detalhamento em
   [`ingresso-aws.md`](./ingresso-aws.md) §11.8.
8. **Falta decidir o provedor de SMTP.** O Amazon SES é o caminho natural dentro da conta, mas começa
   em **sandbox** — só envia para endereço verificado, e sair disso exige um pedido que leva alguns
   dias. Vale descobrir isso antes da Fase 7, não durante.
9. **Padronizar a porta da API em `8080`** (era `3001`, um dígito do `3000` do front). Legibilidade
   operacional, sem ganho técnico. **Lado código: feito** — `src/config/env.ts`, `.env.example`,
   `Dockerfile` e `ci.yml` no repo da API, e o `docker-compose.yml` do repo de deploy. **Lado infra
   AWS: pendente** — Security Group ainda libera só `3001`, e as task definitions
   `cronos-app`/`cronos-front` ainda apontam para `3001`. Ver [`ingresso-aws.md`](./ingresso-aws.md)
   §11.10 para a ordem de execução na AWS.

---

## 14. Referências

- [`arquitetura-aws.md`](./arquitetura-aws.md) — §-1 (estado real), §1.7 (migration), §1.9 (ECR), §3 (ALB), §6 (RAM), §13 (roteiro)
- [`banco-de-dados-aws.md`](./banco-de-dados-aws.md) — a Fase 3, de onde vêm o Postgres, a execution role e o Security Group
- [`revisao-seguranca-deploy-aws.md`](../../sistema-controle-despesas-api/docs/revisao-seguranca-deploy-aws.md) — SEC-03 (secrets), SEC-17 (`/health` vs `/ready`), INFRA-01 (autoscaling), INFRA-05 (IAM)
- [`env.ts`](../../sistema-controle-despesas-api/src/config/env.ts) — o contrato de variáveis de ambiente que a §5 implementa
