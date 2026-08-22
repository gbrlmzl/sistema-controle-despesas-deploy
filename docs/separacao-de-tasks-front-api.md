# Separar o front da API em tasks distintas — CRONOS

Documento de decisão sobre **a topologia de tasks do ECS**: hoje o plano da Fase 6
(`arquitetura-aws.md` §13) põe o container do front na **mesma task** da API (`cronos-app`), e o custo
disso é que **todo deploy do front reinicia a API junto**. Este documento levanta as alternativas, o
que cada uma acarreta, e recomenda um caminho.

> **Estado:** proposta, nada provisionado. Escrito em 20/08/2026, antes da Fase 6.
> **Gatilho:** objeção levantada durante o planejamento — "não quero reiniciar a API pra reiniciar o
> front".
> **Escopo:** só a topologia de tasks/services. Não redecide banco, ingresso nem região.
> **Convenção:** ARNs e IDs de conta aparecem como `<conta>` — o repositório é público.

---

## 0. O resumo, pra quem só quer a resposta

**Separar em duas tasks é a recomendação, e custa US$ 0,00 a mais.** O que ela cobra não é dinheiro, é
uma decisão de endereçamento: hoje o front acharia a API por `links` do Docker (`http://api:8080`,
`3001` até 21/08/2026),
que só funciona **dentro da mesma task**. Separadas, elas precisam de outro caminho.

E há uma descoberta que muda o peso da decisão, detalhada na §2:

> **A trava do "`API_URL` é resolvida em build-time", que o `arquitetura-aws.md` §1.3 trata como
> restrição estrutural, é hoje quase inofensiva.** O rewrite `/api/*` congelado na imagem tem **um
> único consumidor no código**: o link do login com Google — que está **desligado em produção**
> (`api-aws.md` §5). Todo o resto (`src/lib/apiClient.ts` e `src/proxy.ts`) lê `process.env.API_URL`
> em **runtime**.

Ou seja: mudar o endereço da API em produção é, hoje, trocar uma variável de ambiente na task
definition. Não exige rebuild.

---

## 1. O acoplamento real: o que exatamente reinicia junto

Na topologia de task única, **a task é a unidade atômica de implantação**. O ECS não reinicia um
container isolado dentro dela — ele para a task inteira e sobe outra.

Somando isso à configuração que a Fase 4 já registrou (`api-aws.md` §11):

```
desiredCount = 1
minimumHealthyPercent = 0
maximumPercent = 100
```

O `minimumHealthyPercent = 0` existe porque a porta é fixa no host e a instância é uma só: a task
antiga **precisa sair antes** de a nova entrar. O padrão (`100`) travaria o deploy para sempre.

A consequência concreta de juntar front e API:

| Ação | Task única | Tasks separadas |
|---|---|---|
| Deploy do front | **API cai junto**, ~30 s | Só o front cai |
| Deploy da API | Front cai junto, ~30 s | Só a API cai |
| Crash do front (OOM, bug) | `essential: true` derruba a task → **API morre junto** | API intacta |
| Rollback do front | Volta a revisão inteira, incluindo a API | Independente |

A linha do **crash** costuma passar batido e é a mais séria. Em ECS, se um container marcado
`essential: true` morre, o agente **para a task inteira**. Marcar o front como `essential: false`
evitaria derrubar a API, mas cria algo pior: o front morto e ninguém reiniciando ele, com a task ainda
contando como saudável.

> Em ~30 s de indisponibilidade por deploy, o impacto direto é pequeno num projeto de portfólio. O
> argumento forte contra o acoplamento **não é o downtime** — é que ele impede raciocinar sobre as
> duas peças separadamente, e é exatamente o tipo de decisão que um avaliador técnico questiona numa
> entrevista.

---

## 2. A restrição que todo mundo assume, e que não se sustenta

O `arquitetura-aws.md` §1.3 e o `README.md` deste repositório afirmam que o destino do rewrite
`/api/*` é resolvido em build-time e que por isso a topologia fica congelada dentro da imagem. **É
verdade sobre o rewrite, e é irrelevante na prática hoje.** Levantamento feito no código do front em
20/08/2026:

| Consumidor | Como resolve `API_URL` | Usa o rewrite? |
|---|---|---|
| `src/lib/apiClient.ts` (Server Components, Server Actions) | `process.env.API_URL` no boot | ❌ chama a API direto |
| `src/proxy.ts` (guarda de rota, `POST /auth/refresh` a cada request) | `process.env.API_URL` no boot | ❌ chama a API direto |
| `next.config.ts` → rewrite `/api/:path*` | **build-time**, congelado no `routes-manifest.json` | ✅ |
| **Quem de fato navega para `/api/*`** | `LoginForm.tsx` e `RegisterForm.tsx` → `<a href="/api/auth/google">` | ✅ ~~**único**~~ **ERRADO — ver abaixo** |

> 🛑 **A linha acima está errada, e o erro custou um incidente em produção.** Ela concluiu que o
> rewrite tinha um consumidor só porque a busca foi feita por `fetch("/api` e pelo literal `"/api/"`.
> O arquivo `src/lib/apiClient.client.ts` **monta a URL por concatenação** (`const BASE_URL = "/api"`,
> depois `` fetch(`${BASE_URL}${path}`) ``) e por isso não apareceu. Os consumidores reais são **9
> hooks e 8 rotas da API** — praticamente toda a interatividade do app depois do primeiro render.
>
> Em 20/08/2026 a imagem foi publicada com o placeholder e **toda chamada do navegador falhou em
> produção**, enquanto o servidor continuava funcionando normalmente. Diagnóstico completo em
> [`problema-rewrite-api-build-time.md`](../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md).
>
> **A lição:** ao verificar dependências de um rewrite, procure pela **construção da URL**, não só pelo
> literal.

E o login com Google **não está habilitado em produção** — foi decisão explícita da Fase 4
(`api-aws.md` §5): o `env.ts` da API valida o grupo OAuth como "tudo ou nada", então subir sem ele é o
caminho seguro até haver domínio.

**Três conclusões — a primeira continua válida, as outras duas caíram:**

1. Trocar o endereço da API em produção **não exige rebuild** para `apiClient.ts` e `proxy.ts` — esses
   dois leem variável de ambiente em runtime. **Mas exige, sim, para o caminho client-side**, que
   depende do rewrite congelado. É exatamente a assimetria que produziu o bug.
2. ~~Quando o Google login for ligado, o rewrite volta a importar.~~ **O rewrite já importava, o tempo
   todo** — e importa para muito mais que o OAuth. Deixou de ser "dívida datada" para ser causa raiz
   ativa. Corrigida no valor em 21/08/2026 (Repository Variable `API_URL`), não no mecanismo.
3. ~~A afirmação do `README.md` está mais forte do que os fatos sustentam.~~ **A afirmação está
   correta:** com o rewrite congelado e a imagem do GHCR carregando o placeholder, buildar o front do
   fonte é mesmo a única forma de o `/api/*` funcionar dentro do compose. Manter o texto como está.

> **A saída definitiva** — que deixou de ser "se um dia incomodar" e virou trabalho pendente com
> gatilho conhecido: trocar o rewrite do `next.config.ts` por um **Route Handler** em
> `src/app/api/[...path]/route.ts` que faça o proxy lendo `process.env.API_URL` em runtime. Aí a imagem
> deixa de ter qualquer topologia embutida, o `--build-arg API_URL` some do Dockerfile e do CI, e o
> `docker-compose.yml` deste repositório passa a poder consumir a imagem publicada em vez de buildar do
> fonte. É mudança no repositório do front, não aqui. **Gatilho: a Fase 7**, que muda o endereço
> público e portanto reativaria o bug. Ponto crítico da implementação: o repasse de `Set-Cookie`.

---

## 3. O problema que a separação cria: como o front acha a API

Com `networkMode: bridge`, containers da **mesma task** se enxergam por nome via `links` — é o que faz
`http://api:8080` funcionar. **`links` não atravessa tasks.** Separadas, as opções são:

### 3.1 IP do gateway da bridge do Docker — `http://172.17.0.1:8080`

Todo container em modo bridge enxerga o host pelo gateway da rede `docker0`, que é `172.17.0.1` por
padrão do Docker. Como a API publica `8080` no host, o front alcança ela por aí.

- ✅ **Zero RAM extra, zero custo, zero serviço novo.**
- ✅ **Estável entre instâncias** — não é o IP privado da instância (que muda se ela for substituída,
  a dívida registrada em `api-aws.md` §13.1), é uma constante do Docker.
- ❌ É um endereço mágico. Quem ler a task definition daqui a seis meses não vai saber o que é
  `172.17.0.1` sem um comentário.
- ❌ Amarra à premissa "front e API sempre na mesma instância". Verdade hoje (uma instância só), falso
  no dia em que houver duas.
- ⚠️ Depende do subnet default do Docker. Se alguém configurar `bip` no daemon, quebra.

### 3.2 IP privado da instância — `http://172.31.18.227:8080`

O mesmo caminho que o `DATABASE_URL` da API já usa para achar o Postgres.

- ✅ Consistente com o que já existe.
- ❌ **Herda a dívida mais perigosa do projeto**: se a instância for substituída, o IP muda e o front
  quebra em silêncio. É a pendência nº 1 de `api-aws.md`, e replicá-la é dobrar a aposta.

### 3.3 ECS Service Connect

O caminho "certo" da AWS: dá nomes DNS internos (`api.cronos.local`), com retry e métricas.

- ✅ Endereçamento estável e legível, independente de instância ou IP.
- ✅ É o que se espera ver num desenho de microsserviços — bom sinal em portfólio.
- ❌ **Injeta um sidecar Envoy em cada task.** Numa instância de 2 GB isso é caro: a AWS sugere
  reservar da ordem de 64 MB por sidecar, e seriam duas tasks. **~128 MB do orçamento**, ver §5.
- ❌ Complexidade real (namespace no Cloud Map, `serviceConnectConfiguration` nas duas pontas) para um
  sistema com dois serviços numa máquina.

### 3.4 `networkMode: awsvpc` + Cloud Map

Cada task ganha ENI própria e IP na VPC, com DNS pelo Cloud Map.

- ✅ Security group **por task** — a segmentação mais granular disponível.
- ❌ **Limite de ENIs.** A `t4g.small` suporta poucas interfaces, e uma já é da instância. Com
  Postgres, API e front em `awsvpc`, o teto aperta ou estoura. *(Confirmar com
  `aws ec2 describe-instance-types --instance-types t4g.small --query "InstanceTypes[].NetworkInfo"`
  antes de considerar seriamente. O ENI trunking existe, mas é opt-in e nem todo tipo de instância
  suporta.)*
- ❌ Migração maior: as três task definitions mudam de modo de rede de uma vez.

### 3.5 Passar pelo Caddy

O Caddy da Fase 5 já vai existir e publicar porta no host. O front apontaria para ele, e ele rotearia
para a API.

- ✅ Reaproveita peça já planejada, sem serviço novo.
- ❌ Um salto a mais em toda chamada, incluindo o `/auth/refresh` que o `proxy.ts` faz **a cada
  request**.
- ❌ Inverte a hierarquia: o Caddy é a borda; fazer a borda servir tráfego interno confunde o desenho.

---

## 4. Custo

Esta é a parte curta, e é o argumento mais forte a favor de separar:

| Item | Task única | Tasks separadas | Δ |
|---|---|---|---|
| Compute (EC2) | já pago | **mesma instância** | **US$ 0,00** |
| ECS (orquestração) | US$ 0 | US$ 0 | US$ 0 |
| Load balancer | não existe | **continua não existindo** | US$ 0 |
| Service Connect (se escolhido, §3.3) | — | sem taxa de serviço | US$ 0 |
| Cloud Map (namespace + registros) | — | ~US$ 0,10/recurso/mês + consultas | ~US$ 0,10–0,30 |
| ECR (imagem já espelhada) | igual | igual | US$ 0 |

**O ECS não cobra por task nem por service** no launch type EC2 — você paga a instância, e ela já está
paga. Duas tasks de 512 MB custam o mesmo que uma task de 512 MB: nada a mais.

> A intuição de "mais tasks = mais caro" vem do **Fargate**, onde se paga por vCPU/GB-hora de cada
> task. Neste desenho (EC2), o custo marginal de separar é literalmente zero — o que se gasta é RAM, e
> RAM aqui é um teto, não uma fatura.

---

## 5. O orçamento de RAM, que é a moeda de verdade

A instância registra **1846 MB** no ECS (medido em 20/08/2026, depois do upgrade para `t4g.small`).

| Cenário | postgres | api | front | caddy | sidecars | Total | Folga |
|---|---|---|---|---|---|---|---|
| **A** — task única (plano atual) | 384 | 448 | 512 | 118 | — | 1462 | **384** |
| **B** — tasks separadas, endereço direto (§3.1) | 384 | 448 | 512 | 118 | — | 1462 | **384** |
| **C** — tasks separadas + Service Connect | 384 | 448 | 512 | 118 | ~128 | 1590 | **256** |

**A e B custam exatamente a mesma RAM.** Separar tasks, por si só, não consome nada — a memória é
reservada por container, não por task. Só o Service Connect cobra, e cobra caro para o tamanho da
casa.

E vale lembrar do que o upgrade de instância deixou: **2 GB de swap configurados**. Ele não substitui
RAM (swap em EBS é lento), mas transforma "container morto pelo OOM killer sem explicação" em "sistema
lento e diagnosticável" — o que muda a leitura da folga acima.

---

## 6. Como o mercado faz

Vale separar o que é consenso do que é contexto.

**Consenso: uma task por unidade implantável.** O padrão amplamente adotado é *um service ECS por
componente que tem ciclo de vida próprio*. Front e API são versionados separados, têm repositórios
separados, CIs separados e cadências de deploy diferentes — pelo critério do mercado, são dois
serviços. A documentação da AWS sobre task definitions vai na mesma direção: agrupe containers numa
task **só quando eles precisam compartilhar ciclo de vida**.

**O caso legítimo de múltiplos containers numa task é o *sidecar*:** peças que existem em função do
container principal e não fazem sentido sozinhas — coletor de logs (Fluent Bit/FireLens), proxy de
service mesh (Envoy), agente de métricas, container de credenciais. A regra prática:

> Se o container B **não tem razão de existir** sem o container A, é sidecar e vive na mesma task.
> Se B tem valor próprio e roda independente, é outro service.

O front tem valor próprio: serve páginas, tem healthcheck próprio, escala por conta e é implantado por
um pipeline diferente. **Não é sidecar da API.**

**Onde o mercado *aceita* o agrupamento:** ambientes de laboratório e demos com uma instância só, onde
a simplicidade da rede compensa. É o que o `arquitetura-aws.md` §9 propôs — e a proposta é defensável,
mas o motivo dela era **o `links` do Docker resolver o endereçamento de graça**, não uma preferência
arquitetural.

**O que um avaliador provavelmente perguntaria:** "por que o front e a API estão na mesma task?" A
resposta "porque assim o `links` funciona" é honesta, mas revela que a topologia foi decidida por
conveniência de rede. Já "estão separados, e o front acha a API por X" mostra que a pergunta foi
feita.

---

## 7. Comparativo

| | A — task única | B — separadas, endereço direto | C — separadas + Service Connect |
|---|---|---|---|
| Deploy do front derruba a API | ❌ sim | ✅ não | ✅ não |
| Crash do front derruba a API | ❌ sim | ✅ não | ✅ não |
| Custo em dinheiro | US$ 0 | **US$ 0** | ~US$ 0,10–0,30/mês |
| Custo em RAM | 0 | **0** | ~128 MB |
| Endereçamento | `http://api:8080` (`links`) | `http://172.17.0.1:8080` | `http://api.cronos.local:8080` |
| Legibilidade do endereço | ✅ óbvia | ⚠️ precisa de comentário | ✅ óbvia |
| Sobrevive à troca da instância | ✅ | ✅ | ✅ |
| Sobrevive a uma segunda instância | ✅ | ❌ | ✅ |
| Esforço a partir de hoje | nenhum | **baixo** | médio-alto |
| Sinal em portfólio | fraco | bom | ótimo |

---

## 8. Recomendação

**Cenário B: duas tasks, front achando a API pelo gateway da bridge (`http://172.17.0.1:8080`).**

Os motivos, em ordem:

1. **É grátis nos dois orçamentos** — nem dólar nem megabyte. Não existe trade-off a ponderar.
2. **Resolve o problema que originou este documento**, incluindo o caso do crash, que é mais grave que
   o do deploy.
3. **O `API_URL` é variável de ambiente de runtime** (§2). Trocar `http://api:8080` por
   `http://172.17.0.1:8080` é editar um campo da task definition — não exige rebuild, não toca no CI,
   não muda a imagem já espelhada no ECR.
4. **Service Connect continua disponível depois.** Migrar de B para C é trocar o valor de uma variável
   e adicionar `serviceConnectConfiguration` — nada em B fecha essa porta.

O que se aceita conscientemente:

- O `172.17.0.1` é opaco e **exige comentário na task definition** dizendo o que é e por que não é o
  IP da instância.
- Amarra à premissa de uma instância só. Verdadeiro hoje, e o dia em que deixar de ser é o dia de ir
  para o Service Connect — que é a migração natural, não retrabalho.

**Quando escolher C em vez de B:** se aparecer uma segunda instância, se um terceiro serviço entrar na
malha, ou se o objetivo de portfólio pesar mais que os 128 MB. Aí o Service Connect deixa de ser
overkill.

**Quando ficar em A:** se a Fase 6 precisar subir hoje e a simplicidade do `links` valer mais que a
independência. É uma escolha defensável — mas com este documento escrito, passa a ser uma escolha, não
um default.

---

## 9. O que muda na prática, se B for aprovado

Nenhum destes passos foi executado. São o delta em relação ao plano da Fase 6.

1. **Nova task definition `cronos-front`** (família própria), com um container só: imagem do ECR,
   `linux/arm64`, `bridge`, `3000:3000`, memória rígida 512 / flexível 320,
   `readonlyRootFilesystem: true` + `tmpfs` em `/app/.next/cache`, healthcheck em `/`.
2. **`API_URL=http://172.17.0.1:8080`** (`3001` até 21/08/2026) como variável de ambiente, com
   comentário explicando o endereço.
3. **Novo service `cronos-front`**, `desiredCount=1`, `minimumHealthyPercent=0`, `maximumPercent=100`
   (mesma razão de sempre: porta fixa, instância única), disjuntor de implantação com reversão
   automática.
4. **A `cronos-app` não muda** — continua com o container `api` sozinho. Nenhuma revisão nova, nenhum
   downtime da API para fazer esta mudança.
5. **Security group:** confirmar que a porta da API é alcançável a partir da bridge do Docker. O
   tráfego não sai da instância (é loopback via `docker0`), então a regra atual deve bastar — mas é o
   primeiro ponto a verificar se o front subir e não conseguir falar com a API. (Reconfirmado em
   `8080` após a padronização de 21/08/2026 — ver item 6 da lista de pendências abaixo.)
6. **Caddy (Fase 5)** passa a apontar para o front na porta 3000 do host, sem mudança em relação ao
   que já estava previsto.

---

## 10. Pendências e decisões em aberto

1. ~~**O `README.md` deste repositório precisa de revisão.**~~ **Não precisa** — a afirmação está
   correta; era a §2 que estava errada. Ver a correção no topo da §2.
2. **O rewrite `/api/*` é causa raiz ativa, não dívida datada.** Quebrou produção em 20/08/2026 e foi
   corrigido só no valor. A saída limpa continua sendo o **Route Handler** descrito na §2, e o gatilho
   é a **Fase 7** (domínio próprio), que muda o endereço público e reativaria o bug. Ponto crítico:
   o repasse de `Set-Cookie` — se errar, o login quebra de um jeito difícil de diagnosticar.
3. **Confirmar os limites de ENI da `t4g.small`** antes de qualquer conversa séria sobre `awsvpc`.
4. **O `DATABASE_URL` com IP privado da instância** (`api-aws.md` §13.1) continua aberto e é o mesmo
   problema de endereçamento por outro ângulo. Service Connect resolveria os dois de uma vez — vale
   pesar isso quando a hora chegar.
5. **Google OAuth e SMTP sem variáveis na `cronos-app`** (`ingresso-aws.md` §11.8). Toca este documento
   por um detalhe de topologia: `/api/auth/google` é uma **navegação do navegador**, não um `fetch`,
   então atravessa o mesmo rewrite da §2 e depende do Route Handler repassar o `302` corretamente
   quando ele existir.
6. ~~**Padronizar a porta da API em `8080`**~~ (`ingresso-aws.md` §11.10) — **concluído em
   21/08/2026**, código e infra AWS. A regra do Security Group que libera a porta a partir da bridge
   do Docker (item 5 da §9) foi replicada pra `8080` com o mesmo CIDR de antes — o CIDR-vs-
   *self-reference* de `api-aws.md` §13.4 **não foi corrigido**, só migrou de porta.

---

## 11. Referências

- [`arquitetura-aws.md`](./arquitetura-aws.md) — §1.3 (rewrite em build-time), §9 (onde deployar o front), §10 (modo de rede), §13 (roteiro)
- [`api-aws.md`](./api-aws.md) — §5 (task definition campo a campo), §6 (orçamento de memória), §11 (opções do service), §13 (pendências)
- [`banco-de-dados-aws.md`](./banco-de-dados-aws.md) — §6 (rede e *self-reference*)
- `sistema-controle-despesas-front`: `src/proxy.ts`, `src/lib/apiClient.ts`, `next.config.ts` — os três consumidores de `API_URL` levantados na §2
