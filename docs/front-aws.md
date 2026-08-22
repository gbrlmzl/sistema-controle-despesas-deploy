# Front na AWS — CRONOS

Plano de execução da **Fase 6** do roteiro de [`arquitetura-aws.md`](./arquitetura-aws.md) §13, na
topologia de **tasks separadas** decidida em
[`separacao-de-tasks-front-api.md`](./separacao-de-tasks-front-api.md) (cenário B).

> **Estado:** implementado e verificado em 20/08/2026. Service `cronos-front` no ar na revisão
> **`cronos-front:2`** — a `:1` falhou o health check por um bug real, ver §5.1. RSS real medido —
> ver §10.1.
> **Incidente posterior (20-21/08/2026):** o front subiu e renderizava, mas **toda chamada do
> navegador falhava** — a Repository Variable `API_URL` da §7.1, classificada aqui como "higiene, não
> bloqueante", estava ausente e o rewrite `/api/*` ficou congelado no placeholder. Corrigido sem
> mudança de código nem revisão nova de task definition; ver §7.1 e o item 7 da §10.
> **Decisão que ele implementa:** front em task e service próprios, não como container extra da
> `cronos-app` — deploy e crash do front deixam de derrubar a API.
> **Pré-requisitos já cumpridos:** instância em `t4g.small` com 2 GB e swap; imagem `linux/arm64`
> publicada no GHCR e espelhada no ECR como `sistema-despesas-front:latest`.
> **Convenção:** o ID da conta aparece como `<conta>` — o repositório é público. Substitua ao rodar.

---

## 0. O que muda e o que não muda

A propriedade mais importante deste plano é o tamanho do que ele **não** toca:

| Peça | Muda? |
|---|---|
| Task definition `cronos-app` (a API) | ❌ **nenhuma alteração, nenhuma revisão nova** |
| Service `cronos-app` | ❌ intocado — a API não sai do ar em momento algum |
| Task definition / service `cronos-data` (Postgres) | ❌ intocado |
| Security groups | ❌ nenhuma regra nova (ver §1.2) |
| Imagem do front no ECR | ❌ já espelhada, é a que será usada |
| CI do front | ❌ nada obrigatório (uma recomendação opcional na §7) |
| **Log group `/ecs/cronos-front`** | ✅ **criar** |
| **Task definition `cronos-front`** | ✅ **criar** (família nova) |
| **Service `cronos-front`** | ✅ **criar** |

Três recursos novos, zero mudança destrutiva. Se algo der errado, o rollback é apagar o que foi
criado — a API e o banco continuam de pé o tempo todo.

**Custo desta fase: US$ 0,00.** O ECS não cobra por task nem por service no launch type EC2, a
instância já está paga, e a imagem já está no ECR (~90 MB, dentro do que já se paga por
armazenamento).

---

## 1. Etapa 0 — validar a premissa antes de construir

O cenário B se apoia numa afirmação que **ainda não foi verificada nesta instância**: que um container
em modo `bridge` alcança as portas publicadas no host pelo gateway `172.17.0.1`. Se isso for falso, o
resto do plano não funciona — e é melhor descobrir agora, em dois comandos, do que depois de registrar
task definition e service.

### 1.1 Confirmar o IP do gateway

Dentro da instância (via `aws ssm start-session`):

```bash
sudo docker network inspect bridge --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'
```

**O que faz:** lê o gateway da rede `bridge` padrão do Docker — a rede onde o ECS coloca os containers
em `networkMode: bridge`. O esperado é `172.17.0.1`. Se vier outro valor (alguém configurou `bip` no
daemon), **use o valor que apareceu** no lugar de `172.17.0.1` em todo o resto deste documento.

### 1.2 Provar que a porta 8080 responde por ali

```bash
sudo docker exec $(sudo docker ps --filter "name=ecs-cronos-app" --format "{{.Names}}") node -e "fetch('http://172.17.0.1:8080/health').then(r=>r.text()).then(t=>console.log('ALCANCOU:',t)).catch(e=>console.log('FALHOU:',e.message))"
```

**O que faz:** entra no container da API (que já está na bridge e já tem Node) e faz ele chamar a
**própria API pelo caminho que o front vai usar** — saindo do container, batendo no gateway, voltando
pela porta publicada no host. Se imprimir `ALCANCOU: {"status":"ok"}`, o cenário B está validado. Se
imprimir `FALHOU`, pare aqui: o endereçamento precisa ser repensado (Service Connect, §3.3 do
documento de decisão) antes de qualquer outra etapa.

> **Por que isso não precisa de regra no Security Group:** o tráfego container → `172.17.0.1` não sai
> pela ENI da instância — é roteado internamente pelo `docker0` via `iptables`. Security groups
> filtram tráfego de ENI; este nunca chega lá. É por isso que a §0 diz que nenhuma regra nova é
> necessária. Se o teste acima falhar, **não** comece abrindo portas no SG: o problema seria outro.

---

## 2. A escolha que torna o endereço legível: `extraHosts`

O documento de decisão (§8) propôs pôr `API_URL=http://172.17.0.1:8080` (`3001` até 21/08/2026) direto na variável de
ambiente, e registrou como custo aceito que "o `172.17.0.1` é opaco e exige comentário". **Há uma
saída melhor, e ela sai de graça.**

A task definition aceita o campo `extraHosts`, que escreve linhas no `/etc/hosts` do container:

```json
"extraHosts": [{ "hostname": "api", "ipAddress": "172.17.0.1" }]
```

Com isso, `api` volta a ser um nome resolvível dentro do container do front — e o valor da variável
continua sendo exatamente o mesmo de sempre:

```
API_URL=http://api:8080
```

Três ganhos, nenhum custo:

1. **A variável de ambiente não muda** em relação ao plano de task única. Quem ler a task definition vê
   `http://api:8080`, que é auto-explicativo; o mapeamento fica num campo próprio, comentado.
2. **O valor congelado em build-time continua correto.** O rewrite `/api/*` embutido na imagem aponta
   para `http://api:8080` (a Repository Variable do CI — `3001` até esse repositório também mudar, ver `ingresso-aws.md` §11.10). Com `extraHosts`, esse valor volta a resolver
   — o que fecha a dívida datada que a §2 do documento de decisão registrava para quando o Google
   login for ligado.
3. **A migração futura para Service Connect fica trivial:** troca-se o `extraHosts` por
   `serviceConnectConfiguration`, e a variável de ambiente segue intocada.

> `extraHosts` **não é suportado** em `networkMode: awsvpc`. Como este desenho usa `bridge` (decisão da
> §10 do documento de arquitetura), não há conflito. Vale saber caso `awsvpc` volte à mesa.

---

## 3. Etapa 1 — criar o log group

**Antes** de registrar a task definition. A Fase 4 aprendeu isso do jeito difícil: a opção
`awslogs-create-group` teve que sair da configuração (`api-aws.md` §4), então o grupo precisa existir
previamente ou a task falha ao subir com um erro que não menciona logs.

```bash
aws logs create-log-group --region us-east-2 --log-group-name /ecs/cronos-front
```

**O que faz:** cria o grupo de logs no CloudWatch. Sem ele, o driver `awslogs` da task falha na
criação do container.

```bash
aws logs put-retention-policy --region us-east-2 --log-group-name /ecs/cronos-front --retention-in-days 7
```

**O que faz:** define retenção de 7 dias. Sem política de retenção, o CloudWatch guarda **para
sempre** e cobra por GB armazenado indefinidamente — é o vazamento de custo silencioso mais comum em
projeto pequeno. Sete dias é o mesmo padrão de `/ecs/cronos-app` e `/ecs/cronos-data`.

---

## 4. Etapa 2 — a task definition `cronos-front`

Grave o JSON abaixo como `cronos-front.json`, **substituindo `<conta>` pelo ID real da conta**.

```json
{
  "family": "cronos-front",
  "networkMode": "bridge",
  "requiresCompatibilities": ["EC2"],
  "executionRoleArn": "arn:aws:iam::<conta>:role/ecsTaskExecutionRole",
  "runtimePlatform": {
    "cpuArchitecture": "ARM64",
    "operatingSystemFamily": "LINUX"
  },
  "containerDefinitions": [
    {
      "name": "front",
      "image": "<conta>.dkr.ecr.us-east-2.amazonaws.com/sistema-despesas-front:latest",
      "essential": true,
      "memory": 512,
      "memoryReservation": 320,
      "portMappings": [
        { "containerPort": 3000, "hostPort": 3000, "protocol": "tcp" }
      ],
      "extraHosts": [
        { "hostname": "api", "ipAddress": "172.17.0.1" }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "PORT", "value": "3000" },
        { "name": "API_URL", "value": "http://api:8080" },
        { "name": "NEXT_TELEMETRY_DISABLED", "value": "1" }
      ],
      "readonlyRootFilesystem": true,
      "linuxParameters": {
        "tmpfs": [
          {
            "containerPath": "/app/.next/cache",
            "size": 64,
            "mountOptions": ["rw", "noexec", "nosuid", "uid=1000", "gid=1000"]
          },
          {
            "containerPath": "/tmp",
            "size": 16,
            "mountOptions": ["rw", "noexec", "nosuid", "uid=1000", "gid=1000"]
          }
        ]
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "node -e \"fetch('http://localhost:3000').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))\""],
        "interval": 30,
        "timeout": 12,
        "retries": 3,
        "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/cronos-front",
          "awslogs-region": "us-east-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### 4.1 Campo a campo, e o porquê de cada um

| Campo | Valor | Justificativa |
|---|---|---|
| `family` | `cronos-front` | Família **própria**, separada de `cronos-app`. É o que permite versionar e implantar o front sozinho |
| `runtimePlatform` | `ARM64` / `LINUX` | A instância é Graviton. Task amd64 aqui morre com `exec format error` — e a imagem publicada é arm64 puro |
| `networkMode` | `bridge` | Mesma decisão da §10 do documento de arquitetura, e o que faz `extraHosts` funcionar |
| CPU/memória **de tarefa** | ausentes | Mesmo raciocínio de `api-aws.md` §6: no launch type EC2 são opcionais, e o controle fica no nível do container. O limite rígido do container satisfaz a validação |
| `executionRoleArn` | `ecsTaskExecutionRole` | Puxa a imagem do ECR. **Já tem a permissão** — a policy gerenciada cobre `ecr:GetAuthorizationToken` e `ecr:BatchGetImage` desde a Fase 3 |
| Task role | **ausente** | O front não chama nenhuma API da AWS em runtime. Mesmo INFRA-05 aplicado à API |
| `memory` / `memoryReservation` | 512 / 320 | Limite rígido pelo mesmo motivo da API (`api-aws.md` §6): um vazamento no front mata o front, não o Postgres. Ver o orçamento na §4.2 |
| `portMappings` | `3000:3000` | Porta fixa no host, como a API faz com 8080. Não conflita |
| `extraHosts` | `api → 172.17.0.1` | O coração do cenário B — ver §2 |
| `readonlyRootFilesystem` | `true` | Mesma proteção da API (`api-aws.md` §7): uma RCE não consegue gravar payload. Exige os `tmpfs` abaixo |
| `linuxParameters.tmpfs` | `.next/cache` (64 MiB), `/tmp` (16 MiB) | O Next **escreve** cache de imagem/fetch em `.next/cache` — é exatamente o caso que o §7 da API previa ("montar um `tmpfs` só no diretório que precisa, mantendo o resto travado") |
| `mountOptions: uid=1000,gid=1000` | | **Detalhe que quebra em silêncio se faltar:** o container roda como `USER node` (uid 1000), e um `tmpfs` sem dono explícito monta como `root`. O Next falharia ao escrever no cache que acabamos de montar para ele |
| `healthCheck` | `/` a cada 30 s, timeout 12 s | Ver §5 — o timeout tem um motivo específico e não óbvio |
| `startPeriod` | 60 s | Janela de graça no boot. Generosa de propósito: falha de healthcheck durante a partida dispara o disjuntor de implantação e mascara a causa real |

### 4.2 O orçamento de memória depois desta fase

A instância registra **1846 MB** no ECS (medido em 20/08/2026).

```
t4g.small — 1846 MB registrados
├── postgres   384 MB (rígido, Fase 3)
├── api        448 MB (rígido, Fase 4)
├── front      512 MB (rígido, esta fase)
└── ~502 MB    livres — dos quais ~118 MB reservados para o caddy da Fase 5
```

Sobra folga real (~384 MB depois do Caddy), e há 2 GB de swap como rede de segurança. Vale registrar
que **os 512 MB são uma estimativa**: ninguém mediu o RSS do `node server.js` com build `standalone`
em produção. A §8 traz o comando para medir depois que subir.

### 4.3 Registrar

```bash
aws ecs register-task-definition --region us-east-2 --cli-input-json file://cronos-front.json --query "taskDefinition.{Familia:family,Revisao:revision,Arquitetura:runtimePlatform.cpuArchitecture}" --output table
```

**O que faz:** registra a revisão `cronos-front:1`. `--cli-input-json file://` evita o inferno de
aspas de passar JSON inline no shell. Confirme na saída que `Arquitetura` é `ARM64` — se vier vazio ou
`X86_64`, o `runtimePlatform` não foi aceito e a task morreria com `exec format error`.

---

## 5. O healthcheck, e a armadilha do timeout

Este ponto veio de uma leitura do código do front e **não é óbvio na task definition**.

O healthcheck bate em `/`. Duas perguntas importam: essa rota toca a API? E o que acontece se a API
estiver fora?

**O `proxy.ts` não roda em `/`.** O matcher dele é
`["/dashboard/:path*", "/profile/:path*", "/login", "/register", "/forgot-password"]` — a raiz está
fora. Então o healthcheck **não** dispara o `POST /auth/refresh`. Bom sinal.

**Mas o `layout.tsx` roda.** O layout raiz chama `getCurrentUser()`, que chama `GET /users/me` na API
— em **toda** página, inclusive `/`. A boa notícia é que `src/lib/session.ts` trata isso
explicitamente: qualquer falha (401, API fora do ar, API lenta) cai para "deslogado" em vez de
derrubar a renderização. A página responde **200 mesmo com a API morta**.

O risco sobra num cenário estreito e real:

> Se a API estiver **no ar mas travada** (aceitando conexão e não respondendo), o `apiFetch` espera
> até o timeout dele — **10 segundos** (`TIMEOUT_MS` em `src/lib/apiClient.ts`). O render de `/` fica
> pendurado esse tempo todo. Com o healthcheck em `timeout: 5` (o valor do `HEALTHCHECK` do
> Dockerfile), a sonda desiste antes, falha 3 vezes seguidas e **o ECS reinicia o front** — por um
> problema que é da API.

É exatamente a armadilha liveness/readiness que `api-aws.md` §9 descreve, chegando por outro caminho.

**A mitigação adotada: `timeout: 12` na task definition**, acima dos 10 s do `apiFetch`. Assim o
render sempre termina (degradando para "deslogado") antes de a sonda desistir, e o front só é
reiniciado quando ele próprio está travado. O `healthCheck` da task definition **prevalece** sobre o
`HEALTHCHECK` do Dockerfile, então não é preciso mexer na imagem.

> **A correção limpa**, se um dia valer a pena: um Route Handler em `src/app/healthz/route.ts`
> retornando 200 seco. Route Handlers **não executam o layout raiz**, então a sonda deixaria de
> depender da API por completo. É mudança no repositório do front e não bloqueia nada aqui.

### 5.1 O erro real que a `:1` teve, e o que ele ensinou

A primeira revisão (`cronos-front:1`) subiu, rodou por ~2min44s (`startPeriod` 60s + 3 tentativas ×
`interval` 30s) e foi morta com `exitCode 143` (SIGTERM do próprio ECS) — o disjuntor de implantação
funcionou exatamente como desenhado, mas não havia revisão anterior pra reverter, então o service
ficou em `0` tasks. Log da aplicação limpo, sem um único erro: `✓ Ready in 0ms` em toda tentativa. A
task nunca reportou `healthy`.

**A causa não estava na task definition nem na AWS — estava numa consequência não documentada da
migração para `output: 'standalone'`, feita antes da Fase 6 (commit `96c7438`).** O `server.js`
gerado pelo standalone resolve o endereço de escuta assim (Next 16,
`node_modules/next/dist/build/utils.js:1125`):

```js
const hostname = process.env.HOSTNAME || '0.0.0.0'
```

O `next start` (usado antes do `standalone`, via `npm start`) **ignora** `HOSTNAME` e sempre escuta em
`0.0.0.0`. O `server.js` do standalone **respeita** essa variável — e o Docker define `HOSTNAME`
automaticamente como o ID do container em todo container criado. O `|| '0.0.0.0'` nunca entra em ação
dentro de um container: o servidor faz bind só no IP da bridge (`172.17.0.x`), e qualquer sonda via
`localhost` — o `HEALTHCHECK` do Dockerfile, o healthcheck do `docker-compose.yml`, e o `healthCheck`
desta task definition — é recusada por conexão.

Isso explica por que o passo de verificação da §8 tinha passado: o `curl localhost:3000` rodou **do
host**, não de dentro do container, e a porta publicada faz DNAT direto para o container — mascarando
o problema até o primeiro healthcheck real, que roda de dentro.

**Correção aplicada nos dois lugares:**

1. **Na task definition (`cronos-front:2`, imediata):** `HOSTNAME=0.0.0.0` somado ao `environment`.
   Sobrescreve o valor que o Docker injeta, sem exigir rebuild da imagem.
2. **No `Dockerfile` do front (durável, para compose local e e2e):**
   ```dockerfile
   ENV HOSTNAME=0.0.0.0
   ```
   Sem essa segunda correção, o `HEALTHCHECK` da própria imagem continua quebrado — o que também
   quebraria o `docker compose up --wait` do repositório de deploy (o serviço `front` espera esse
   healthcheck) e o e2e que depende dele.

**A lição a generalizar:** trocar `next start` por `node server.js` (a consequência do `standalone`
documentada em `arquitetura-aws.md` §9) não é só uma troca de comando — o binário gerado tem
comportamento de bind diferente, sensível a uma variável que o Docker define por padrão. Qualquer
migração futura de `next start` para o `server.js` do standalone, em qualquer repositório Next deste
projeto, carrega essa mesma pegadinha.

---

## 6. Etapa 3 — o service `cronos-front`

```bash
aws ecs create-service --region us-east-2 --cluster ec2-sistema-despesas --service-name cronos-front --task-definition cronos-front --desired-count 1 --launch-type EC2 --deployment-configuration "minimumHealthyPercent=0,maximumPercent=100,deploymentCircuitBreaker={enable=true,rollback=true}" --query "service.{Servico:serviceName,Status:status,Desejado:desiredCount}" --output table
```

**O que faz:** cria o service que mantém uma task do front rodando. Os parâmetros, e por que cada um:

- **`--launch-type EC2`**, e não estratégia de capacity provider: um capacity provider pode estar
  ligado ao *managed scaling* do ASG e **criar instâncias sozinho**. É o INFRA-01 já invocado na Fase
  4 — teto de compute fixo é a proteção estrutural contra susto de fatura. `launch-type` só posiciona
  no que já existe e falha de forma visível se não couber.
- **`minimumHealthyPercent=0` / `maximumPercent=100`**: a porta 3000 é fixa e a instância é uma só, então
  a task antiga precisa sair antes de a nova entrar. O padrão (`100`) travaria o deploy para sempre —
  mesma razão de `cronos-app` e `cronos-data`.
- **Disjuntor com `rollback=true`**: depois de N falhas consecutivas o ECS **para de tentar** e volta
  para a revisão anterior. Com `desiredCount=1` o piso efetivo é 3 falhas, como o console avisa.
- **Sem load balancer**: nenhum ALB, pelos motivos de `api-aws.md` §8. A entrada HTTPS continua sendo
  CloudFront → Caddy, na Fase 5.
- **Sem Service Auto Scaling e sem ECS Exec**: mesmas decisões da Fase 4. Para entrar num container, o
  caminho é `ssm start-session` + `sudo docker exec`.

```bash
aws ecs wait services-stable --region us-east-2 --cluster ec2-sistema-despesas --services cronos-front
```

**O que faz:** bloqueia até o service alcançar steady state. Retorna em silêncio se der certo; se
estourar, o diagnóstico está na §8.

---

## 7. Etapa 4 — mudanças fora da AWS

> ⚠️ **Esta seção continha um erro de análise que derrubou o sistema em produção.** O texto original
> classificava as duas mudanças abaixo como "higiene, nenhuma bloqueante". A §7.1 era, na verdade,
> **bloqueante** — deixá-la para depois quebrou toda a interatividade do app. O diagnóstico completo
> está em
> [`problema-rewrite-api-build-time.md`](../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md).
> Corrigido em 21/08/2026; o erro está preservado abaixo porque é instrutivo.

### 7.1 Repository Variable `API_URL` no repositório do front — **bloqueante**

O CI passa `API_URL` como `--build-arg`, congelando o destino do rewrite `/api/*` na imagem. Com o
`extraHosts` da §2, o valor correto é `http://api:8080`. Confirme em *Settings → Secrets and variables
→ Actions → Variables*.

Se estiver ausente, o CI cai no placeholder `http://localhost:8080` e **o rewrite fica quebrado dentro
do container** — `localhost:8080` não é nada ali. O efeito não é cosmético: **toda chamada feita pelo
navegador falha**, enquanto tudo que roda no servidor continua funcionando. O sintoma é uma página que
renderiza autenticada, com o cabeçalho certo, mas cujo corpo mostra "Erro na comunicação com o
servidor" — apontando para a API quando o problema está no front.

> ✅ **Configurada em 21/08/2026**, depois de o sistema quebrar exatamente assim em produção.

**Por que a análise original errou** (e o que aprender com isso): ela concluiu que o rewrite tinha "um
único consumidor, o link do Google login, que está desligado". A busca foi feita por `fetch("/api` e
pelo literal `"/api/"`, e não encontrou
[`src/lib/apiClient.client.ts`](../../sistema-controle-despesas-front/src/lib/apiClient.client.ts),
que monta a URL por **concatenação**:

```ts
const BASE_URL = "/api";
fetch(`${BASE_URL}${path}`, ...)
```

São **9 hooks** e 8 rotas da API por trás disso — praticamente toda a interatividade do app depois do
primeiro render. **Ao verificar dependências de um rewrite, procure pela construção da URL, não só
pelo literal.**

### 7.2 O `README.md` deste repositório

Ele afirma que buildar o front do fonte é "a única forma de o proxy funcionar de verdade dentro do
compose". **A afirmação está correta e deve ser mantida** — a revisão sugerida aqui originalmente
partia da mesma análise errada da §7.1. Com o rewrite congelado em build-time e a imagem publicada no
GHCR carregando o placeholder, buildar do fonte com `API_URL=http://api:8080` é de fato a única forma
de o `/api/*` funcionar dentro do compose.

Isso deixa de valer quando a Abordagem B (Route Handler em runtime) for implementada — aí o compose
passa a poder consumir a imagem publicada, alinhando com o "build once, promote everywhere" do resto
do projeto. Ver §10.7.

---

## 8. Etapa 5 — verificação

O `steady state` prova que o container não morreu. **Não prova que o front fala com a API.** Os testes
que fecham isso, de dentro da instância:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000
```

**O que faz:** confirma que o front responde na porta publicada. Espera-se `200`.

```bash
sudo docker exec $(sudo docker ps --filter "name=ecs-cronos-front" --format "{{.Names}}") node -e "fetch('http://api:8080/health').then(r=>r.text()).then(t=>console.log('FRONT->API OK:',t)).catch(e=>console.log('FALHOU:',e.message))"
```

**O que faz:** é **o teste que importa nesta fase**. Entra no container do front e resolve o nome
`api` — provando de uma vez que o `extraHosts` foi aplicado, que o gateway responde e que a porta 8080
está acessível. Se falhar com `getaddrinfo`, o `extraHosts` não pegou; se falhar com `ECONNREFUSED`, o
gateway está errado (volte à §1.1).

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/login
```

**O que faz:** exercita o caminho completo — essa página passa pelo `layout.tsx`, que chama
`getCurrentUser()` → `GET /users/me` na API. Espera-se `200` (renderizado como deslogado, já que o
curl não manda cookie). Um `500` aqui indica `API_URL` ausente ou irresolvível no runtime.

```bash
sudo docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}"
```

**O que faz:** mede o consumo real de memória dos containers. **É a medição que nunca foi feita**: os
512 MB do front são estimativa herdada de um chute anterior ao build `standalone`. Anote o valor — ele
permite ajustar o limite na próxima revisão e fecha a dúvida registrada em
`arquitetura-aws.md` §-1.

```bash
aws ecs describe-container-instances --region us-east-2 --cluster ec2-sistema-despesas --container-instances $(aws ecs list-container-instances --region us-east-2 --cluster ec2-sistema-despesas --query "containerInstanceArns[]" --output text) --query "containerInstances[].{Registrada:registeredResources[?name=='MEMORY'].integerValue|[0],Livre:remainingResources[?name=='MEMORY'].integerValue|[0]}" --output table
```

**O que faz:** confirma quanta memória sobrou no nível do cluster depois das três tasks. `Livre` deve
ficar em torno de 500 MB — o espaço reservado para o Caddy da Fase 5.

### 8.1 Se a task não subir

Na ordem de probabilidade:

| Sintoma | Causa provável |
|---|---|
| `exec format error` no log | Imagem amd64. Confirme o manifesto no ECR (`aws ecr batch-get-image`) |
| Task para logo após iniciar, log com `Error: Variável de ambiente API_URL não configurada.` | Faltou `API_URL` no `environment` — é a armadilha da §9 do documento de arquitetura |
| `EROFS: read-only file system` | Falta um `tmpfs`. Monte o diretório reclamado; **não** desligue o `readonlyRootFilesystem` |
| Erro de permissão ao escrever em `.next/cache` | `tmpfs` sem `uid=1000,gid=1000` — montou como root |
| Task nunca fica `healthy`, sem erro no log | Healthcheck. Confirme que `/` responde: `sudo docker exec <container> node -e "fetch('http://localhost:3000').then(r=>console.log(r.status))"` |
| `RESOURCE:MEMORY` nos eventos do service | Não coube. Confira o `Livre` do comando acima |

O log da aplicação está em `/ecs/cronos-front`; os eventos do ECS, em
`aws ecs describe-services --cluster ec2-sistema-despesas --services cronos-front --query "services[].events[:10]"`.

---

## 9. Rollback

Esta fase é reversível sem tocar em nada que já funcionava:

```bash
aws ecs update-service --region us-east-2 --cluster ec2-sistema-despesas --service cronos-front --desired-count 0
```

**O que faz:** derruba a task do front, liberando a memória e a porta 3000. A API e o banco não são
afetados — é o benefício concreto de ter separado as tasks. Para remover de vez:
`aws ecs delete-service --cluster ec2-sistema-despesas --service cronos-front --force`, e a task
definition pode ficar registrada (não custa nada).

---

## 10. O que esta fase deixa aberto

1. ~~**O RSS real do front nunca foi medido.**~~ **Medido em 20/08/2026:** 55,73 MiB de 512 MiB
   reservados (10,88%) — bem abaixo da estimativa de ~250–400 MB que vinha desde antes do build
   `standalone`. Os outros dois containers, na mesma medição: API 123,2 MiB / 448 MiB (27,5%),
   Postgres 35,79 MiB / 384 MiB (9,3%). Memória livre no cluster depois das três tasks: **694 MB**.
   O limite de 512 MB rígido / 320 flexível **fica como está** — a folga é saudável e não vale o
   churn de uma revisão nova só por isso; revisitar se o uso real crescer com tráfego de verdade.
2. **`FRONTEND_URL` na API continua sendo `http://localhost:3000`** (`api-aws.md` §13.5). Ele alimenta
   o `Access-Control-Allow-Origin` e o redirect do OAuth — vira a origem real na Fase 7.
3. ~~**Nada está exposto na internet ainda.**~~ **Resolvido pela Fase 5** ([`ingresso-aws.md`](./ingresso-aws.md)):
   o sistema está acessível em `https://d3c5d6t3539m1d.cloudfront.net`.
4. ~~**Ainda não existe Elastic IP.**~~ **Resolvido na Fase 5:** EIP associado, e é ele que o
   CloudFront usa como origem.
5. **O espelhamento no ECR continua manual** (`api-aws.md` §13.6). Automatizar no job `promote` é a
   Fase 8 — e agora são **três** imagens a espelhar (API, front e Caddy).
6. **`extraHosts` amarra à premissa de uma instância só.** No dia em que houver duas, o caminho é
   Service Connect (§3.3 do documento de decisão), trocando um campo por outro.
7. **O rewrite `/api/*` continua congelado em build-time** — a §7.1 conserta o *valor*, não o
   *mecanismo*. A imagem do front segue específica do ambiente: trocar o endereço da API exige rebuild
   + re-espelhamento + redeploy, e o erro reaparece silenciosamente se alguém esquecer. A correção é
   um **Route Handler** que resolve `process.env.API_URL` em runtime — Abordagem B de
   [`problema-rewrite-api-build-time.md`](../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md)
   §7. **Gatilho: a Fase 7**, que muda o endereço de qualquer jeito. Ponto crítico da implementação: o
   repasse de `Set-Cookie` — se errar, o login quebra de um jeito difícil de diagnosticar.
8. **Google OAuth e SMTP não têm variáveis configuradas na task da API** (`api-aws.md` §5). O código
   dos dois está pronto, mas inerte: o botão de login com Google não funciona, e a recuperação de
   senha **completa o fluxo sem nunca enviar o email**. O
   [`env.ts`](../../sistema-controle-despesas-api/src/config/env.ts) trata cada grupo como "tudo ou
   nada" — preencher pela metade **impede a API de subir**. São 4 variáveis para o Google
   (`GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL`, `COOKIE_SESSION_SECRET`) e 5
   para o SMTP (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, `MAIL_FROM`), os segredos via
   SSM. Ambas dependem da Fase 7: o `GOOGLE_CALLBACK_URL` precisa da URL pública real (registrada
   também no Google Cloud Console), e `FRONTEND_URL` — que alimenta o redirect final do OAuth —
   precisa deixar de ser placeholder junto (item 2).
9. ~~**Padronizar a porta da API em `8080`.**~~ — **concluído em 21/08/2026**, fora da Fase 7 (não
   valeu esperar). Os 6 arquivos de código e as 3 mudanças de infra (Security Group, `cronos-app:2`,
   `cronos-front:3`) estão feitos e verificados ponta a ponta via SSM. O repo do front não teve nenhum
   arquivo a mudar — só a Repository Variable `API_URL`, que segue em `http://api:3001` (sem efeito no
   que está no ar). Detalhamento em
   [`problema-rewrite-api-build-time.md`](../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md)
   §14.4.

---

## 11. Referências

- [`separacao-de-tasks-front-api.md`](./separacao-de-tasks-front-api.md) — a decisão que este plano executa
- [`arquitetura-aws.md`](./arquitetura-aws.md) — §-1 (estado real), §9 (o front e a armadilha do `API_URL`), §10 (modo de rede), §13 (roteiro)
- [`api-aws.md`](./api-aws.md) — §4 (log group antes do console), §5 (task definition campo a campo), §6 (memória), §7 (filesystem somente leitura), §9 (health check), §11 (opções do service)
- [`banco-de-dados-aws.md`](./banco-de-dados-aws.md) — §7 (as três roles IAM)
- `sistema-controle-despesas-front`: `src/proxy.ts` (matcher), `src/lib/session.ts` (degradação graciosa), `src/lib/apiClient.ts` (`TIMEOUT_MS`), `Dockerfile` (`USER node`, healthcheck)
