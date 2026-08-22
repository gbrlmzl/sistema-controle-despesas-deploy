# Arquitetura de deploy na AWS — CRONOS

Documento de decisão sobre **onde e como hospedar o CRONOS na AWS** com orçamento apertado, sem abrir mão de um desenho que se defenda numa entrevista.

> **Estado:** **parcialmente provisionado.** Fases 0–4 do §13 feitas (conta, rede, instância, banco, API).
> O que está no ar e como foi feito está em [`banco-de-dados-aws.md`](./banco-de-dados-aws.md)
> (Fase 3) e [`api-aws.md`](./api-aws.md) (Fase 4).
> **Revisão 2:** decisão tomada de usar **ECS com capacity provider EC2** (não Fargate). O documento foi reescrito em torno disso; §2 registra por que a decisão se sustenta em números. Inclui a comparação de custo **EBS vs RDS** (§8).
> **Revisão 3 (19/08/2026):** o provisionamento real divergiu do plano em três pontos — região,
> instância e arquitetura. **Leia a §-1 antes de qualquer coisa**: partes deste documento descrevem a
> proposta original, não o que existe.
> **Revisão 4 (19/08/2026):** o front migrou para `linux/arm64` e ganhou build `standalone` — as duas
> pendências que a Revisão 3 registrava contra a Fase 6. **Sobrou a RAM**, que agora é a única coisa
> entre o front e a instância. Ver §-1, §1.10 e §9.
> **Preços:** referência de agosto/2026, on-demand, em dólar. Confira na [calculadora oficial](https://calculator.aws) antes de provisionar.
> **Escopo:** os três repositórios do sistema (`-api`, `-front`, `-deploy`).

---

## -1. O que foi realmente provisionado (e onde diverge deste documento)

Esta seção existe porque o resto do documento é uma **proposta escrita antes do provisionamento**, e
seguir os números dele hoje leva a erro. Três decisões mudaram na prática:

| | Proposta original | **Provisionado de fato** | Impacto |
|---|---|---|---|
| Região | `us-east-1` (§5) | **`us-east-2`** (Ohio) | Nenhum no custo — os "20% mais caro" eram contra `sa-east-1`, não entre regiões dos EUA |
| Instância | `t3.small` — 2 vCPU, **2 GB**, x86 | **`t4g.micro`** — 2 vCPU, **1 GB**, ARM64 | Custo cai de US$ 15,18 para **~US$ 6,13/mês**. Em troca, **a RAM deixou de fechar** — ver abaixo |
| Arquitetura das imagens | amd64, com multi-arch como exercício opcional (§1.10) | **API e front migrados para arm64 puro** | Resolvido. A trava do front passou a ser RAM, não arquitetura |

### O orçamento de RAM não fecha mais

A §6 dimensionou o sistema em **~900 MB – 1,2 GB** e concluiu que `t3.micro` (1 GB) "não fecha". A
`t4g.micro` tem exatamente 1 GB. O que isso significa, concretamente:

| Processo | RAM | Cabe na `t4g.micro`? |
|---|---|---|
| AL2023 + agente ECS + containerd | ~250 MB | — |
| `postgres` (limite rígido já aplicado) | 384 MB | ✅ no ar |
| `api` | ~150–250 MB | ✅ deve caber, com pouca folga |
| `caddy` | ~20 MB | ✅ |
| `front` | ~250–400 MB | ❌ **não cabe** |

> **Consequência prática:** Postgres + API + Caddy é o teto realista desta instância. O front (Fase 6)
> exige uma das três saídas: subir para `t4g.small` (2 GB, ~US$ 12,26/mês), mover o front para a
> Vercel (§9 já registra isso como plano B, e libera ~350 MB), ou aceitar swap e a lentidão que vem
> junto. **Não tente encaixar os quatro containers em 1 GB** — o OOM killer escolhe a vítima, e
> estatisticamente é o Postgres.
>
> **O que a migração do front mudou aqui: nada.** O build `standalone` (§9) atacou o **tamanho da
> imagem** — payload da aplicação de >1 GB para ~47 MB, imagem final ~390 MB —, e isso resolve o
> problema de **disco**, que era real numa raiz de 30 GB. Ninguém mediu o RSS do `node server.js` em
> produção; até que alguém meça, tratar os ~250–400 MB como inalterados é a hipótese segura.
>
> Configurar **1–2 GB de swap** deixou de ser opcional aqui: numa instância de 1 GB é o que separa
> "lento" de "container morto sem explicação".

### O ARM64 deixou de ser pendência — nos dois repositórios

A §1.10 abaixo foi escrita quando **as duas** imagens eram amd64 e ainda tratava Graviton como
migração futura. Hoje as duas publicam ARM:

| Repositório | Plataforma publicada | Roda na `t4g.micro`? |
|---|---|---|
| `-api` | **`linux/arm64`** (commit `365de83`, via QEMU no runner x86) | ✅ |
| `-front` | **`linux/arm64`** (`platforms: linux/arm64` + `docker/setup-qemu-action` no job `docker-publish`) | ✅ |

**Nenhuma das duas é multi-arch.** As imagens publicadas hoje rodam **só** em ARM — é uma via de mão
única deliberada, e voltar para uma instância x86 (`t3`) exigiria mexer nos dois CIs.

> ⚠️ **Estado no disco, em 19/08/2026:** a mudança do front está **no working tree do branch
> `dev/gbrlmzl`, ainda sem commit** — `.github/workflows/ci.yml`, `Dockerfile`, `next.config.ts` e
> `README.md` aparecem como modificados. Enquanto isso não for commitado, mergeado na `main` e rodado
> pelo CI, **o pacote no GHCR continua sendo o amd64 antigo**. O que esta seção descreve é o
> código-fonte, não ainda o artefato publicado.

---

---

## 0. Resumo executivo

| Peça | Escolha | Custo/mês |
|---|---|---|
| Compute | 1 EC2 `t3.small` (2 vCPU burst, 2 GB), ECS capacity provider com ASG `min=max=1` | US$ 15,18 (~US$ 10 com Savings Plan) |
| Containers | `caddy` + `front` + `api` num service; `postgres` em service separado | US$ 0 (marginal) |
| Banco | Postgres em container, volume **EBS gp3 dedicado**, snapshot diário (DLM) + `pg_dump` para S3 | ~US$ 1,35 |
| Entrada HTTPS | CloudFront na frente da instância (TLS válido de graça, 1 TB/mês grátis) | US$ 0 |
| IP | Elastic IP (endereço estável = DNS estável) | US$ 3,65 |
| Disco | EBS gp3 30 GB (raiz) + 10 GB (dados) | US$ 3,20 |
| Domínio | `.com.br` no Registro.br — R$ 40/ano fixo | ~US$ 0,65 |
| Segredos | SSM Parameter Store (SecureString, tier Standard) | US$ 0 |
| Logs, ECR, snapshots | retenção de 7 dias, lifecycle policy | ~US$ 1,50 |
| **Total** | **sistema inteiro na AWS, banco incluído** | **~US$ 25/mês** (~US$ 20 com Savings Plan) |

Trocar o Postgres em container pelo **RDS `db.t4g.micro`** custa **+US$ 12,65/mês** (US$ 25 → US$ 38). O que se compra com essa diferença está na §8 — é a decisão de maior impacto financeiro do documento.

E a decisão mais importante, que não é técnica:

> **O entregável de portfólio é o Terraform + o pipeline, não a instância ligada 24/7.**
> Com a infra em código, `terraform apply` sobe tudo em minutos antes de uma entrevista e `terraform destroy` derruba depois. Quem avalia um portfólio lê o código da infra e o pipeline; ninguém monitora seu uptime. Isso transforma um custo fixo de US$ 25/mês num custo de centavos, **sem** transformar o projeto em algo menor.

Os créditos do free tier atual (US$ 100–200, ver §4) cobrem **~8 meses** desse desenho ligado direto.

---

## 1. Restrições que vêm do código

Antes de comparar serviços: o sistema já tomou decisões que limitam o que faz sentido. Nenhuma é opinião — todas estão no código hoje.

### 1.1 A API é stateless — e isso é bom

Sessão é JWT em cookie `httpOnly` + refresh token opaco guardado no banco (tabela `RefreshToken`). Nada em memória, nada em disco. A API escala horizontalmente sem trabalho nenhum e roda em qualquer compute — sem sticky session, sem volume compartilhado, sem Redis.

### 1.2 O front **não** é estático

`next.config.ts` define um `rewrites()`, e `src/proxy.ts` é um middleware que lê cookies, decodifica o JWT e chama `POST /auth/refresh` no servidor antes de renderizar. Isso exige um **servidor Node rodando**.

> **Consequência:** o combo barato "S3 + CloudFront" (site estático) está descartado sem refatorar o front inteiro. Não procure essa economia — ela não existe aqui.

### 1.3 O destino do rewrite é resolvido em *build-time*

Já documentado em [`docker-e-arquitetura.md`](./docker-e-arquitetura.md) §4.2: o Next grava o destino do rewrite no `routes-manifest.json` durante o `next build`. **Trocar a API alvo exige rebuild da imagem do front.**

> **Consequência:** a imagem do front é *específica do ambiente*. O "build once, promote everywhere" do CI atual **não vale para o front** — a `:stable` carrega o `API_URL` que valia no momento do build, vindo de uma Repository Variable configurada à mão no GitHub e versionada em lugar nenhum.
>
> Com tudo na mesma instância isso fica quase indolor: o front aponta para `http://api:8080` (`3001` até 21/08/2026), que é um valor **fixo**, igual em qualquer deploy. O `API_URL` deixa de ser uma variável de ambiente e vira uma constante da arquitetura.
>
> **"Quase indolor" acabou custando um incidente.** Em 20/08/2026 a Repository Variable não existia, o CI caiu no fallback `http://localhost:8080` e a imagem foi para produção com o rewrite apontando para lugar nenhum — quebrando **toda** a interatividade client-side, enquanto o servidor seguia funcionando. Ver §9 e [`problema-rewrite-api-build-time.md`](../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md). O valor foi corrigido; **o mecanismo, não**. A saída definitiva é o Route Handler em runtime, que faz a imagem do front voltar a ser genérica e o "build once, promote everywhere" valer para os três repositórios.

### 1.4 O proxy same-origin significa que você precisa de **um** domínio, não dois

O navegador só fala com a origem do front (`/api/*`); quem fala com a API é o servidor do front. Os cookies de sessão nascem no domínio do front.

> **Consequência:** a API nunca precisa ser exposta na internet. Com tudo na mesma instância, ela pode nem publicar porta no host — só o Caddy fica exposto. **Você compra no máximo um domínio**, e a superfície de ataque encolhe.

### 1.5 Cookies são `secure` em produção → HTTPS não é opcional

`src/lib/session.ts` marca os cookies com `secure: env.NODE_ENV === 'production'`. Sem HTTPS válido no domínio do front, **o login não funciona**: o navegador descarta o `Set-Cookie` silenciosamente. E `app.ts` já liga HSTS via helmet.

### 1.6 Se o login com Google for habilitado, o callback muda de dono

O front aponta o botão para `/api/auth/google` (`LoginForm.tsx`, `RegisterForm.tsx`), ou seja, **através do proxy**. Para o `Set-Cookie` do `googleCallback` cair no domínio do front, o `GOOGLE_CALLBACK_URL` em produção precisa ser:

```
https://<dominio-do-front>/api/auth/google/callback
```

e não `https://<host-da-api>/auth/google/callback`. Idem para o cookie `oauth_state`. Se o Google Login ficar desligado (as 4 variáveis vazias — a API sobe sem elas, ver `config/env.ts`), a restrição some.

### 1.7 Migration é um processo separado, e o desenho precisa respeitar isso

A API nunca migra ao subir. Na AWS isso vira um **`aws ecs run-task`** com `command` sobrescrito para `npx prisma migrate deploy`, rodado pelo pipeline **antes** do `update-service`. A imagem de runtime já suporta: `prisma` está em `dependencies`, o `openssl` está instalado e `prisma/` é copiado para a imagem final.

### 1.8 A purga de tokens já foi desenhada para o ECS

O comentário em `src/scripts/purgeTokens.ts` é explícito: **ECS Scheduled Task via EventBridge**, não `setInterval` dentro da API. Custo: ~30s por dia ≈ **US$ 0,01/mês**. Nada a decidir, só implementar.

### 1.9 As imagens estão no GHCR, e o pacote da API é privado

Na EC2 o `docker pull` sai da instância, então dá para autenticar no GHCR com um token no user-data — mas continua sendo melhor **espelhar para o ECR** no job `promote` que já existe: pull mais rápido, sem credencial de terceiro no host, sem depender de rate limit externo. Custo: ~US$ 0,10/GB-mês.

### 1.10 A arquitetura das imagens — arm64 puro nos dois repositórios

> **Atualizado em 19/08/2026 (revisão 4).** Esta seção descrevia um estado em que **as duas** imagens
> eram amd64 e Graviton era migração futura. As duas migraram. O resumo está na §-1; abaixo fica o
> detalhe.

**API — `linux/arm64`.** O workflow builda com `DOCKER_DEFAULT_PLATFORM: linux/arm64` e emulação QEMU
no runner x86 (`docker/setup-qemu-action`), salva a imagem como artifact (`docker save`) e a recarrega
no job `smoke-test` — que precisa do QEMU de novo, agora para **executar** a imagem ARM no runner x86.

**Front — `linux/arm64`.** Mesma peça: `docker/setup-qemu-action` antes do `build-push-action`, que
agora passa `platforms: linux/arm64`. Multi-arch (`linux/amd64,linux/arm64`) foi considerado e
recusado no próprio comentário do workflow: a emulação QEMU do build do Next é lenta e não há
consumidor x86 da imagem hoje. Se aparecer um, é somar `linux/amd64` na mesma linha.

**O que a escolha cobra:** com imagem ARM pura, qualquer runner ou máquina x86 que precise **rodar**
(não buildar) esses containers depende de QEMU registrado. Isso vale para o E2E deste repositório, que
consome a imagem publicada da API num runner `ubuntu-latest` — ver §6 de
[`docker-e-arquitetura.md`](./docker-e-arquitetura.md), onde a consequência ficou anotada como ponto a
verificar.

**O ganho de custo já foi capturado:** `t4g.micro` (~US$ 6,13/mês) contra `t3.small` (US$ 15,18) —
bem mais que os US$ 2,92 que esta seção previa, porque a mudança de tamanho veio junto com a de
arquitetura. O preço disso está na §-1: **1 GB de RAM não comporta os quatro containers.**

### Resumo

| Restrição | Consequência na AWS |
|---|---|
| API stateless | Qualquer compute serve |
| Front precisa de Node | S3 estático fora; container na instância |
| `API_URL` em build-time | Com tudo na mesma instância, vira constante |
| Proxy same-origin | Um domínio só; API não precisa ser exposta |
| Cookies `secure` | HTTPS obrigatório no front |
| Google OAuth (se ligado) | `GOOGLE_CALLBACK_URL` passa pelo front |
| Migration fora do boot | `ecs run-task` antes do `update-service` |
| Purga de tokens | EventBridge → ECS Scheduled Task |
| Imagens no GHCR privado | Espelhado no ECR (feito — ver §-1) |
| **API e front são arm64 puro** | **A instância `t4g` (ARM) é o único destino possível das imagens publicadas** |
| **Instância com 1 GB de RAM** | **Cabem Postgres + API + Caddy. O front não cabe** |

---

## 2. Por que EC2 e não Fargate

A decisão já está tomada, mas vale registrar que ela se sustenta em números — não é economia de aparência.

O Fargate cobra **por task**, por vCPU e por GB reservados. Este sistema tem quatro containers, um deles com estado:

| | Fargate | ECS EC2 (`t3.small`) |
|---|---|---|
| `api` (0.25 vCPU / 0.5 GB) | US$ 9,01 | — |
| `front` (0.5 vCPU / 1 GB) | US$ 18,02 | — |
| `caddy` / ingress | ALB obrigatório: ~US$ 25 | — |
| `postgres` com volume | não roda bem: precisa de EFS (US$ 0,30/GB-mês) e o desempenho não presta | — |
| IPv4 público | US$ 3,65 **por task** com IP | US$ 3,65 (um, da instância) |
| **Total realista** | **US$ 55–70/mês** | **US$ 15,18 + disco** |

O ponto: **no Fargate, cada container novo tem preço; na EC2, o preço já foi pago** — o limite é a RAM da instância, não a fatura. Um sistema de quatro containers, um deles stateful, é exatamente o caso em que EC2 ganha de longe.

O que você aceita em troca, e precisa saber nomear:

| Perde | Impacto real aqui |
|---|---|
| Host gerenciado (patch, AMI, disco cheio) | Vira sua responsabilidade — mitigado por AMI imutável (§12) |
| Isolamento por task | Todos os containers dividem kernel e RAM da mesma instância |
| Escala elástica trivial | `desiredCount` esbarra na capacidade da instância, não na conta |
| Ponto único de falha | Uma instância, uma AZ. Falha = restauração manual |

Para um portfólio, esse é um trade-off consciente e defensável. Para produção com SLA, não seria.

---

## 3. Quanto custa cada peça

| Item | Preço unitário | Mês (24/7) |
|---|---|---|
| **EC2 `t3.small`** (2 vCPU burst, 2 GB, x86) | US$ 0,0208/h | **US$ 15,18** |
| EC2 `t3a.small` (AMD, 2 GB) | US$ 0,0188/h | US$ 13,72 |
| EC2 `t3.micro` (1 GB) | US$ 0,0104/h | US$ 7,59 |
| EC2 `t4g.small` (ARM, 2 GB) — exige imagem ARM (§1.10) | US$ 0,0168/h | US$ 12,26 |
| Savings Plan de 1 ano, sem entrada | −30% a −40% | `t3.small` ≈ US$ 10 |
| EC2 Spot | −70% | `t3.small` ≈ US$ 4,55 |
| **EBS gp3** (3.000 IOPS e 125 MB/s inclusos em qualquer tamanho) | US$ 0,08/GB-mês | 30 GB = US$ 2,40 |
| Snapshot de EBS (incremental) | US$ 0,05/GB-mês | ~US$ 0,50 |
| IPv4 público / Elastic IP em instância ligada | US$ 0,005/h | **US$ 3,65** |
| **RDS `db.t4g.micro`** Postgres Single-AZ | US$ 0,016/h | **US$ 11,68** |
| Storage do RDS (gp3, mínimo 20 GB) | US$ 0,115/GB-mês | US$ 2,30 |
| Backup automático do RDS | grátis até 100% do storage | US$ 0 |
| **ALB** | US$ 0,0225/h + LCU + IPv4 por AZ | US$ 20–25 |
| **NAT Gateway** | US$ 0,045/h + US$ 0,045/GB | US$ 33+ |
| CloudFront | 1 TB de saída + 10M req/mês **sempre grátis** | **US$ 0** |
| Transferência EC2 → CloudFront (origin fetch) | grátis | US$ 0 |
| ACM (certificado TLS público) | grátis | US$ 0 |
| Route 53 (hosted zone) | US$ 0,50/zona | US$ 0,50 |
| ECR (storage) | US$ 0,10/GB-mês | ~US$ 0,20 |
| CloudWatch Logs | US$ 0,50/GB ingerido + US$ 0,03/GB-mês | <US$ 1 com retenção de 7 dias |
| SSM Parameter Store (Standard) / Session Manager | grátis | US$ 0 |
| S3 (dumps) | US$ 0,023/GB-mês | centavos |
| Taxa de orquestração do ECS sobre EC2 | **não existe** | US$ 0 |

### As armadilhas que estouram orçamento

1. **NAT Gateway — US$ 33/mês.** Existe só para dar internet a subnet privada e custa mais que a instância inteira. **Não use.** Instância em subnet pública, com Security Group fechado — que é onde a proteção real sempre esteve.
2. **ALB — US$ 20–25/mês.** Com uma instância e um Elastic IP, ele não resolve nenhum problema que você tenha: o endereço já é estável. Pagaria US$ 25/mês por um health check.
3. **Multi-AZ / segunda instância — dobra tudo.** É failover automático. Portfólio precisa de **backup** (barato), não de failover (caro).
4. **CloudWatch com retenção infinita.** É o default e é assim que ele vira uma linha inexplicável na fatura.

---

## 4. Free tier em 2026 — leia antes de planejar

**Mudou, e muda o planejamento.** Desde 15/07/2025 a AWS aposentou o "12 meses grátis" para contas novas:

- Conta nova recebe **US$ 100 em créditos**, e mais **US$ 100** completando cinco tarefas de onboarding (US$ 20 cada: subir/terminar uma EC2, configurar um RDS, publicar uma Lambda, testar um prompt no Bedrock, criar um budget).
- O **Free Plan** dura o que vier primeiro: **6 meses** ou o fim dos créditos.
- Uso de EC2 e RDS **consome crédito** — não existe mais a franquia de 750 h/mês.
- Créditos expiram 12 meses após a criação da conta.
- A camada **Always Free** continua (CloudFront 1 TB/mês, SSM Parameter Store, ECS sem taxa de orquestração, 10 alarmes do CloudWatch…) — e é dela que vem boa parte da economia aqui.

> **Implicação:** não planeje "vai ficar de graça no primeiro ano". Planeje **US$ 200 de orçamento total** e custo real a partir do mês ~6–8. O desenho de US$ 25/mês vive ~8 meses de crédito.
>
> **Primeira coisa na conta nova, antes de qualquer recurso:** um AWS Budget de US$ 5–10 com alerta por e-mail. Além de boa prática, ainda rende US$ 20 de crédito.

---

## 5. Região: `us-east-1` ou `sa-east-1`?

| | `us-east-1` (Virgínia) | `sa-east-1` (São Paulo) |
|---|---|---|
| Preço | O mais barato da AWS (referência deste doc) | ~20% mais caro em compute; RDS chega a **+60%** |
| Latência do Brasil | ~110–150 ms | ~10–30 ms |
| Disponibilidade de serviços | Tudo, primeiro | Subconjunto, mais tarde |

**Recomendação: `us-east-1`.** Com orçamento limitadíssimo, 20% a mais na instância e 60% no RDS é decisivo; a latência não é. O CloudFront serve o conteúdo estático a partir da borda em São Paulo de qualquer forma, então o que atravessa o continente são só as chamadas de API.

Se um dia houver usuários reais no Brasil, aí sim vale `sa-east-1` — e a decisão fica registrada aqui com o motivo.

---

## 6. Dimensionando a instância

### RAM é o recurso escasso, não CPU

Este app é I/O-bound e quase sempre ocioso; 2 vCPUs burstable sobram. A RAM é que decide:

| Processo | RAM aproximada |
|---|---|
| Amazon Linux 2023 + agente ECS + containerd | ~250 MB |
| `caddy` | ~20 MB |
| `api` (Node 24 / Express) | ~150–250 MB |
| `front` (Next 16 em produção) | ~250–400 MB |
| `postgres` (com `shared_buffers=128MB`) | ~200–300 MB |
| **Total** | **~900 MB – 1,2 GB** |

> **`t3.micro` (1 GB) não fecha** — o OOM killer vai escolher uma vítima, e estatisticamente será o Postgres. **`t3.small` (2 GB) é o mínimo realista**, com ~800 MB de folga para picos de build/deploy.
>
> Configure **1–2 GB de swap em arquivo** no volume raiz (a AMI ECS-optimized não traz swap). Custa centavos de EBS e transforma um OOM kill em lentidão — que é infinitamente melhor de diagnosticar.

> ⚠️ **O que foi provisionado contraria este dimensionamento.** A instância real é uma **`t4g.micro`,
> com 1 GB** — exatamente o tamanho que o parágrafo acima descarta. A conclusão continua válida: os
> quatro containers **não** cabem. O que cabe é Postgres + API + Caddy, e o front precisa de outro
> destino ou de uma instância maior. Ver §-1 para o orçamento de RAM detalhado e as três saídas.
>
> Nesse contexto o **swap deixa de ser recomendação e vira requisito** — ainda não configurado.

### Limites de memória no ECS EC2 (detalhe que trava deploy)

Diferente do Fargate, na EC2 o scheduler **só coloca uma task se houver memória não reservada suficiente**. Se você declarar `memory` (limite rígido) em todos os containers e a soma passar da RAM da instância, o deploy falha com `RESOURCE:MEMORY` e nada sobe.

Use **`memoryReservation`** (limite flexível) na maioria dos containers e `memory` só no Postgres, para ele nunca ser espremido. Deixe ~300 MB fora de qualquer reserva para o SO e o agente (`ECS_RESERVED_MEMORY`).

### Compra: on-demand, Savings Plan ou Spot?

| | Custo `t3.small` | Quando faz sentido |
|---|---|---|
| On-demand | US$ 15,18 | Primeiros meses, enquanto o desenho muda |
| **Savings Plan Compute 1 ano, sem entrada** | ~US$ 10 | Quando estabilizar. Compromisso de 1 ano — só depois de decidir que o projeto fica no ar |
| Spot | ~US$ 4,55 | **Só se o banco estiver no RDS.** Interrupção derruba a instância inteira; com o Postgres em container, você perde o que não tiver sido escrito no volume e leva um restart não planejado do banco |

**Recomendação:** on-demand nos primeiros meses (créditos cobrem), Savings Plan quando estabilizar. Spot é falsa economia enquanto o banco morar na instância.

---

## 7. Cenários de arquitetura

Quatro desenhos. Todos usam ECS com capacity provider EC2 — o que muda é o banco, o ingresso e quantas instâncias.

### Cenário A — Vitrine (a referência de mercado)

O desenho que um time de produção usaria. Está aqui para contraste — **não é o recomendado**.

```
                     Internet
                        │
                   ┌────▼────┐
                   │CloudFront│
                   └────┬────┘
                   ┌────▼────┐  ACM: TLS
                   │   ALB   │  health check + rolling deploy
                   └────┬────┘
            ┌───────────┴───────────┐
      ┌─────▼─────┐           ┌─────▼─────┐   ASG 2 instâncias,
      │  EC2 AZ-a │           │  EC2 AZ-b │   subnets privadas + NAT
      │ api/front │           │ api/front │
      └─────┬─────┘           └─────┬─────┘
            └───────────┬───────────┘
                  ┌─────▼──────┐
                  │RDS Multi-AZ │  subnets isoladas
                  └────────────┘
```

| Item | Custo/mês |
|---|---|
| 2× `t3.small` | US$ 30,36 |
| ALB (+ IPv4 por AZ) | ~US$ 25 |
| NAT Gateway (1 AZ) | ~US$ 33 |
| RDS Multi-AZ + storage espelhado | ~US$ 28 |
| EBS, logs, ECR, Route 53 | ~US$ 7 |
| **Total** | **~US$ 123/mês** |

**Veredito:** queima os US$ 200 de crédito em 7 semanas para servir cinco visitas por semana. Documente-o (como está aqui) e explique numa entrevista *por que você não o escolheu*.

### Cenário B — Enxuto com RDS

Uma instância para os containers, banco gerenciado.

```
   Internet ──► CloudFront ──► Elastic IP ──► EC2 t3.small (ECS)
                (TLS grátis)                    ├── caddy   :80/:443
                                                ├── front   :3000
                                                └── api     :3001
                                                        │
                                                  ┌─────▼──────┐  subnet privada
                                                  │RDS t4g.micro│ Single-AZ, PITR 7d
                                                  └────────────┘
```

| Item | Custo/mês |
|---|---|
| EC2 `t3.small` | US$ 15,18 |
| EBS 30 GB (raiz) | US$ 2,40 |
| Elastic IP | US$ 3,65 |
| RDS `db.t4g.micro` + 20 GB | US$ 13,98 |
| CloudFront, ACM, Parameter Store | US$ 0 |
| Logs, ECR, snapshots | ~US$ 1,50 |
| **Total** | **~US$ 37/mês** |

Com o banco fora da instância, a RAM sobra (~1,5 GB livres) e a instância vira **descartável** — pode ser Spot (−70%, total cai para ~US$ 26), pode ser substituída pela ASG sem cerimônia, e o deploy não tem nada a preservar. É o desenho mais **operacionalmente tranquilo** da lista.

### Cenário C — Tudo na instância (**recomendado**)

Mesmo desenho, com o Postgres em container e um volume EBS dedicado.

```
   Internet ──► CloudFront ──► Elastic IP ──► EC2 t3.small (ECS container instance)
                (TLS grátis)                   │
                                               │  service "app"  (deploy frequente)
                                               ├── caddy    :80/:443  ──► front/api
                                               ├── front    :3000
                                               ├── api      :3001
                                               │
                                               │  service "data" (quase nunca muda)
                                               └── postgres :5432
                                                        │
                                                 /var/lib/postgresql
                                                        │
                                                 EBS gp3 10 GB dedicado
                                                        │
                                        ┌───────────────┴───────────────┐
                                  snapshot diário (DLM)          pg_dump diário → S3
                                  = consistente-em-crash         = consistente-em-aplicação
```

| Item | Custo/mês |
|---|---|
| EC2 `t3.small` | US$ 15,18 |
| EBS 30 GB (raiz) + 10 GB (dados) | US$ 3,20 |
| Snapshots (7 diários incrementais) | ~US$ 0,50 |
| Elastic IP | US$ 3,65 |
| S3 (dumps, com lifecycle) | ~US$ 0,10 |
| CloudFront, ACM, Parameter Store | US$ 0 |
| Logs, ECR | ~US$ 1,20 |
| Domínio (R$ 40/ano) | ~US$ 0,65 |
| **Total** | **~US$ 25/mês** (~US$ 20 com Savings Plan) |

**Atende aos quatro requisitos ao mesmo tempo:** API em container no ECS ✅, banco em container próprio, isolado e com backup ✅, front na AWS ✅, e o menor custo possível sem sair da AWS ✅.

**Dois pontos de atenção que este cenário cria** (ambos com solução na §8 e §12):

1. **O volume de dados precisa sobreviver à troca da instância.** Se a ASG substituir a instância, o volume raiz vai junto. O volume de dados tem que ser separado, e o user-data precisa reanexá-lo (`aws ec2 attach-volume` por ID) e montá-lo antes do agente ECS subir. São ~10 linhas — mas se não existirem, você descobre isso no pior dia possível. Como o volume vive numa AZ, **a ASG fica presa a uma AZ** (o que você já aceitou ao escolher Single-AZ).
2. **Deploy tem alguns segundos de indisponibilidade.** Com uma instância e portas fixas, a task antiga precisa parar antes de a nova subir: `minimumHealthyPercent: 0`, `maximumPercent: 100`. Separar `app` e `data` em dois services garante que **o banco não reinicia a cada deploy** — é o motivo de existirem dois services em vez de um.

### Cenário D — Fuga (para quando os créditos acabarem)

Instância `t3.micro` em Spot + banco em Neon/Supabase (tier grátis de Postgres gerenciado, com backup incluso) + front na Vercel.

| Item | Custo/mês |
|---|---|
| EC2 `t3.micro` Spot | ~US$ 2,30 |
| EBS 20 GB + Elastic IP | ~US$ 5,25 |
| Banco (Neon/Supabase free) | US$ 0 |
| **Total** | **~US$ 8/mês** |

Foge do requisito "banco na AWS" e o tier grátis hiberna quando ocioso. Mas é como manter o link do portfólio **vivo por anos** — com o Terraform do Cenário C no repositório, provando que você sabe fazer o desenho completo.

### Comparativo

| | A — Vitrine | B — RDS | **C — Tudo na instância** | D — Fuga |
|---|---|---|---|---|
| Custo/mês | ~US$ 123 | ~US$ 37 | **~US$ 25** | ~US$ 8 |
| Créditos duram | ~7 semanas | ~5 meses | **~8 meses** | ~2 anos |
| API em container no ECS | ✅ | ✅ | ✅ | ✅ |
| Banco isolado com backup | ✅ PITR + Multi-AZ | ✅ PITR | ⚠️ backup seu, sem PITR | ⚠️ fora da AWS |
| Front na AWS | ✅ | ✅ | ✅ | ❌ |
| RAM livre na instância | folgada | ~1,5 GB | ~800 MB | apertada |
| Instância pode ser Spot | sim | **sim** | não | sim |
| Deploy sem downtime | ✅ | ❌ (~30 s) | ❌ (~30 s) | ❌ |
| Operação sua | baixa | média | **alta** | baixa |

**Recomendação: comece no C.** Ele entrega os quatro requisitos pelo menor custo, e a distância dele para o A é toda em itens que você sabe nomear e justificar — que é exatamente o que se espera de quem entende de infraestrutura. **Migrar para o B é uma mudança de poucas linhas no Terraform** no dia em que os dados começarem a importar de verdade (§8).

---

## 8. O banco: EBS é mais barato que RDS? (sim, ~US$ 12,65/mês)

A comparação direta, para o porte deste app:

| | **Postgres em container + EBS** | **RDS `db.t4g.micro`** |
|---|---|---|
| Compute | **US$ 0** — divide a instância que você já paga | US$ 11,68 (2 vCPU, 1 GB dedicados) |
| Storage | 10 GB gp3 × US$ 0,08 = **US$ 0,80** | 20 GB (mínimo) × US$ 0,115 = **US$ 2,30** |
| Backup | 7 snapshots incrementais ≈ US$ 0,45 + dumps no S3 ≈ US$ 0,10 | Grátis até 100% do storage provisionado |
| **Total/mês** | **~US$ 1,35** | **~US$ 13,98** |
| **Diferença** | | **US$ 12,63/mês ≈ R$ 70/mês ≈ R$ 840/ano** |

Vale notar dois detalhes de preço que costumam passar batido:

- **O GB do RDS é 44% mais caro que o do EBS** (US$ 0,115 vs US$ 0,08) — é o mesmo gp3 por baixo, com o gerenciamento embutido no preço.
- **O RDS tem mínimo de 20 GB** para Postgres. Este app não usa 1 GB tão cedo, então metade do storage que você paga no RDS é piso comercial, não necessidade.
- **Desempenho de disco é equivalente**: gp3 entrega 3.000 IOPS e 125 MB/s de baseline em qualquer tamanho, dos dois lados.

### O que você compra com os US$ 12,63

Custo não é o único eixo. O que o RDS entrega e o container não:

| | Container + EBS | RDS |
|---|---|---|
| **PITR** (restaurar para "14:32 de terça") | ❌ Só volta ao snapshot/dump mais recente — perde até 24 h | ✅ Restauração ao segundo, últimos 7–35 dias |
| Restauração | Procedimento que **você escreve e testa** | Um comando; a AWS cria a instância nova |
| Patch de segurança do Postgres | Sua (trocar tag da imagem, reiniciar) | Janela de manutenção automática |
| Sobreviver à troca da instância | Exige reanexar o volume via user-data (§7-C) | Independente da instância — é outro recurso |
| RAM | Disputa com front e API na mesma máquina | 1 GB dedicado, sem vizinho barulhento |
| Métricas de query lenta | Você instrumenta | Performance Insights, 7 dias, grátis |
| Upgrade de versão maior | Manual, com dump/restore | Assistido |
| Segurança de rede | Rede Docker + SG da instância | Subnet privada, sem IP público, SG dedicado |

### Recomendação

**Comece com o container + EBS (Cenário C), com uma condição não-negociável:** o backup precisa existir de verdade **antes** de o app receber dado que importe. Isso significa três coisas, todas baratas:

1. **Snapshot diário do volume de dados via DLM** (Data Lifecycle Manager — o serviço é grátis, você paga só o storage do snapshot), com retenção de 7 dias.
2. **`pg_dump` diário para o S3**, como ECS Scheduled Task — exatamente o mesmo padrão do `purgeTokens` (§1.8), com lifecycle para Glacier IR depois de 30 dias. Isso importa **mais** que o snapshot: snapshot de volume com Postgres escrevendo é consistente-em-crash, não consistente-em-aplicação. O dump é o backup que você realmente vai usar.
3. **Restaure uma vez, de propósito**, e anote quanto tempo levou (RTO) e quanto dado se perderia (RPO). Duas linhas no README. Backup nunca restaurado é hipótese, não backup — e essa é a diferença que uma entrevista de infra procura.

**Migre para o RDS (Cenário B) quando** qualquer uma destas for verdade: o app tiver usuário real além de você; a RAM da instância começar a apertar; você quiser PITR; ou o tempo gasto operando o Postgres passar a valer mais que R$ 70/mês. No Terraform é trocar um bloco por outro e mudar o `DATABASE_URL` no Parameter Store.

> **O que não fazer:** rodar o Postgres em container **sem** volume EBS dedicado (dado dentro da camada gravável do container morre no primeiro `docker rm`), ou com o volume no disco raiz (some junto com a instância). Se for economizar aqui, economize com método.

---

## 9. O front: onde deployar

Requisito de partida (§1.2): precisa de um servidor Node. Isso elimina S3, GitHub Pages e qualquer hospedagem estática.

**Com ECS EC2, a resposta mudou em relação ao Fargate:** o custo marginal de rodar o front na instância é **RAM, não dinheiro**. Ele cabe nos 2 GB (§6). Então o seu requisito nº 3 — "front na AWS, para conhecimento prático" — sai de graça.

| Opção | Custo | A favor | Contra |
|---|---|---|---|
| **Container na mesma instância** | **US$ 0 marginal** | Tudo na AWS; mesma task definition, mesmo pipeline; `API_URL` vira constante (§1.3); comunicação com a API não sai da máquina | ~350 MB de RAM; deploy do front derruba o front por ~30 s |
| Vercel (Hobby) | US$ 0 | Feita para Next 16; preview por PR; HTTPS e domínio inclusos; tira RAM da instância | Não é AWS; licença Hobby é **uso pessoal/não comercial**; 100 GB de banda e 1M edge requests/mês — e **todo** `/api/*` passa por lá |
| AWS Amplify Hosting | ~US$ 1–3/mês | O "Vercel da AWS": build por push, TLS, domínio e preview por PR prontos | **Documentação declara suporte só até o Next 15** — o front está no 16 (ver abaixo); builda do fonte, furando o "build once, promote everywhere" |
| S3 + CloudFront | ~US$ 0 | — | **Inviável** sem refatorar o front |

### Por que o Amplify Hosting está fora (hoje)

O Amplify seria o candidato natural — é AWS, é gerenciado e custaria ~US$ 1–3/mês para este tráfego (US$ 0,01/min de build, US$ 0,15/GB servido, US$ 0,30 por milhão de requisições SSR, US$ 0,20/GB-hora). O custo não é o problema. São dois outros:

1. **Versão do Next.** A documentação da AWS declara suporte a Next.js **12 a 15**; o `package.json` do front está em `^16.3.0`. E não é um detalhe qualquer: o Next 16 renomeou o middleware de `middleware.ts` para **`proxy.ts`** — exatamente o arquivo da guarda de rota, que lê o cookie, decodifica o JWT e chama `/auth/refresh` antes de renderizar. Um adaptador que não conhece o Next 16 ou quebra no build ou, pior, sobe **sem executar esse arquivo** — uma falha silenciosa de controle de acesso, com as páginas renderizando normalmente.
2. **Conflito com o pipeline.** O Amplify builda do **código-fonte**, não consome imagem. Todo o CI/CD dos três repositórios é "build once, promote everywhere", com o E2E deste repositório como portão. O Amplify faria o próprio build a cada push na `main`, fora desse portão. Conciliável (apontar o Amplify para um branch `production` que o job `promote` avança por fast-forward depois do E2E), mas é complexidade nova para um problema que o container na instância não tem.

Vale registrar também o que ele ensina: um PaaS gerenciado — ele **esconde** justamente as primitivas (VPC, ECS, IAM, EBS, CloudFront) que o requisito de "conhecimento prático de AWS" busca e que o lado da API já entrega.

**Revisitar se:** a AWS anunciar suporte a Next 16, **ou** aparecer um motivo forte para tirar o front de container. Nesse segundo caso, a Vercel continua sendo o plano B melhor — é a implementação de referência do Next e suporta versão nova no dia do lançamento, que é onde o Amplify falha.

**Recomendação: container na instância.** É o que atende ao requisito nº 3 sem custo, e o `API_URL` deixa de ser um problema de build por ambiente. Mantenha a Vercel no bolso como plano B: se a RAM apertar, mover o front para lá libera ~350 MB em cinco minutos e não custa nada.

### O Dockerfile do front — feito

Esta seção pedia quatro mudanças antes de o front subir no ECS. As quatro estão no código:

| Pedido | Estado |
|---|---|
| `output: 'standalone'` + multi-stage | ✅ `next.config.ts` + `Dockerfile` em três estágios (`deps`, `build`, `runtime`) |
| `npm ci` em vez de `npm install` | ✅ no estágio `deps` |
| `USER node` antes do `CMD` | ✅ |
| `HEALTHCHECK` no Dockerfile | ✅ usando o `fetch` global do Node 24, sem instalar `curl` |

**A estimativa de "~150 MB" estava errada, e a correção importa.** O `standalone` derruba o *payload da
aplicação* de >1 GB para **~47 MB** (medido: 43 MB de standalone + 2 MB de `.next/static` + 2 MB de
`public`), mas a **imagem final fecha em ~390 MB** — a base `node:24-bookworm-slim` responde sozinha
por uns 250 MB dela, e trocar de base ficou fora do escopo. Ainda assim o objetivo foi atingido: três
tags de 390 MB convivem numa raiz de 30 GB; três de 1 GB não conviviam.

O entrypoint virou `node server.js` — o servidor que o próprio Next gera dentro do standalone — no
lugar de `npm start`/`next start`, que exigia o `node_modules` de produção inteiro na imagem, ou seja,
exatamente o que o standalone eliminou.

### A armadilha do `API_URL`: build-time **e** runtime

Anote isto antes de escrever a task definition da Fase 6. O front precisa de `API_URL` nos dois
momentos, por dois motivos diferentes:

- **Build-time**, como `--build-arg`: congela o destino do rewrite `/api/*` no `routes-manifest.json`
  (§1.3). Sem ele, a imagem sai com o placeholder `http://localhost:8080`.
- **Runtime**, como variável de ambiente do container: o `src/proxy.ts` (a guarda de rota do Next 16)
  chama `POST /auth/refresh` **a cada request** e lê `process.env.API_URL` no boot. Sem a variável no
  ambiente, ele lança `Error: Variável de ambiente API_URL não configurada.`

Os dois valores têm de ser **o mesmo**. Na topologia final (tasks separadas — ver
[`separacao-de-tasks-front-api.md`](./separacao-de-tasks-front-api.md)) esse valor é `http://api:8080`
(`3001` até 21/08/2026), com `api` resolvido pelo `extraHosts` da task do front. No build ele vem da Repository Variable
`API_URL` do repositório do front, configurada à mão na interface do GitHub e **fora de qualquer
automação** — se ela não existir, o CI publica a imagem com o placeholder e o erro só aparece no ECS.

**O sintoma de esquecer o runtime:** toda página responde 500 e o container nunca fica `healthy` (o
`HEALTHCHECK` bate na raiz), então o service entra em loop de restart. O evento do ECS só diz que a
task foi derrubada por health check; a causa está no log da aplicação, em
`Error: Variável de ambiente API_URL não configurada.` — vale saber disso antes, porque a mensagem do
ECS aponta para o lugar errado.

**O sintoma de esquecer o build-time é pior, porque não parece um erro de infraestrutura.** O
container sobe, fica `healthy`, as páginas renderizam autenticadas — e **toda chamada feita pelo
navegador falha**, com o corpo da tela mostrando "Erro na comunicação com o servidor". Nada nos
eventos do ECS nem nos logs indica a causa: do ponto de vista do servidor, está tudo funcionando.

> ⚠️ **Aconteceu exatamente assim em 20/08/2026.** A variable não existia, a imagem foi publicada com
> o placeholder e o sistema ficou parcialmente quebrado em produção por um ciclo inteiro de deploy.
> Configurada em 21/08/2026. O diagnóstico completo, as quatro alternativas avaliadas e a correção
> aplicada estão em
> [`problema-rewrite-api-build-time.md`](../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md).
>
> **A lição estrutural:** a assimetria entre os dois mecanismos (um resolvido em build, outro em
> runtime) é a própria causa. Enquanto ela existir, o bug pode voltar a cada mudança de endereço — e a
> Fase 7 é uma dessas. A saída é o Route Handler da Abordagem B, que unifica tudo em runtime.

---

## 10. Rede, ingresso e TLS

### O Elastic IP resolve o problema que o Fargate criava

Com Fargate, o IP da task muda a cada deploy e você precisa de ALB (US$ 25/mês) ou de uma Lambda atualizando DNS. **Na EC2 com Elastic IP o endereço é fixo** — e o EIP custa os mesmos US$ 3,65 do IP auto-atribuído, ou seja, estabilidade de graça. Some daí a necessidade de load balancer.

Bônus: um EIP ganha um nome DNS público estável (`ec2-<ip-com-tracos>.compute-1.amazonaws.com`), e **o CloudFront exige nome DNS como origem — não aceita IP**. É esse nome que você aponta como origin.

### Modo de rede das tasks

| Modo | Custo | Quando usar aqui |
|---|---|---|
| **`bridge`** | US$ 0 | **Recomendado.** Sem ENI extra, sem IP por task. Caddy publica 80/443 no host; front e API não publicam nada; o banco publica 5432 só no host |
| `awsvpc` | US$ 0 (mas consome ENIs) | Security group **por task** (mais granular). `t3.small` tem 3 ENIs: uma da instância, duas para tasks. Exige Cloud Map para o front achar a API |
| `host` | US$ 0 | Sem isolamento de rede. Não |

**Comunicação entre os services:** em `bridge`, containers da **mesma task** se enxergam por nome (`links`), e é assim que `caddy → front → api` funciona. Para o service `app` alcançar o service `data` (Postgres), o caminho estável é o **IP privado da instância** (fixo, é a mesma máquina) na porta publicada. A alternativa "correta" é **ECS Service Connect / Cloud Map**, que dá nomes DNS internos (`db.cronos.local`) — vale como upgrade quando o desenho crescer; não vale a complexidade na primeira subida.

### Entrada HTTPS: três montagens

| Montagem | Custo | Observação |
|---|---|---|
| **CloudFront → Caddy (HTTP) na instância** | US$ 0 | TLS válido e grátis em `*.cloudfront.net`, **sem precisar de domínio**; cache na borda de São Paulo; esconde o IP de origem. O salto CloudFront→origem é HTTP: restrinja o SG à prefix list `com.amazonaws.global.cloudfront.origin-facing` e exija um header secreto |
| **Caddy com Let's Encrypt direto** | US$ 0 | TLS ponta a ponta, renovação automática. **Exige domínio próprio.** Sem CloudFront, sua instância recebe tráfego bruto da internet |
| **CloudFront → Caddy (HTTPS)** | US$ 0 | O melhor dos dois: TLS na borda e na origem. Exige domínio (o CloudFront não aceita certificado self-signed na origem) |

**Recomendação:** comece na primeira montagem (funciona no dia 1, sem domínio), e vá para a terceira assim que o domínio estiver registrado. As duas usam a mesma infra — muda a config do Caddy e o `origin protocol policy` da distribuição.

---

## 11. Domínio e TLS

### Primeiro: você talvez não precise comprar nada

Certificado TLS válido, de graça, sem domínio próprio: **CloudFront** entrega `https://d1a2b3c4.cloudfront.net` com certificado incluído. O que **não** entrega: ALB (só HTTP no DNS dele) e a instância nua — o ACM só emite certificado para domínio que você controla.

Como o front precisa de **uma** origem bonita e a API nem é exposta (§1.4), a pergunta se reduz a: o link do portfólio vai ser `d1a2b3c4.cloudfront.net` ou `cronos.com.br`?

### Opções

| Opção | Custo/ano | Observações |
|---|---|---|
| **`.com.br` no Registro.br** | **R$ 40 fixo** | Sem promoção de primeiro ano; WHOIS privado automático para pessoa física; exige CPF. Melhor custo-benefício do Brasil |
| `.com` (Namecheap, Porkbun, Cloudflare) | US$ 10–15 | O Cloudflare Registrar vende a preço de custo, sem markup na renovação |
| TLDs baratos (`.xyz`, `.site`, `.online`) | US$ 1–3 no 1º ano | **Cuidado com a renovação** — costuma pular para US$ 20–40. Alguns têm reputação ruim e caem em filtro de spam |
| Registrar no **Route 53** | ~US$ 14 | Integração perfeita, preço pior. Registro e DNS podem ficar em lugares diferentes |
| `*.cloudfront.net` | **grátis** | Certificado válido, zero configuração. Sinaliza "projeto de estudo" — o que é verdade |
| **DuckDNS**, `nip.io`, `sslip.io` | grátis | Subdomínios dinâmicos. Úteis para homelab e para Let's Encrypt via DNS-01; num link de portfólio, pesam contra |
| Freenom (`.tk`, `.ml`, `.ga`) | — | **Morto.** Não registra desde 2023. Tutorial que sugerir isso é tutorial velho |

### Recomendação

**Compre o `.com.br` no Registro.br (R$ 40/ano ≈ R$ 3,33/mês).** É ~2,5% do custo mensal e a única peça que fica com você depois de destruir a infra. Um domínio próprio muda a leitura do link no currículo.

Montagem, toda gratuita a partir daí:

```
Registro.br (R$ 40/ano)
      │  nameservers apontados para
      ▼
Cloudflare DNS (grátis)
      ├── cronos.com.br         → CNAME (flattening) → CloudFront
      └── origin.cronos.com.br  → A → Elastic IP     (Caddy + Let's Encrypt)
                                     │
                               ACM (grátis) emite o certificado do CloudFront,
                               validado por registro DNS
```

Por que Cloudflare e não Route 53: a hosted zone do Route 53 custa US$ 0,50/mês (US$ 6/ano — 40% do preço do domínio) e, neste desenho, você não usa nada do que ela tem de especial. Se o Terraform precisar automatizar registros DNS, aí o Route 53 se paga em conveniência.

### Detalhes que quebram o login se passarem batido

- Depois de trocar o domínio, `FRONTEND_URL` na API precisa apontar para a origem **exata** de produção (`https://cronos.com.br`) — é ela que alimenta o `Access-Control-Allow-Origin` com `credentials: true` e o redirect final do Google.
- Com Google Login ligado, `GOOGLE_CALLBACK_URL` e os *Authorized redirect URIs* no Google Cloud Console apontam para o **front** (§1.6), não para a API.
- `www` e apex devem responder a mesma coisa, com um redirecionando para o outro. Dois domínios servindo o app com cookies separados é uma classe de bug difícil de enxergar.

---

## 12. Boas práticas que custam zero

O que não aumenta a conta e melhora bastante o desenho — e é exatamente o que se olha num portfólio de infra.

### Específicas de ECS sobre EC2

- **SSM Session Manager em vez de SSH.** Instance profile com `AmazonSSMManagedInstanceCore` (o agente já vem na AMI ECS-optimized), **porta 22 fechada no SG, nenhuma chave PEM para vazar**, e sessão auditada no CloudTrail. É a melhoria de segurança mais barata da lista.
- **Três papéis IAM distintos**, sem misturar: `ecsInstanceRole` (instance profile — registra a instância no cluster), `executionRole` (o agente puxa imagem do ECR e lê segredos do SSM) e `taskRole` (o que a **aplicação** pode chamar na AWS — aqui, quase nada). Quase todo tutorial funde os três.
- **AMI ECS-optimized do Amazon Linux 2023**, atualizada por **substituição** (novo launch template + rotação da ASG), não por `dnf update` manual. Host imutável é o que evita a instância virar bicho de estimação.
- **ASG com `min=max=desired=1` e health check** — não é para escalar, é para **auto-recuperação**: instância morta é substituída sozinha. Grátis. Lembre da §7-C: com o banco em container, a substituição precisa reanexar o volume de dados.
- **Limpeza automática de imagens.** O agente ECS já expira imagens antigas (`ECS_IMAGE_CLEANUP_INTERVAL`, `ECS_IMAGE_MINIMUM_CLEANUP_AGE`). Confirme que está ligado: numa raiz de 30 GB, três tags de uma imagem de 1 GB enchem o disco e o sintoma (task que não sobe) não parece disco cheio.
- **Log driver `awslogs`** com retenção de 7 dias. Sem isso, os logs ficam só no disco da instância e somem com ela.
- **Swap de 1–2 GB** (§6).

### Gerais

- **OIDC entre GitHub Actions e AWS**, com role assumida por federação — em vez de `AWS_ACCESS_KEY_ID` como secret. Chave estática de longa duração em repositório é a observação mais fácil de fazer num code review de pipeline.
- **SSM Parameter Store SecureString** para `JWT_SECRET`, `DATABASE_URL` e afins, referenciados no bloco `secrets` da task definition. Tier Standard é grátis; o Secrets Manager cobra US$ 0,40 por segredo.
- **Security group referenciando security group / prefix list**, nunca CIDR aberto. Entrada 80/443 só da prefix list do CloudFront; 22 fechada (Session Manager); 5432 não sai da instância.
- **AWS Budget com alerta** antes do primeiro recurso.
- **Lifecycle policy no ECR**: manter as 5 imagens mais recentes.
- **Desligar fora do horário.** EventBridge Scheduler chamando `StopInstances` às 23h e `StartInstances` às 8h corta ~60% da EC2 (o EBS continua sendo cobrado). Com Elastic IP o endereço sobrevive ao stop/start — mais uma razão para ele existir.
- **Alarme de instância/serviço fora do ar** no CloudWatch → SNS → seu e-mail. Dez alarmes por mês são grátis.
- **Terraform** (ou CDK) para tudo, com state num bucket S3 versionado. É o que torna viável o `apply`/`destroy` sob demanda da §0 — e é o artefato de portfólio mais valioso deste documento inteiro.
- **Deploy = `run-task` (migrate) → `update-service` (app)**, nessa ordem, com `wait services-stable` no fim. É a tradução direta do `depends_on: service_completed_successfully` que o `docker-compose.yml` deste repositório já expressa.

---

## 13. Roteiro de implementação

Cada fase entrega algo verificável e não depende da seguinte.

| Fase | O que fazer | Entregável |
|---|---|---|
| **0. Conta** | Conta AWS, MFA no root, usuário IAM separado, **Budget de US$ 10 com alerta**, região fixada em `us-east-1` | Conta segura e com trava de gasto |
| **1. Base** | VPC (subnet pública, **sem NAT**), SGs, ECR, parâmetros no SSM, papéis IAM (os três da §12) | `terraform apply` reproduzível |
| **2. Instância** | Launch template com AMI ECS-optimized AL2023 (**ARM64** — a instância real é `t4g.micro`, §-1), ASG `min=max=1`, Elastic IP, swap, cluster ECS, Session Manager funcionando | `aws ecs list-container-instances` mostrando a instância |
| **3. Banco** | Volume EBS de dados + user-data que reanexa e monta; service `data` com o Postgres; DLM de snapshot | `psql` respondendo de dentro da instância |
| **4. API** | Task definition do service `app`, log group com retenção, `run-task` da migration | `GET /health` respondendo na porta da instância |
| **5. Ingresso** ✅ | Caddy no service `app`, CloudFront apontando para o DNS do EIP | **Concluída (20/08/2026)** — no ar em `https://d3c5d6t3539m1d.cloudfront.net`. Ver [`ingresso-aws.md`](./ingresso-aws.md) |
| **6. Front** ✅ | Front em **task e service próprios** (`cronos-front`), não como container do service `app` — decisão revista em [`separacao-de-tasks-front-api.md`](./separacao-de-tasks-front-api.md) | **Concluída (20-21/08/2026).** Ver [`front-aws.md`](./front-aws.md). Subiu com um bug: a Repository Variable `API_URL` faltando quebrou todo o caminho client-side; corrigido em 21/08 (§9) |
| **7. Domínio** | `.com.br` no Registro.br, DNS na Cloudflare, ACM, Caddy com Let's Encrypt na origem. **Agrupar aqui:** `FRONTEND_URL` real na API, as 4 variáveis do Google OAuth, as 5 do SMTP, o Route Handler que tira o rewrite do build-time (§9) e, se quiser, a padronização da porta da API em `8080` | `https://cronos.com.br`, com login pelo Google e email de recuperação de senha funcionando |
| **8. Pipeline** | OIDC no GitHub Actions; estender o job `promote` para espelhar no ECR e disparar `run-task` + `update-service` | Push na `main` → produção, sem passo manual |
| **9. Operação** | EventBridge para a purga de tokens e para o `pg_dump`; alarmes; scheduler de desligamento noturno; **teste de restauração do backup** | Runbook curto no README |
| **10. Opcional** | Multi-arch no CI (só faz sentido se voltar a existir um destino x86 — hoje não existe); migrar o banco para o RDS quando os dados importarem | Changelog do README |

O que já está pronto e não precisa ser refeito: imagens versionadas no GHCR, migration como processo separado, healthcheck real, segredos fora da imagem, E2E validando a combinação antes de promover. O deploy na AWS é a continuação desse pipeline — não uma reconstrução dele.

---

## 14. Decisões em aberto

1. ~~**Banco em container ou RDS**~~ — **decidido: container + EBS.** Provisionado em 19/08/2026, ver [`banco-de-dados-aws.md`](./banco-de-dados-aws.md). Os gatilhos de migração para RDS (§8) continuam valendo.
2. ~~**x86 agora ou multi-arch + Graviton depois**~~ — **decidido e concluído: Graviton, arm64 puro na API e no front.** Ver §-1 e §1.10. A mudança do front foi mergeada na `main` (PR #7) e a imagem `linux/arm64` está publicada e rodando em produção. A instância, por sinal, não é mais `t4g.micro`: virou **`t4g.small`** (2 GB) para comportar os quatro containers — ver item 4.
3. ~~**Google Login em produção: liga ou não?**~~ — **decidido: liga, na Fase 7.** Subiu sem, como planejado. Ligar exige as **4 variáveis do grupo** (`GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL`, `COOKIE_SESSION_SECRET`) na task `cronos-app` — o [`env.ts`](../../sistema-controle-despesas-api/src/config/env.ts) as valida como "tudo ou nada" e **recusa-se a subir** se só algumas estiverem lá. O `GOOGLE_CALLBACK_URL` precisa da URL pública real, registrada também no Google Cloud Console, e `FRONTEND_URL` (o redirect final do OAuth) tem de deixar de ser placeholder junto — por isso depende do domínio. Atenção adicional: `/api/auth/google` é uma **navegação do navegador**, não um `fetch`; se o Route Handler da Abordagem B entrar antes, ele precisa repassar o `Location` do 302 em vez de segui-lo.
4. ~~**Front na instância ou na Vercel**~~ — **decidido: na instância, e concluído.** A instância subiu para `t4g.small` (2 GB) e o RSS real do front, medido em 20/08/2026, foi de **55,73 MiB** — uma ordem de grandeza abaixo da estimativa de ~250–400 MB, que era anterior ao build `standalone`. Os quatro containers cabem com folga: sobraram 694 MB livres no cluster. Ver [`front-aws.md`](./front-aws.md) §10.1.
5. **Terraform ou CDK.** Terraform é mais comum em vaga de infra; o CDK deixa você em TypeScript, a linguagem dos três repositórios. Sugestão: Terraform, pelo mercado.
6. **Savings Plan de 1 ano** (§6): só depois de decidir que o projeto fica no ar. Economiza ~US$ 5/mês, mas amarra 12 meses.
7. **SMTP: qual provedor?** As 5 variáveis (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, `MAIL_FROM`) também são "tudo ou nada". Sem elas, a recuperação de senha **completa o fluxo sem nunca enviar o email** — degradação graciosa que é invisível para quem testa só a resposta HTTP. Vale decidir junto com a Fase 7, porque `MAIL_FROM` num domínio próprio tem entregabilidade melhor que num `*.cloudfront.net`. O Amazon SES é a opção óbvia dentro da conta, mas exige sair do sandbox para enviar a destinatário não verificado — o que leva alguns dias e é o tipo de fricção que vale descobrir antes, não durante.
8. ~~**Padronizar a porta da API em `8080`?**~~ — **decidido e concluído em 21/08/2026**, fora da
   Fase 7 (não valeu esperar — o custo real foi bem menor que o levantado). Código e infra AWS
   (Security Group, `cronos-app:2`, `cronos-front:3`) todos em `8080`, verificado ponta a ponta via
   SSM. Falta só a Repository Variable `API_URL` do repo do front (sem efeito em produção); ver
   [`problema-rewrite-api-build-time.md`](../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md) §14.4.

---

## Referências

- [AWS Free Tier — US$ 200 em créditos e plano gratuito de 6 meses](https://aws.amazon.com/about-aws/whats-new/2025/07/aws-free-tier-credits-month-free-plan/)
- [Amazon ECS — preços](https://aws.amazon.com/ecs/pricing/) (sem taxa sobre EC2)
- [Amazon EC2 — preços sob demanda](https://aws.amazon.com/ec2/pricing/on-demand/)
- [Amazon EBS — preços](https://aws.amazon.com/ebs/pricing/)
- [Amazon RDS — preços](https://aws.amazon.com/rds/postgresql/pricing/)
- [Cobrança de endereços IPv4 públicos](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights)
- [Elastic Load Balancing — preços](https://aws.amazon.com/elasticloadbalancing/pricing/)
- [Registro.br](https://registro.br/) — `.com.br` a R$ 40/ano
- [`docker-e-arquitetura.md`](./docker-e-arquitetura.md) — o estado atual dos três repositórios
