# Ingresso HTTPS na AWS — CRONOS

Plano de execução da **Fase 5** do roteiro de [`arquitetura-aws.md`](./arquitetura-aws.md) §13: pôr o
sistema acessível na internet, com TLS válido, **sem comprar domínio** e **sem load balancer**.

> **Estado:** implementado e verificado em 20/08/2026. Distribuição `E2EHRXTE92L61B`
> (`d3c5d6t3539m1d.cloudfront.net`) `Deployed`, EIP `77.112.41.239` associado, Caddy no ar em
> `cronos-edge`. Os quatro testes da §9 passaram — falta só o teste manual de login no navegador.
> **O usuário operacional não tinha permissão de IAM nem de CloudFront** — ver §4.2 (novo).
> **Montagem escolhida:** CloudFront → Caddy (HTTP) na instância — a primeira das três da
> [`arquitetura-aws.md`](./arquitetura-aws.md) §10.
> **Pré-requisitos já cumpridos:** front no ar (`cronos-front`, porta 3000 do host), API
> (`cronos-app`, porta 3001), Postgres (`cronos-data`). Ver [`front-aws.md`](./front-aws.md).
> **Ordem em relação à Fase 6:** invertida de propósito — o Caddy só existe para fazer proxy ao
> front, e subir a borda antes do destino seria montar um proxy sem para onde apontar.
> **Convenção:** o ID da conta aparece como `<conta>`; segredos, como `<...>`. O repositório é
> público.

---

## 0. O que muda e o que não muda

| Peça | Muda? |
|---|---|
| Task definitions / services `cronos-app`, `cronos-data`, `cronos-front` | ❌ intocados |
| Imagens do front e da API no ECR | ❌ intocadas |
| CI dos três repositórios | ❌ intocado |
| **Elastic IP** | ✅ **alocar e associar** |
| **Imagem do Caddy no ECR** | ✅ **espelhar** |
| **`/etc/caddy/Caddyfile` na instância** | ✅ **criar** |
| **Parâmetro SSM `/cronos/edge/origin-secret`** | ✅ **criar** |
| **Inline policy da `ecsTaskExecutionRole`** | ✅ **ampliar** (novo prefixo) |
| **Log group `/ecs/cronos-edge`** | ✅ **criar** |
| **Task definition + service `cronos-edge`** | ✅ **criar** |
| **Regra de entrada 80 no Security Group** | ✅ **criar** (restrita à prefix list do CloudFront) |
| **Distribuição CloudFront** | ✅ **criar** |
| `FRONTEND_URL` na task da API | ⚠️ **opcional** — ver §11.2, exige revisão nova e reinicia a API |

Nenhuma alteração destrutiva. O sistema atual continua no ar durante toda a fase; o que se constrói
é uma camada **na frente** dele.

### Custo

| Item | Custo/mês |
|---|---|
| Elastic IP (associado a instância ligada) | ~US$ 3,65 — **já pago hoje** pelo IPv4 público efêmero |
| CloudFront | **US$ 0** — 1 TB de saída e 10M requisições no free tier perpétuo |
| Certificado TLS (`*.cloudfront.net`) | **US$ 0** — incluso |
| Caddy (container) | US$ 0 marginal — mesma instância |
| ECR (imagem do Caddy, ~50 MB) | ~US$ 0,01 |
| SSM Parameter Store (SecureString, tier Standard) | US$ 0 |
| **Δ real da fase** | **~US$ 0,01** |

O EIP merece nota: **desde a mudança de preço de 2024 a AWS cobra por todo IPv4 público**, inclusive
o auto-atribuído. Trocar o IP efêmero por um EIP é neutro na fatura e compra estabilidade.

---

## 1. A montagem escolhida, e por que não as outras

A §10 do documento de arquitetura lista três entradas HTTPS possíveis. A escolhida é a primeira:

```
Internet ──HTTPS──▶ CloudFront ──HTTP──▶ Caddy :80 ──▶ front :3000 ──▶ api :3001
                    (TLS grátis)         (instância)
```

| Montagem | Por quê / por que não |
|---|---|
| **CloudFront → Caddy (HTTP)** | ✅ **Escolhida.** Funciona no dia 1, **sem domínio próprio**. TLS válido de graça em `*.cloudfront.net`, cache na borda, e o IP de origem deixa de ser exposto |
| Caddy com Let's Encrypt direto | ❌ **Exige domínio.** E sem CloudFront a instância recebe tráfego bruto da internet |
| CloudFront → Caddy (HTTPS) | ❌ Melhor no fim, mas **exige domínio** (o CloudFront não aceita certificado self-signed na origem). É para onde se migra na Fase 7 |

**O salto CloudFront → origem é HTTP.** Isso não é aceitável sozinho, e por isso a fase inclui duas
proteções que andam juntas:

1. **Security group restrito à prefix list** `com.amazonaws.global.cloudfront.origin-facing` — só
   endereços do CloudFront alcançam a porta 80.
2. **Header secreto** `X-Origin-Verify` que o CloudFront injeta e o Caddy exige. Sem ele, a porta 80
   responde `403`.

> Por que as duas, e não só a prefix list: a prefix list libera **todo** o CloudFront, não só a *sua*
> distribuição. Sem o header, qualquer pessoa poderia criar uma distribuição apontando para o DNS da
> sua instância e servir seu app pelo domínio dela. O header é o que amarra a origem a **esta**
> distribuição.

### Por que o Caddy existe, já que o front atende HTTP

Pergunta legítima — o CloudFront poderia apontar direto para a porta 3000. O Caddy paga o próprio
custo (~20 MB de RAM) por três motivos:

- **É onde o header secreto é validado.** O Next.js não tem um lugar natural para isso, e enfiar essa
  responsabilidade no `proxy.ts` misturaria borda com aplicação.
- **Desacopla a borda do destino.** Trocar o que fica atrás (front, uma página de manutenção, um
  segundo serviço por path) vira mudança de configuração do Caddy, não da distribuição CloudFront —
  cuja propagação leva minutos.
- **É a peça que ganha TLS na Fase 7.** Quando o domínio existir, o Let's Encrypt entra no Caddy e o
  salto origem passa a ser HTTPS, sem mexer no front.

---

## 2. Etapa 0 — o Elastic IP

Primeiro passo porque **o CloudFront exige um nome DNS como origem — não aceita IP**. É o EIP que dá
esse nome estável (`ec2-<ip-com-tracos>.us-east-2.compute.amazonaws.com`). Sem ele, o IP público muda
a cada stop/start (já mudou uma vez, no resize de 20/08) e a distribuição quebra em silêncio.

```bash
aws ec2 allocate-address --region us-east-2 --domain vpc --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=cronos-eip}]' --query "{IP:PublicIp,Alloc:AllocationId}" --output table
```

**O que faz:** reserva um IPv4 público na conta, no escopo da VPC, já com a tag `Name` — sem ela, o
EIP vira uma linha anônima na fatura daqui a três meses. **Guarde o `Alloc`**, o próximo comando usa.

```bash
aws ec2 associate-address --region us-east-2 --instance-id i-0694b1330083a3450 --allocation-id <ALLOC_ID> --query "AssociationId" --output text
```

**O que faz:** associa o EIP à instância. No momento da associação o IP efêmero atual é **liberado e
perdido** — esperado. A partir daí o endereço sobrevive a stop/start.

```bash
aws ec2 describe-instances --region us-east-2 --instance-ids i-0694b1330083a3450 --query "Reservations[].Instances[].{IP:PublicIpAddress,DNS:PublicDnsName}" --output table
```

**O que faz:** revela o **nome DNS público** que o CloudFront vai usar como origem. Anote o valor de
`DNS` — ele aparece na §8. Se vier vazio, a VPC está sem `enableDnsHostnames` e é preciso ligar antes
de seguir.

---

## 3. Etapa 1 — espelhar a imagem do Caddy no ECR

Mesma decisão da Fase 4 (`api-aws.md` §1): o ECS puxa do ECR, não de registry de terceiro. Aqui o
motivo não é pacote privado — é **rate limit do Docker Hub**, que se manifesta como uma task que
simplesmente para de subir num dia qualquer, sem mudança nenhuma no seu lado.

```bash
aws ecr create-repository --region us-east-2 --repository-name caddy --image-scanning-configuration scanOnPush=true --query "repository.repositoryUri" --output text
```

**O que faz:** cria o repositório para a imagem espelhada.

```bash
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin <conta>.dkr.ecr.us-east-2.amazonaws.com
```

**O que faz:** token temporário do ECR entregue ao Docker por stdin (nunca por `-p`, que deixa a
senha no histórico e na lista de processos).

```bash
docker buildx imagetools create --tag <conta>.dkr.ecr.us-east-2.amazonaws.com/caddy:2-alpine docker.io/library/caddy:2-alpine
```

**O que faz:** copia a imagem oficial do Caddy registry-a-registry, sem materializar nada na sua
máquina. A `caddy:2-alpine` é multi-arch e **inclui `linux/arm64`** — a cópia preserva o índice
inteiro, então o ECS escolhe a variante Graviton sozinho.

> Fixar `2-alpine` em vez de `latest` é deliberado: `latest` mudaria de major sem aviso, e a borda é
> o pior lugar para uma surpresa dessas.

---

## 4. Etapa 2 — o segredo do header

```bash
openssl rand -hex 32
```

**O que faz:** gera o valor do `X-Origin-Verify`. 32 bytes em hexadecimal é indistinguível de
aleatório para qualquer atacante. **Copie a saída** — ela vai em dois lugares (SSM e CloudFront) e
não deve ser digitada à mão.

```bash
aws ssm put-parameter --region us-east-2 --name /cronos/edge/origin-secret --type SecureString --value "<SEGREDO_GERADO>" --description "Header X-Origin-Verify exigido pelo Caddy e injetado pelo CloudFront"
```

**O que faz:** grava o segredo como `SecureString` (criptografado com a chave KMS gerenciada da
conta, grátis no tier Standard). Mesmo padrão de `/cronos/api/*` — segredo nunca entra em task
definition em texto plano nem em arquivo versionado.

### 4.1 Ampliar a permissão da execution role

A inline policy da `ecsTaskExecutionRole` hoje lista `/cronos/postgres/*` e `/cronos/api/*`. O
prefixo `/cronos/edge/*` é novo — **sem isso a task do Caddy falha ao subir**, com um erro sobre
`ResourceInitializationError` que não menciona a policy.

> **Detalhe que já mordeu neste projeto** (`api-aws.md` §3): `put-role-policy` com o mesmo
> `--policy-name` **substitui a policy inteira**, não acrescenta. O JSON abaixo repete os prefixos
> antigos de propósito — omiti-los quebraria a API e o Postgres.

```bash
aws iam put-role-policy --role-name ecsTaskExecutionRole --policy-name ReadCronosPostgresParams --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["ssm:GetParameters","ssm:GetParameter"],"Resource":["arn:aws:ssm:us-east-2:<conta>:parameter/cronos/postgres/*","arn:aws:ssm:us-east-2:<conta>:parameter/cronos/api/*","arn:aws:ssm:us-east-2:<conta>:parameter/cronos/edge/*"]},{"Effect":"Allow","Action":"kms:Decrypt","Resource":"*"}]}'
```

**O que faz:** reescreve a policy com os **três** prefixos. Confira antes de rodar que o conteúdo
bate com o que existe hoje (`aws iam get-role-policy --role-name ecsTaskExecutionRole --policy-name ReadCronosPostgresParams`)
— se a policy real divergir deste JSON, ajuste em vez de sobrescrever às cegas.

### 4.2 O usuário operacional não tinha permissão de IAM, nem de CloudFront

Na execução real, `aws iam get-role-policy` (e também `put-role-policy`) falhou com `AccessDenied`
para `gabrielmizael_ecs` — o usuário IAM que a CLI usa neste projeto desde a Fase 2. Ele tem
permissão pra EC2, ECS, ECR, SSM (escopado a `/cronos/*`) e Logs — tudo que as Fases 3-6 precisaram —
mas **nada de IAM**. Isso se repetiu, na mesma fase, pro **CloudFront**: `ListCachePolicies`,
`CreateDistribution` e depois `GetDistribution` (usado por baixo dos panos pelo
`wait distribution-deployed`) — todos negados, um de cada vez, conforme cada comando era tentado.

**Why:** é um desenho de menor privilégio coerente, não um acidente — um usuário operacional que
administra containers e logs mas não consegue editar IAM roles nem criar recursos de borda limita o
raio de um vazamento de credencial. O problema é que ele não foi *documentado* como tal até esbarrar
nele na prática, três vezes na mesma fase.

**A correção aplicada:** ampliar `gabrielmizael_ecs` com uma policy inline escopada — não
`cloudfront:*` nem `iam:*`, só as ações específicas usadas neste documento e as previstas pra Fase 7
(`UpdateDistribution`, já que o domínio vai exigir reconfigurar a origem):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudfront:CreateDistribution",
        "cloudfront:GetDistribution",
        "cloudfront:UpdateDistribution",
        "cloudfront:ListDistributions",
        "cloudfront:TagResource",
        "cloudfront:ListCachePolicies",
        "cloudfront:ListOriginRequestPolicies"
      ],
      "Resource": "*"
    }
  ]
}
```

A permissão de `iam:GetRolePolicy`/`iam:PutRolePolicy` **não foi concedida** — a ampliação da
`ReadCronosPostgresParams` (o `put-role-policy` acima) foi feita pelo console, com uma identidade
diferente. É uma escolha deliberada: editar `ecsTaskExecutionRole` continua exigindo trocar de
identidade, e isso é aceitável porque acontece raramente (foi a segunda vez desde a Fase 3).

**How to apply:** antes de assumir que qualquer comando `aws iam` ou `aws cloudfront` vai funcionar
com as credenciais já configuradas neste projeto, esperar `AccessDenied` como resultado mais provável
na primeira tentativa. Não é bug — é o desenho. A Fase 7 (domínio + `UpdateDistribution` no Caddy com
TLS) já está coberta pela policy acima; uma eventual Fase de observabilidade (CloudWatch alarms, §11.3)
provavelmente vai esbarrar no mesmo padrão com outro serviço.

---

## 5. Etapa 3 — o Caddyfile na instância

Via `aws ssm start-session --region us-east-2 --target i-0694b1330083a3450`:

```bash
sudo mkdir -p /etc/caddy && sudo tee /etc/caddy/Caddyfile > /dev/null << 'EOF'
{
	admin off
	auto_https off
}

:80 {
	# Sonda de liveness do ECS. Fica fora da checagem do header de propósito:
	# o health check precisa dizer "o Caddy está vivo", não "o CloudFront me
	# mandou algo válido".
	handle /_edge-health {
		respond "ok" 200
	}

	handle {
		# Só o CloudFront desta distribuição conhece o segredo. A prefix list no
		# Security Group já limita a origem ao CloudFront em geral; este header
		# amarra a *esta* distribuição — sem ele, qualquer um poderia criar uma
		# distribuição apontando pro DNS desta instância.
		@naoAutorizado not header X-Origin-Verify {env.ORIGIN_SECRET}
		respond @naoAutorizado "forbidden" 403

		# "front" é resolvido pelo extraHosts da task definition, apontando pro
		# gateway da bridge do Docker (172.17.0.1) — o mesmo mecanismo que o
		# front usa pra achar a API. Ver separacao-de-tasks-front-api.md.
		reverse_proxy front:3000
	}
}
EOF
```

**O que faz:** grava a configuração do Caddy no host, para ser montada no container. Os pontos que
importam:

- **`admin off`** desliga a API administrativa (porta 2019). Ela não é usada aqui e é superfície de
  ataque de graça.
- **`auto_https off`** impede o Caddy de tentar emitir certificado. Nesta montagem o TLS termina no
  CloudFront; o Caddy fala HTTP puro. Sem isso ele tentaria ACME e falharia em loop.
- **`{env.ORIGIN_SECRET}`** lê a variável de ambiente que a task definition injeta a partir do SSM —
  o segredo não fica neste arquivo.
- **`handle` em vez de matchers soltos** garante ordem determinística: a rota de health é avaliada
  antes da checagem do header.

```bash
sudo chmod 644 /etc/caddy/Caddyfile && cat /etc/caddy/Caddyfile
```

**O que faz:** garante que o arquivo é legível pelo processo do container e imprime o conteúdo para
conferência.

> **Dívida assumida, e é a mesma do volume EBS** (`banco-de-dados-aws.md` §12.3): este arquivo foi
> criado à mão e **não sobrevive à substituição da instância**. Se o ASG trocar a máquina, o Caddy
> sobe sem configuração. A correção são poucas linhas de user-data — está na §11.

---

## 6. Etapa 4 — log group e task definition

```bash
aws logs create-log-group --region us-east-2 --log-group-name /ecs/cronos-edge && aws logs put-retention-policy --region us-east-2 --log-group-name /ecs/cronos-edge --retention-in-days 7
```

**O que faz:** cria o grupo com retenção de 7 dias, **antes** da task definition — a lição da Fase 4
(`api-aws.md` §4): grupo criado automaticamente nasce sem retenção, guardando log para sempre.

Grave como `cronos-edge.json`, substituindo `<conta>`:

```json
{
  "family": "cronos-edge",
  "networkMode": "bridge",
  "requiresCompatibilities": ["EC2"],
  "executionRoleArn": "arn:aws:iam::<conta>:role/ecsTaskExecutionRole",
  "runtimePlatform": {
    "cpuArchitecture": "ARM64",
    "operatingSystemFamily": "LINUX"
  },
  "volumes": [
    {
      "name": "caddyfile",
      "host": { "sourcePath": "/etc/caddy/Caddyfile" }
    }
  ],
  "containerDefinitions": [
    {
      "name": "caddy",
      "image": "<conta>.dkr.ecr.us-east-2.amazonaws.com/caddy:2-alpine",
      "essential": true,
      "memory": 128,
      "memoryReservation": 64,
      "portMappings": [
        { "containerPort": 80, "hostPort": 80, "protocol": "tcp" }
      ],
      "extraHosts": [
        { "hostname": "front", "ipAddress": "172.17.0.1" }
      ],
      "mountPoints": [
        {
          "sourceVolume": "caddyfile",
          "containerPath": "/etc/caddy/Caddyfile",
          "readOnly": true
        }
      ],
      "secrets": [
        {
          "name": "ORIGIN_SECRET",
          "valueFrom": "arn:aws:ssm:us-east-2:<conta>:parameter/cronos/edge/origin-secret"
        }
      ],
      "readonlyRootFilesystem": true,
      "linuxParameters": {
        "tmpfs": [
          { "containerPath": "/data", "size": 16, "mountOptions": ["rw", "noexec", "nosuid"] },
          { "containerPath": "/config", "size": 16, "mountOptions": ["rw", "noexec", "nosuid"] }
        ]
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "wget -q -O /dev/null http://localhost/_edge-health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 15
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/cronos-edge",
          "awslogs-region": "us-east-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### 6.1 Os campos que merecem explicação

| Campo | Por quê |
|---|---|
| `family: cronos-edge` | Família própria. Mesma lógica da separação front/API: a borda tem ciclo de vida próprio e trocar a config do Caddy não deve tocar no front |
| `extraHosts: front → 172.17.0.1` | O mesmo mecanismo já validado na Fase 6 — o gateway da bridge do Docker, estável entre instâncias, ao contrário do IP privado |
| `volumes` + `mountPoints` | Bind mount do Caddyfile do host, montado **`readOnly: true`**: o Caddy não tem motivo para reescrever a própria configuração |
| `secrets` (não `environment`) | O segredo vem do SSM em tempo de execução, decifrado pela execution role. Nunca aparece na task definition, que é legível por qualquer um com `describe-task-definition` |
| `memory` 128 / `memoryReservation` 64 | O Caddy usa ~20 MB. O limite rígido segue o padrão do projeto: um vazamento aqui mata a borda, não o Postgres |
| `readonlyRootFilesystem` + tmpfs em `/data` e `/config` | Mesma proteção da API e do front. O Caddy grava autosave em `/config/caddy` mesmo com `admin off` — sem o tmpfs, ele falha ao iniciar |
| `startPeriod: 15` | O Caddy sobe em ~1 s. Janela curta é adequada e faz uma falha real aparecer rápido, ao contrário do front (60 s), que compila rotas no boot |
| Sem `dependsOn` | `dependsOn` só funciona entre containers da **mesma task**. O Caddy e o front estão em tasks diferentes — se o front não estiver de pé, o Caddy responde `502`, que é o comportamento correto de um proxy |

```bash
aws ecs register-task-definition --region us-east-2 --cli-input-json file://cronos-edge.json --query "taskDefinition.{Familia:family,Revisao:revision,Arquitetura:runtimePlatform.cpuArchitecture}" --output table
```

**O que faz:** registra `cronos-edge:1`. Confirme `ARM64` na saída.

---

## 7. Etapa 5 — Security group e service

### 7.1 Descobrir a prefix list do CloudFront

```bash
aws ec2 describe-managed-prefix-lists --region us-east-2 --filters "Name=prefix-list-name,Values=com.amazonaws.global.cloudfront.origin-facing" --query "PrefixLists[].{Id:PrefixListId,Nome:PrefixListName,MaxEntradas:MaxEntries}" --output table
```

**O que faz:** a AWS mantém uma prefix list gerenciada com **todos** os IPs de borda do CloudFront,
atualizada por ela. Referenciá-la é o que evita ter que manter uma lista de CIDRs à mão — que
envelheceria e quebraria sem aviso. Anote o `Id` (formato `pl-...`) e o `MaxEntradas`.

> **`MaxEntradas` conta contra o limite de regras do Security Group** (padrão: 60 por SG). Se essa
> prefix list tiver muitas entradas e o SG já estiver cheio, a regra é recusada. É um erro confuso
> quando acontece — vale saber antes.

### 7.2 Descobrir o Security Group da instância

```bash
aws ec2 describe-instances --region us-east-2 --instance-ids i-0694b1330083a3450 --query "Reservations[].Instances[].SecurityGroups[]" --output table
```

**O que faz:** lista os security groups da instância. Anote o `GroupId` (`sg-...`).

### 7.3 Abrir a porta 80 — só para o CloudFront

```bash
aws ec2 authorize-security-group-ingress --region us-east-2 --group-id <SG_ID> --ip-permissions 'IpProtocol=tcp,FromPort=80,ToPort=80,PrefixListIds=[{PrefixListId=<PL_ID>,Description="CloudFront origin-facing"}]'
```

**O que faz:** libera a porta 80 **exclusivamente** para os endereços do CloudFront. A porta nunca é
exposta a `0.0.0.0/0`, nem temporariamente para testes — a verificação da §8 é feita por dentro da
instância e pela URL do CloudFront, justamente para não precisar dessa janela.

### 7.4 Criar o service

```bash
aws ecs create-service --region us-east-2 --cluster ec2-sistema-despesas --service-name cronos-edge --task-definition cronos-edge --desired-count 1 --launch-type EC2 --deployment-configuration "minimumHealthyPercent=0,maximumPercent=100,deploymentCircuitBreaker={enable=true,rollback=true}" --query "service.{Servico:serviceName,Status:status,Desejado:desiredCount}" --output table
```

**O que faz:** mesmos parâmetros dos outros três services, pelos mesmos motivos: `launch-type EC2`
(não capacity provider, INFRA-01), `minimumHealthyPercent=0` (porta 80 fixa, instância única) e
disjuntor com reversão automática.

```bash
aws ecs wait services-stable --region us-east-2 --cluster ec2-sistema-despesas --services cronos-edge
```

### 7.5 Verificar o Caddy antes do CloudFront

Na sessão SSM, antes de criar a distribuição — é muito mais barato depurar aqui do que esperar
propagação de CDN:

```bash
curl -s -o /dev/null -w "health=%{http_code}\n" http://localhost/_edge-health && curl -s -o /dev/null -w "sem_header=%{http_code}\n" http://localhost/ && curl -s -o /dev/null -w "com_header=%{http_code}\n" -H "X-Origin-Verify: <SEGREDO>" http://localhost/
```

**O que faz:** exercita os três caminhos do Caddyfile de uma vez. Esperado: `health=200` (sonda
viva), `sem_header=403` (a proteção funciona) e `com_header=200` (o proxy para o front funciona). Um
`com_header=502` significa que o Caddy não alcançou `front:3000` — confira o `extraHosts`.

---

## 8. Etapa 6 — a distribuição CloudFront

Grave como `cloudfront.json`, substituindo o DNS da §2 e o segredo da §4:

```json
{
  "CallerReference": "cronos-2026-08-20",
  "Comment": "CRONOS - borda HTTPS para a instancia ECS",
  "Enabled": true,
  "HttpVersion": "http2and3",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "cronos-origin",
        "DomainName": "<DNS_PUBLICO_DO_EIP>",
        "CustomOriginConfig": {
          "HTTPPort": 80,
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "http-only",
          "OriginSslProtocols": { "Quantity": 1, "Items": ["TLSv1.2"] },
          "OriginReadTimeout": 30,
          "OriginKeepaliveTimeout": 5
        },
        "CustomHeaders": {
          "Quantity": 1,
          "Items": [
            {
              "HeaderName": "X-Origin-Verify",
              "HeaderValue": "<SEGREDO>"
            }
          ]
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "cronos-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 7,
      "Items": ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"],
      "CachedMethods": { "Quantity": 2, "Items": ["GET", "HEAD"] }
    },
    "Compress": true,
    "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
    "OriginRequestPolicyId": "216adef6-5c7f-47e4-b989-5492eafa07d3"
  },
  "CacheBehaviors": {
    "Quantity": 1,
    "Items": [
      {
        "PathPattern": "/_next/static/*",
        "TargetOriginId": "cronos-origin",
        "ViewerProtocolPolicy": "redirect-to-https",
        "AllowedMethods": {
          "Quantity": 2,
          "Items": ["GET", "HEAD"],
          "CachedMethods": { "Quantity": 2, "Items": ["GET", "HEAD"] }
        },
        "Compress": true,
        "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
      }
    ]
  },
  "PriceClass": "PriceClass_100"
}
```

### 8.1 As decisões embutidas nesse JSON

| Campo | Escolha | Por quê |
|---|---|---|
| `OriginProtocolPolicy` | `http-only` | O salto para a origem é HTTP — é a montagem 1 da §1. Vira `https-only` na Fase 7, com domínio |
| `CustomHeaders` | `X-Origin-Verify` | O CloudFront injeta em **toda** requisição à origem. O viewer não consegue sobrescrever: o CloudFront descarta um header de mesmo nome vindo do cliente |
| `ViewerProtocolPolicy` | `redirect-to-https` | HTTP do visitante vira redirect 301 para HTTPS. **Os cookies de sessão são `secure` em produção** (`arquitetura-aws.md` §1.5) — sem HTTPS o login simplesmente não funciona |
| `AllowedMethods` | os 7 | O app faz `POST`/`PATCH`/`DELETE` via Server Actions e pelo proxy `/api/*`. Só `GET`/`HEAD` quebraria todo formulário |
| `CachePolicyId` (default) | **CachingDisabled** | O front é SSR e depende de cookies de sessão. Cachear a resposta padrão serviria **a página de um usuário para outro** — é a falha de segurança clássica de CDN na frente de app autenticado |
| `CacheBehaviors` `/_next/static/*` | **CachingOptimized** | Esses arquivos têm hash no nome e são imutáveis por construção. É aqui que o CDN paga o próprio custo, sem risco de vazar sessão |
| `OriginRequestPolicyId` | **AllViewer** | Encaminha headers, cookies e query string à origem. Sem cookies, o `proxy.ts` e o `getCurrentUser()` nunca enxergariam a sessão |
| `PriceClass_100` | EUA/Europa | Reduz custo excluindo bordas caras. Como o free tier cobre 1 TB, é mais higiene que economia — **e vale revisitar se o público for Brasil**, já que exclui a borda de São Paulo |
| `Compress` | `true` | Gzip/Brotli na borda, de graça |

> **Confirme os IDs das políticas gerenciadas** antes de aplicar — são estáveis, mas não custa:
> `aws cloudfront list-cache-policies --type managed --query "CachePolicyList.Items[].{Nome:CachePolicy.CachePolicyConfig.Name,Id:CachePolicy.Id}" --output table`

```bash
aws cloudfront create-distribution --distribution-config file://cloudfront.json --query "Distribution.{Id:Id,Dominio:DomainName,Status:Status}" --output table
```

**O que faz:** cria a distribuição. **Anote o `Dominio`** (`d<algo>.cloudfront.net`) — é a URL pública
do sistema. O `Status` sai como `InProgress`: a propagação leva de 5 a 15 minutos, e testar antes
disso dá erros que não significam nada.

```bash
aws cloudfront wait distribution-deployed --id <DISTRIBUTION_ID>
```

**O que faz:** bloqueia até o status virar `Deployed`. Evita a rodada de testes prematuros.

---

## 9. Verificação

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://<DOMINIO>.cloudfront.net
```

**O que faz:** o teste de ponta a ponta. Esperado `200`, com certificado TLS válido (o `curl` falharia
com erro de certificado se não fosse). Isso prova a cadeia inteira: CloudFront → Caddy → front.

```bash
curl -s -o /dev/null -w "%{http_code} -> %{redirect_url}\n" http://<DOMINIO>.cloudfront.net
```

**O que faz:** confirma o redirect de HTTP para HTTPS. Esperado: `301` apontando para o `https://`.

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://<DOMINIO>.cloudfront.net/login
```

**O que faz:** exercita uma página que passa pelo `layout.tsx` → `getCurrentUser()` → API. Esperado
`200`. Um `502` indicaria o front fora; um `500`, problema de `API_URL`.

```bash
curl -s -I https://<DOMINIO>.cloudfront.net/_next/static/ -o /dev/null -w "%{http_code}\n"
```

**O que faz:** verifica que o comportamento de cache adicional está roteando. O código em si importa
menos que a ausência de erro de configuração.

**E o teste que fecha a segurança**, do seu terminal local (fora da AWS):

```bash
curl -s -m 10 -o /dev/null -w "%{http_code}\n" http://<IP_DO_EIP>/ ; echo "timeout/erro acima = correto"
```

**O que faz:** tenta acessar a instância **diretamente**, sem passar pelo CloudFront. O esperado é
**timeout ou conexão recusada** — prova de que a prefix list está fazendo seu trabalho. Se responder
qualquer coisa, a regra do Security Group está aberta demais e precisa ser revista antes de considerar
a fase concluída.

E, no navegador: abrir `https://<DOMINIO>.cloudfront.net`, fazer login de verdade e navegar até o
dashboard. É o único teste que exercita cookie `secure`, `Set-Cookie` atravessando o CloudFront e o
refresh do `proxy.ts` — nenhum `curl` cobre isso de forma convincente.

---

## 10. Rollback

```bash
aws ecs update-service --region us-east-2 --cluster ec2-sistema-despesas --service cronos-edge --desired-count 0
```

**O que faz:** derruba o Caddy, liberando a porta 80. Front, API e banco continuam de pé — só deixam
de ser alcançáveis de fora.

```bash
aws ec2 revoke-security-group-ingress --region us-east-2 --group-id <SG_ID> --ip-permissions 'IpProtocol=tcp,FromPort=80,ToPort=80,PrefixListIds=[{PrefixListId=<PL_ID>}]'
```

**O que faz:** fecha a porta 80 de volta.

Para a distribuição CloudFront, o caminho é `update-distribution` com `"Enabled": false` e depois
`delete-distribution` — a AWS **exige desabilitar e aguardar** antes de permitir a exclusão, e o ciclo
completo leva ~15 minutos. Desabilitar já basta para interromper o acesso.

O Elastic IP: **não libere no rollback**. Ele é o único recurso desta fase que custa dinheiro parado, e
liberá-lo significa perder o endereço para sempre — e o IP público que o substituiria seria efêmero de
novo.

---

## 11. Pendências a resolver

Em ordem de importância. Nenhuma bloqueia a conclusão da fase.

### 11.1 O Caddyfile não sobrevive à substituição da instância

O arquivo `/etc/caddy/Caddyfile` foi criado à mão (§5). Se o ASG trocar a instância, o bind mount
aponta para um caminho inexistente e o Caddy sobe sem configuração — a borda cai e o sintoma não
menciona arquivo nenhum.

**É a mesma classe da pendência nº 3 de [`banco-de-dados-aws.md`](./banco-de-dados-aws.md)** (o mount
do EBS), e a correção é a mesma: user-data no launch template escrevendo o arquivo no boot, antes do
agente ECS subir. Vale resolver as duas juntas, num único user-data.

**Alternativa mais limpa:** buildar uma imagem própria do Caddy com o Caddyfile embutido e publicá-la
no ECR. Elimina o bind mount, mas cria um artefato novo para versionar e um pipeline para mantê-lo.

### 11.2 `FRONTEND_URL` na API ainda é placeholder

A task da API tem `FRONTEND_URL=http://localhost:3000` (`api-aws.md` §13.5). Ele alimenta o
`Access-Control-Allow-Origin` com `credentials: true` e o redirect final do OAuth.

**Hoje não quebra nada:** o front faz proxy same-origin, então o navegador nunca emite requisição
cross-origin para a API. Passa a importar quando o Google login for ligado.

Atualizar exige **revisão nova da `cronos-app` e reinício da API** (~30 s). Como o valor definitivo
será o domínio próprio da Fase 7, a recomendação é **esperar** e trocar uma vez só, em vez de duas.

> Não é uma revisão isolada: quando ela acontecer, vale entrar junto com as 4 variáveis do Google
> OAuth e as 5 do SMTP (§11.8) — todas moram na mesma task definition e todas dependem do domínio.

### 11.3 Nenhum alarme, nenhuma observabilidade da borda

Se o CloudFront começar a servir `502`, ninguém fica sabendo. O `arquitetura-aws.md` §12 já
recomendava alarme no CloudWatch → SNS → e-mail, e dez alarmes por mês são grátis. Os dois que valem
mais aqui: taxa de erro 5xx da distribuição, e o service `cronos-edge` com `runningCount` em zero.

### 11.4 O `PriceClass_100` exclui a borda de São Paulo

Se o público real for brasileiro, a latência sofre — o tráfego atravessa para os EUA. Trocar para
`PriceClass_All` é uma linha, e o free tier de 1 TB continua valendo. **Decisão adiada** por ser
trivialmente reversível.

### 11.5 O espelhamento no ECR continua manual

Agora são **três** imagens (API, front, Caddy). A automação é a Fase 8 (`api-aws.md` §13.6). A do
Caddy é a menos urgente — ela só muda quando você decidir atualizar a versão.

### 11.6 Sem WAF, sem rate limiting

A borda aceita qualquer volume de requisições. O AWS WAF resolveria, mas custa ~US$ 5-8/mês (US$ 5 por
web ACL + US$ 1 por regra), o que é significativo num orçamento de ~US$ 25. **Fora de escopo
conscientemente** — vale registrar que a decisão foi tomada, não esquecida.

### 11.7 O sistema fica exposto sem qualquer proteção de cadastro

Com a URL pública no ar, qualquer pessoa pode criar conta. Não é um problema técnico desta fase, mas é
uma decisão de produto que passa a existir a partir dela.

### 11.8 Google OAuth e SMTP continuam sem configuração

Duas funcionalidades **já implementadas no código** estão inertes em produção por falta de variáveis
de ambiente na task `cronos-app` (`api-aws.md` §5). O efeito visível para o usuário:

- **O botão "entrar com Google" não funciona.** `googleAuthEnabled` é `false`, então a rota nem é
  registrada.
- **A recuperação de senha completa o fluxo sem enviar email.** `mailEnabled` é `false`: a pessoa pede
  a redefinição, recebe a resposta de sucesso, e nada chega na caixa — o link só é registrado em log.
  É a degradação mais traiçoeira das duas, porque quem testa só a resposta HTTP conclui que funcionou.

O [`env.ts`](../../sistema-controle-despesas-api/src/config/env.ts) valida cada grupo como **"tudo ou
nada"**: com todas ausentes a API sobe e degrada graciosamente; com **algumas** preenchidas ela
**recusa-se a subir**. Não preencha pela metade — o erro aparece como task que nunca fica `healthy`.

| Grupo | Variáveis | Onde |
|---|---|---|
| Google OAuth | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `COOKIE_SESSION_SECRET` | `secrets` (SSM) |
| | `GOOGLE_CALLBACK_URL` | texto plano — **depende do domínio da §11.2/Fase 7** |
| SMTP | `SMTP_PASSWORD` | `secrets` (SSM) |
| | `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `MAIL_FROM` | texto plano |

**Por que esperar a Fase 7:** o `GOOGLE_CALLBACK_URL` tem de ser a URL pública real e estar registrada
no Google Cloud Console — trocar de domínio depois obriga a mexer nos dois lugares. E `MAIL_FROM` num
domínio próprio tem entregabilidade melhor do que num `*.cloudfront.net`, que muitos filtros tratam
com desconfiança.

**Detalhe que morde na borda:** `/api/auth/google` é uma **navegação do navegador**, não um `fetch` — a
API responde com `302` para o Google. Hoje isso atravessa o rewrite do Next; se o Route Handler da
§11.9 entrar antes, ele precisa **repassar o `Location`** em vez de seguir o redirect.

### 11.9 O rewrite `/api/*` do front ainda é resolvido em build-time

Fora do escopo desta fase, mas é o vizinho direto dela: o front resolve `/api/*` por um rewrite que o
Next congela na imagem, então **a imagem carrega a topologia da rede**. Isso quebrou produção em
20/08/2026 e foi corrigido no valor, não no mecanismo — ver
[`problema-rewrite-api-build-time.md`](../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md).

Importa aqui porque a **Fase 7 muda o endereço público**, que é exatamente o gatilho do bug. A saída é
o Route Handler (Abordagem B), que unifica os três consumidores de `API_URL` em runtime.

> Vale registrar por que a **Abordagem C — o Caddy roteando `/api/*` direto para a API** — foi
> rejeitada, já que o Caddy é justamente a peça desta fase e a ideia reaparece naturalmente: em
> desenvolvimento e no `docker-compose.yml` do e2e **não existe Caddy**. Produção passaria a exercitar
> um caminho que os testes nunca tocam, e um bug de proxy só apareceria depois do deploy. Somado à
> §11.1 (o `Caddyfile` não é versionado e não sobrevive à troca da instância), colocar roteamento
> crítico de aplicação nele aumentaria a dívida em vez de reduzi-la.

### 11.10 A porta da API (`3001`) confunde com a do front (`3000`)

Um dígito de diferença, em `curl`, log, Security Group e task definition. Padronizar a API em `8080`
elimina a ambiguidade. **Não há ganho técnico** — é legibilidade operacional, e por isso não tem
urgência nem risco associado a adiar.

Custo levantado: ~~6 arquivos de código~~ (5 no repo da API, o `docker-compose.yml` deste — **os 6 já
usam `8080`**, ver [`docker-e-arquitetura.md`](./docker-e-arquitetura.md) §1 e §3) e 3 mudanças de
infra — regra do Security Group, revisão da `cronos-app`, revisão da `cronos-front` — **ainda
pendentes**, feitas à mão no console AWS. O repo do front não tem **nenhum** arquivo a mudar: só o
valor da Repository Variable `API_URL`, também pendente. Detalhamento em
[`problema-rewrite-api-build-time.md`](../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md)
§14.4.

**A ordem importa**, porque a porta muda dos dois lados: Security Group liberando **as duas** portas →
API → front → remover a regra da `3001`. Sem isso há uma janela em que o front aponta para a porta
velha.

---

## 12. Referências

- [`arquitetura-aws.md`](./arquitetura-aws.md) — §1.5 (cookies `secure`), §10 (rede, ingresso e TLS), §11 (domínio), §12 (boas práticas), §13 (roteiro)
- [`front-aws.md`](./front-aws.md) — a Fase 6, de onde vêm o `extraHosts` e o padrão de task definition
- [`api-aws.md`](./api-aws.md) — §1 (por que ECR), §3 (a armadilha do `put-role-policy`), §4 (log group antes), §8 (por que não há ALB), §13 (pendências)
- [`banco-de-dados-aws.md`](./banco-de-dados-aws.md) — §12.3 (a pendência do mount manual, irmã da §11.1)
- [`separacao-de-tasks-front-api.md`](./separacao-de-tasks-front-api.md) — a decisão de uma task por unidade implantável, que este documento estende à borda
