# Banco de dados na AWS — CRONOS

Registro de **como o Postgres foi provisionado na AWS**, o que cada peça faz e por que cada decisão
foi tomada dessa forma. Corresponde à **Fase 3** do roteiro de [`arquitetura-aws.md`](./arquitetura-aws.md) §13.

> **Estado:** provisionado e verificado em 19/08/2026. Postgres 17 rodando como service ECS,
> dados em volume EBS dedicado, snapshot diário automático.
> **Método:** tudo criado **manualmente** (AWS CLI + console), por decisão deliberada — ver §1.
> **Região:** `us-east-2` (Ohio). Diverge da recomendação original do documento de arquitetura; ver §2.
> **Instância:** `t4g.micro` — Graviton (**ARM64**), 2 vCPU, **1 GB de RAM**. Também diverge do plano
> (`t3.small`, x86, 2 GB); as consequências estão em [`arquitetura-aws.md`](./arquitetura-aws.md) §-1.
> A mais importante: **toda task definition deste projeto precisa ser `linux/arm64`.**
> **Escopo:** volume EBS, credenciais, rede, IAM, task definition, service e backup.
> **Convenção:** ARNs neste documento usam `<conta>` no lugar do ID real da conta AWS. O repositório
> é público, e ID de conta é insumo de enumeração e engenharia social — não é segredo, mas também
> não precisa estar indexado no GitHub.

---

## 0. O que ficou de pé

| Peça | Recurso | Identificação |
|---|---|---|
| Disco de dados | EBS gp3, 10 GB | tag `Name=cronos-postgres-data` |
| Ponto de montagem | `/mnt/cronos-postgres`, dados em `pgdata/` | `fstab` por UUID, com `nofail` |
| Credenciais | 3 parâmetros no SSM Parameter Store | `/cronos/postgres/{user,password,db}` |
| Rede | Regra de entrada 5432 *self-reference* | SG da instância |
| Permissão | `ecsTaskExecutionRole` | policy gerenciada + inline `ReadCronosPostgresParams` |
| Definição | Task definition `cronos-data` | revisão `:2` (a `:1` falhou — ver §8) |
| Execução | Service ECS `cronos-data` | cluster `ec2-sistema-despesas`, `desiredCount=1` |
| Logs | CloudWatch Logs, retenção 7 dias | `/ecs/cronos-data` |
| Backup | Política DLM, snapshot diário 03:00 UTC, retenção 7 | `policy-05c9…` |

Custo somado desta fase: **~US$ 1,30/mês** (US$ 0,80 do volume + ~US$ 0,50 dos snapshots). O compute
é US$ 0 marginal — o Postgres divide a instância que já estava paga.

---

## 1. Por que manualmente, e não em Terraform

O documento de arquitetura é enfático que **o entregável de portfólio é o Terraform, não a instância
ligada** (§0). Esta fase contraria isso de propósito, e vale registrar o motivo para que a decisão
não pareça descuido:

> **O objetivo desta rodada é aprender AWS, não entregar infraestrutura reproduzível.**
> Provisionar à mão obriga a encontrar cada peça, ler cada erro e entender por que cada permissão
> existe — conhecimento que o `terraform apply` esconde justamente por ser bom no que faz.

O plano continua sendo migrar para Terraform (e depois Kubernetes) **depois** que o sistema inteiro
estiver no ar manualmente. Isso tem um custo real e conhecido: nada aqui é reproduzível hoje, e uma
instância perdida significa refazer tudo lendo este documento. É um débito aceito de olhos abertos,
e este documento é a mitigação parcial dele.

**O que isso implica na prática, e que a §12 detalha:** algumas coisas foram feitas de um jeito que
funciona hoje mas não sobrevive à automação (o mount manual é o exemplo central). Elas estão marcadas
como pendências, não como implementação final.

---

## 2. A região mudou: `us-east-2`, não `us-east-1`

O documento de arquitetura recomendava `us-east-1` (§5), com a justificativa de ser a região mais
barata da AWS. A infraestrutura acabou provisionada em **`us-east-2` (Ohio)**, e a decisão é para
ficar.

**Por que não é um problema.** Os "20% mais caro" que o documento cita eram a comparação com
`sa-east-1` (São Paulo), não entre regiões americanas. `us-east-1` e `us-east-2` têm preço
praticamente idêntico nos serviços deste desenho — EC2, EBS, snapshot. Migrar agora custaria refazer
tudo para economizar aproximadamente nada.

**Por que precisa estar escrito.** Quase todo serviço da AWS é **regional**, e isso morde silenciosamente:

- Um parâmetro criado no SSM de `us-east-1` é **invisível** para uma task rodando em `us-east-2` —
  o container falharia ao subir por não achar a secret, sem nenhuma mensagem indicando que o problema
  é geográfico.
- `aws ssm start-session` contra a região errada devolve `TargetNotConnected`, que sugere problema de
  agente ou de rede — e não "essa instância não existe aqui". Foi exatamente o que aconteceu na
  primeira tentativa de conexão.

> **Regra operacional:** todo comando da AWS CLI neste projeto leva `--region us-east-2` explícito, ou
> o default do perfil está fixado nela (`aws configure set region us-east-2`). IAM é a exceção — é um
> serviço global e não aceita `--region`.

---

## 3. O volume EBS: por que separado do disco raiz

O volume de dados é um EBS **gp3 de 10 GB, criado à parte** do volume raiz de 30 GB da instância.

**Por que não usar o disco raiz, que tem 28 GB livres.** Porque ele morre com a instância. No desenho
com Auto Scaling Group (mesmo com `min=max=1`), substituir uma instância não saudável é uma operação
normal e automática — e o volume raiz da instância antiga é descartado junto. Com o banco no disco
raiz, uma auto-recuperação de rotina viraria perda total dos dados. Um volume separado sobrevive à
instância e pode ser reanexado.

**Por que gp3 e não gp2.** O gp3 entrega 3.000 IOPS e 125 MB/s de baseline em **qualquer tamanho**,
enquanto o gp2 amarra desempenho ao tamanho do disco (3 IOPS/GB — um volume de 10 GB teria 30 IOPS de
baseline). Além disso o gp3 é ~20% mais barato por GB. Não há cenário em que o gp2 ganhe aqui.

**Por que 10 GB.** É o piso confortável: o app não usa 1 GB tão cedo, e crescer um volume EBS depois é
uma operação online (`modify-volume` + `resize2fs`), sem downtime. Vale contrastar com o RDS, cujo
mínimo é 20 GB — metade do que se pagaria lá seria piso comercial, não necessidade.

### O detalhe que decide o dia ruim: a AZ

Um volume EBS **vive dentro de uma única Availability Zone** e só pode ser anexado a instâncias da
mesma AZ. Isso tem uma consequência arquitetural que não é óbvia:

> Enquanto o banco morar neste volume, **o ASG está preso a uma AZ**. Não é possível deixá-lo
> distribuir instâncias entre AZs — uma instância que subisse na AZ errada não conseguiria anexar o
> volume, e o service `data` nunca ficaria saudável.

Isso não é uma perda nova: o Cenário C já assumia Single-AZ conscientemente (§7 do documento de
arquitetura). Mas a razão técnica de estar preso passa a ser esta, concretamente.

---

## 4. Formatação e montagem: as três decisões

Depois de anexado, o volume aparece como `/dev/nvme1n1` — **não** como `/dev/sdf`, que foi o nome
pedido no `attach-volume`. Instâncias Nitro renomeiam os dispositivos; confiar no nome do
`attach-volume` é um erro comum.

> **Sempre confirme com `lsblk` antes de formatar.** O disco raiz aparece na mesma lista, com
> partições montadas em `/` e `/boot/efi`. `mkfs` no dispositivo errado destrói a instância.

**ext4.** É o padrão do Amazon Linux 2023, tem o melhor suporte de ferramentas e é o que o Postgres
espera encontrar. Não há aqui o volume de escrita que justificaria avaliar XFS.

**Montagem por UUID no `/etc/fstab`, não por nome de dispositivo.** A linha gravada:

```
UUID=a65c7117-…  /mnt/cronos-postgres  ext4  defaults,nofail  0  2
```

Dois porquês, ambos aprendidos de erros alheios:

- **UUID e não `/dev/nvme1n1`**: a numeração dos dispositivos NVMe depende da ordem em que os volumes
  são detectados no boot. Anexar um segundo volume no futuro pode renomear este — e o `fstab`
  montaria o disco errado no lugar certo. O UUID é gravado no sistema de arquivos e não muda.
- **`nofail`**: sem essa opção, um `fstab` apontando para um volume ausente **trava o boot da
  instância**. Com ela, a instância sobe e o problema aparece como um service ECS não saudável — que
  é diagnosticável remotamente, ao contrário de uma instância que não termina de iniciar.

**`mount -a` como verificação.** Testar a linha do `fstab` com `mount -a` (que relê o arquivo e monta o
que falta) é o que transforma um erro de sintaxe em uma mensagem imediata, em vez de uma instância que
não volta no próximo reboot — o pior momento possível para descobrir.

---

## 5. Credenciais no SSM Parameter Store

Os três valores que a imagem do Postgres precisa vivem no Parameter Store, não na task definition:

| Parâmetro | Tipo | Por quê |
|---|---|---|
| `/cronos/postgres/user` | `String` | Não é segredo — `postgres` está no `docker-compose.yml` versionado |
| `/cronos/postgres/password` | `SecureString` | Criptografado com a chave KMS `aws/ssm` |
| `/cronos/postgres/db` | `String` | Nome do banco, também não é segredo |

**Por que isso e não `environment` na task definition.** É o **SEC-03** da
[revisão de segurança da API](../../sistema-controle-despesas-api/docs/revisao-seguranca-deploy-aws.md):
variável de ambiente em task definition é **texto plano**, legível por qualquer pessoa com permissão
de leitura no ECS — inclusive no console, inclusive em `describe-task-definition`, inclusive para quem
só deveria poder olhar a configuração. O bloco `secrets` resolve isso: a task definition guarda apenas
o **ARN do parâmetro**, e o valor real só é resolvido em runtime, pelo agente ECS.

**Por que a senha foi gerada com `openssl rand -base64 24` e nunca digitada.** Senha inventada por
humano é o elo fraco previsível. Como nada além do Postgres precisa conhecê-la — a API vai lê-la do
mesmo Parameter Store —, não há motivo para ela existir em lugar nenhum fora do KMS.

**Por que `SecureString` só na senha.** Cada `SecureString` custa uma chamada de descriptografia ao
KMS e uma permissão a mais na policy. Usuário e nome do banco não são segredos (estão no repositório
público), e marcá-los como secretos daria a falsa impressão de que são.

**Por que o prefixo `/cronos/postgres/`.** O Parameter Store trata `/` como hierarquia. Isso permite
`get-parameters-by-path --path /cronos/postgres` para listar tudo de uma vez e, mais importante,
permite escrever a policy IAM com `parameter/cronos/postgres/*` — dando à execution role acesso a
exatamente esses três parâmetros e a nenhum outro da conta.

**Tier Standard, que é gratuito.** O Secrets Manager faria a mesma coisa com rotação automática, por
US$ 0,40 por segredo por mês. Para uma senha que não rotaciona, é pagar por um recurso não usado.

---

## 6. Rede: a regra *self-reference*

A porta 5432 foi liberada no Security Group da instância com origem no **próprio Security Group**, e
não em um bloco CIDR:

```
Protocolo TCP, porta 5432, origem: sg-00ef… (o mesmo grupo)
```

**O que isso significa.** Só recursos que carregam esse mesmo Security Group podem falar com a 5432.
Como só a instância `ec2-sistema-despesas` o carrega, na prática o banco só aceita conexão de dentro
dela — que é exatamente o caminho que o service `app` (Fase 4) vai usar, pelo IP privado da instância.

**Por que não CIDR.** É o **INFRA-03/04** da revisão de segurança — banco alcançável da internet é
severidade crítica. Mas o ponto mais sutil é que *nem um CIDR privado é a resposta certa*: a
referência a Security Group é **dinâmica** (vale para qualquer instância que receba aquele SG, hoje ou
depois), enquanto um CIDR é uma aposta sobre a topologia de rede que envelhece — se a VPC mudar de
faixa, ou se outro recurso qualquer nascer dentro da faixa liberada, a regra continua valendo sem que
ninguém tenha decidido isso.

### O `0.0.0.0:5432` do `docker ps` não é o que parece

A verificação mostrou:

```
0.0.0.0:5432->5432/tcp, :::5432->5432/tcp
```

Isso assusta à primeira vista, mas é apenas o Docker escutando em todas as interfaces **do host**.
Quem decide se um pacote vindo de fora chega até ali é o Security Group, que está fechado. As duas
camadas são independentes, e a de rede é a que vale.

> **Inconsistência conhecida, a corrigir na Fase 4:** a regra da porta 3001 (API) usa
> `CidrIp: 172.31.0.0/16` em vez de self-reference. Funciona e não expõe nada à internet — é a faixa
> privada da VPC —, mas é o padrão que este documento acabou de argumentar contra. Deve ser trocada
> quando o service `app` for criado.

---

## 7. IAM: três roles, três papéis distintos

Um dos pontos que o documento de arquitetura levanta (§12) é que quase todo tutorial funde roles que
deveriam ser separadas. O que existe hoje:

| Role | Quem assume | Para quê |
|---|---|---|
| `ecsInstanceRole` | a instância EC2 | registrar-se no cluster ECS (já existia, da Fase 2) |
| `ecsTaskExecutionRole` | o agente ECS (`ecs-tasks.amazonaws.com`) | puxar a imagem, escrever logs, **ler as secrets** |
| `cronos-dlm-role` | o serviço DLM (`dlm.amazonaws.com`) | criar e expirar snapshots |

Não existe **task role** — e isso é deliberado. A task role é o que a *aplicação* pode chamar na AWS
em runtime; o Postgres não chama nada. Criar uma role vazia "por precaução" só adiciona uma superfície
para alguém anexar permissão depois sem pensar.

### A execution role e as duas permissões que andam juntas

Além da policy gerenciada `AmazonECSTaskExecutionRolePolicy` (puxar imagem + escrever logs), foi
anexada uma policy **inline** com duas permissões:

```json
{ "Action": "ssm:GetParameters", "Resource": "arn:aws:ssm:us-east-2:<conta>:parameter/cronos/postgres/*" },
{ "Action": "kms:Decrypt",       "Resource": "arn:aws:kms:us-east-2:<conta>:key/alias/aws/ssm" }
```

**As duas são necessárias, e a segunda é a que se esquece.** `ssm:GetParameters` sozinha permite *pedir*
o parâmetro; como a senha é `SecureString`, o valor volta cifrado e inútil sem `kms:Decrypt` na chave
que o cifrou. O sintoma dessa falta é um container que sobe com uma senha que parece lixo — não um
`AccessDenied` óbvio.

**Por que inline e não policy gerenciada por você.** Uma policy gerenciada faz sentido quando é
reutilizada por várias identidades. Esta é específica de uma role e de três parâmetros; inline mantém
as duas coisas coladas e impede que alguém a anexe em outro lugar por engano.

**Por que o `Resource` é restrito.** `parameter/cronos/postgres/*` em vez de `parameter/*` significa que
um comprometimento dessa role não abre os segredos de mais nada na conta — presente ou futuro.

### A jornada das permissões do usuário IAM

Provisionar isso exigiu ampliar as permissões do próprio usuário `gabrielmizael_ecs`, e o processo
ensinou mais que o resultado:

**`iam:PassRole` é a permissão que não parece necessária até falhar.** Criar uma role e *entregá-la a
um serviço* são autorizações diferentes. Sem `PassRole`, tudo funciona até o
`register-task-definition`, que falha com `AccessDenied` — depois de a role já existir, o que confunde
o diagnóstico.

**O Access Analyzer estava certo sobre o curinga.** A primeira versão da policy usava
`Resource: "role/ecs*"`. O aviso da AWS foi seguido e o ARN passou a ser exato, mais a condição:

```json
"Condition": { "StringEquals": { "iam:PassedToService": "ecs-tasks.amazonaws.com" } }
```

O risco real que isso fecha: `PassRole` com curinga permite entregar **qualquer** role que bata o
padrão a **qualquer** serviço. Se amanhã nascer uma role poderosa cujo nome comece com `ecs`, ela
entraria no escopo sem ninguém decidir isso. A condição amarra cada `PassRole` ao serviço que
legitimamente a usa.

**A exceção do curinga, e por que ela é aceitável.** A policy do DLM usa
`Resource: "arn:aws:dlm:us-east-2:<conta>:policy/*"`. O ID de uma política DLM é gerado pela AWS **no
momento da criação** — não existe antes para ser nomeado. O que continua restringindo é a conta e a
região no ARN. Curinga sobre um ID futuro é diferente de curinga sobre escopo.

---

## 8. A task definition, e o erro que ela precisou aprender

A definição registrada (`cronos-data`), campo a campo relevante:

| Campo | Valor | Justificativa |
|---|---|---|
| `networkMode` | `bridge` | §10 do doc de arquitetura: sem ENI extra, sem IP por task, custo zero |
| `requiresCompatibilities` | `["EC2"]` | Não é Fargate — roda na instância que já está paga |
| `image` | `postgres:17-alpine` | **Mesma versão do `docker-compose.yml`** — é a que o E2E já exercita |
| `memory` | `384` (limite **rígido**) | Ver abaixo |
| `portMappings` | `5432:5432` | Porta fixa: é como o service `app` vai encontrar o banco |
| `secrets` | 3 ARNs do SSM | SEC-03, ver §5 |
| `healthCheck` | `pg_isready -U postgres` | O ECS precisa saber a diferença entre "processo vivo" e "aceita conexão" |
| `logConfiguration` | `awslogs` → `/ecs/cronos-data` | Sem isso, o log morre junto com a instância |

**Por que `memory` (rígido) e não `memoryReservation` (flexível), ao contrário dos outros containers.**
No ECS sobre EC2, o scheduler só coloca uma task se houver memória **não reservada** suficiente. A
recomendação geral (§6 do doc de arquitetura) é usar `memoryReservation` na maioria dos containers,
para que a soma dos limites não impeça o deploy. O Postgres é a exceção deliberada: ele é o único
container com estado, e o único cuja morte por OOM causa dano real. Um limite rígido garante que ele
nunca seja espremido por um pico do front ou da API.

**Por que 384 MB.** A estimativa do documento de arquitetura é 200–300 MB com `shared_buffers=128MB`;
384 dá folga para picos.

> ⚠️ **Isso é mais apertado do que parecia quando foi escolhido.** A instância é uma **`t4g.micro` com
> 1 GB de RAM**, não a `t3.small` de 2 GB do plano original (ver `arquitetura-aws.md` §-1). Com
> ~250 MB do sistema operacional e do agente ECS, os 384 MB rígidos do Postgres consomem uma fatia
> grande do que resta. Cabem ainda a API (~150–250 MB) e o Caddy (~20 MB); **o front não cabe**.
>
> O limite rígido continua sendo a escolha certa — numa instância com pouca RAM, garantir que o banco
> nunca seja espremido importa **mais**, não menos. Mas isso torna o swap (ainda não configurado) um
> requisito, e não uma recomendação.

**Retenção de 7 dias no log group.** O padrão do CloudWatch é **nunca expirar**, e é assim que ele vira
uma linha crescente e inexplicável na fatura (§3 do doc de arquitetura, armadilha 4). Sete dias cobrem
qualquer diagnóstico que se faça de verdade.

### O erro: `initdb` recusa a raiz de um mount point

A revisão `:1` da task definition apontava o volume direto para `/mnt/cronos-postgres`. O resultado foi
um **crash loop** — task nova a cada ~10 segundos, `runningCount: 0`, `exitCode: 1` sem `reason`.

O log do CloudWatch entregou a causa exata:

```
initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
initdb: detail: It contains a lost+found directory, perhaps due to it being a mount point.
initdb: hint: Using a mount point directly as the data directory is not recommended.
```

**O que estava acontecendo.** `mkfs.ext4` cria automaticamente um diretório `lost+found` na raiz de
todo sistema de arquivos. Para o `initdb`, um diretório de dados "não vazio" é sinal de que ele pode
estar prestes a inicializar por cima de algo — e ele se recusa a continuar em vez de arriscar.

**A correção:** criar `pgdata/` **dentro** do ponto de montagem e apontar o `sourcePath` para ela.
A subpasta nasce genuinamente vazia; o volume continua sendo o mesmo.

```
/mnt/cronos-postgres/          ← ponto de montagem (contém lost+found)
└── pgdata/                    ← sourcePath do bind mount, diretório de dados do Postgres
```

> **Três lições que valem mais que a correção**, e é por isso que o erro está documentado em vez de
> apagado:
>
> 1. **`exitCode: 1` sem `reason` significa "a aplicação decidiu sair"** — não é falha de permissão,
>    de imagem ou de rede (essas aparecem como `CannotPullContainerError` e afins). O diagnóstico
>    está no log da aplicação, não no `describe-tasks`.
> 2. **O log group configurado desde o início foi o que tornou isso um problema de cinco minutos.**
>    Sem `awslogs` na task definition, a saída do container morre com o container, e restaria adivinhar.
> 3. **Nunca monte um volume EBS diretamente como diretório de dados** de um serviço com estado —
>    Postgres, MySQL, Elasticsearch, todos rejeitam pelo mesmo motivo. A subpasta é o padrão.

---

## 9. O service, e por que `minimumHealthyPercent=0`

```
--desired-count 1
--deployment-configuration "minimumHealthyPercent=0,maximumPercent=100"
```

**`desiredCount=1`, e não pode ser outro número.** Duas razões independentes, cada uma suficiente:
duas instâncias de Postgres escrevendo no mesmo volume corromperiam os dados, e a porta 5432 do host
só pode ser ocupada por um container.

**`minimumHealthyPercent=0` é contraintuitivo e obrigatório aqui.** O padrão (100) diz ao ECS: *"suba
a task nova e confirme que está saudável antes de derrubar a antiga"*. Isso é o comportamento correto
em qualquer ambiente com folga — e é impossível neste: com uma única instância e uma porta fixa, não
existe onde a task nova rodar enquanto a antiga vive. Com o padrão, todo deploy travaria para sempre.

O preço disso está registrado no documento de arquitetura e é aceito: **alguns segundos de
indisponibilidade a cada deploy**.

**Por que `data` é um service separado de `app`.** Esta é a decisão que justifica toda a estrutura: se
Postgres, API e front estivessem na mesma task, **todo deploy da aplicação reiniciaria o banco**.
Separados, o service `app` pode ser reimplantado quantas vezes por dia for necessário enquanto o
`data` — que quase nunca muda — segue de pé.

**`--task-definition cronos-data` sem número de revisão.** Omitir o `:N` faz o ECS resolver para a
revisão mais recente do family no momento da chamada. Evita a classe de erro em que se registra uma
revisão nova e se esquece de apontar o service para ela.

---

## 10. Verificação: o teste que realmente importa

`pg_isready` respondendo e o healthcheck em `healthy` provam que o Postgres subiu. Não provam a única
coisa que esta fase existe para garantir: **que o dado está no volume EBS, e não na camada gravável do
container**. Os dois cenários são indistinguíveis enquanto nada reinicia.

O teste feito:

1. `CREATE TABLE teste_persistencia (...)` no banco;
2. `update-service --force-new-deployment` — o ECS destrói o container e cria outro;
3. `\dt` no container **novo** → a tabela continua lá.

Se o bind mount estivesse errado (apontando para o lugar errado, ou ausente), o passo 3 devolveria uma
lista vazia. É a diferença entre "o banco funciona" e "o banco persiste".

> **O que este teste ainda não cobriu:** restaurar um snapshot do DLM. O documento de arquitetura é
> direto sobre isso — *"backup nunca restaurado é hipótese, não backup"* (§8). Está na §12.

---

## 11. Backup: o que o DLM cobre e o que não cobre

A política criada (`policy-05c9…`) tira um snapshot do volume todo dia às **03:00 UTC** (meia-noite em
Brasília) e mantém os **7 mais recentes**, descartando o mais antigo automaticamente.

**Como ele encontra o volume: por tag, não por ID.** `TargetTags: Name=cronos-postgres-data`. A
consequência é útil e vale saber: qualquer volume futuro com essa mesma tag entra na política sozinho.
E a consequência inversa é o risco: **remover ou renomear a tag desliga o backup silenciosamente** —
nada falha, os snapshots simplesmente param de existir.

**Custo:** o serviço DLM é gratuito; paga-se apenas o armazenamento dos snapshots, a US$ 0,05/GB-mês.
Como snapshots são **incrementais** (só blocos alterados desde o anterior), a estimativa fica em
~US$ 0,50/mês, não em 7 × 10 GB.

### O limite que define a próxima tarefa

> **Snapshot de volume com o Postgres escrevendo é consistente-em-crash, não
> consistente-em-aplicação.** Equivale a puxar a tomada da instância naquele instante.

O Postgres foi projetado para se recuperar disso — é para isso que o WAL existe — e na prática quase
sempre se recupera. Mas "quase sempre" é uma garantia diferente da que um backup deveria dar.

Por isso o DLM é a **primeira** camada, não a única. A segunda, planejada para a Fase 9, é um
`pg_dump` diário para o S3, rodando como ECS Scheduled Task (mesmo padrão da purga de tokens do
SEC-09). Um dump é feito *através* do Postgres, então é consistente por construção — e é o backup que
se usa numa restauração de verdade.

| | Snapshot DLM | `pg_dump` para S3 |
|---|---|---|
| Consistência | Em crash | Em aplicação |
| Recupera de | Disco perdido, instância destruída | O mesmo, mais corrupção lógica e erro humano |
| Granularidade | Volume inteiro | Banco, tabela, linha |
| Restauração | Criar volume novo a partir do snapshot | `psql < dump.sql` |
| Estado | ✅ feito | ⏳ Fase 9 |

---

## 12. Pendências conhecidas

Em ordem de importância. Nenhuma bloqueia a Fase 4.

1. **`pg_dump` diário para o S3** (Fase 9). É o backup que se usa de verdade — ver §11.
2. **Restaurar um backup de propósito, uma vez.** Medir quanto tempo levou (RTO) e quanto dado se
   perderia (RPO), e anotar as duas linhas. Até que isso aconteça, o backup é uma hipótese.
3. **O mount não sobrevive à troca da instância.** Hoje o `attach-volume` e o `mount` foram feitos à
   mão. Se o ASG substituir a instância, o volume fica órfão e o service `data` nunca sobe. A correção
   são ~10 linhas de user-data reanexando o volume por ID e montando-o **antes** do agente ECS subir.
   É a pendência mais perigosa desta lista, porque só se manifesta no dia em que algo já deu errado.
4. **A regra da porta 3001 usa CIDR em vez de self-reference** (§6). Trocar ao criar o service `app`.
5. **Nada disso é reproduzível.** É a consequência aceita da §1, e a razão de este documento existir
   com este nível de detalhe.

---

## 13. Referências

- [`arquitetura-aws.md`](./arquitetura-aws.md) — a decisão de desenho que esta fase implementa (§7-C, §8, §13)
- [`docker-e-arquitetura.md`](./docker-e-arquitetura.md) — o estado dos três repositórios
- [`revisao-seguranca-deploy-aws.md`](../../sistema-controle-despesas-api/docs/revisao-seguranca-deploy-aws.md) — SEC-03 (secrets), INFRA-03/04 (rede), INFRA-05 (IAM)
- [Amazon EBS — preços](https://aws.amazon.com/ebs/pricing/)
- [Data Lifecycle Manager — documentação](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-lifecycle.html)
- [Especificar recursos sensíveis em `iam:PassRole`](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_passrole.html)
