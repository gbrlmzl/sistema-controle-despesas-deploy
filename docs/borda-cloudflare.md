# Troca da borda: CloudFront → Cloudflare — CRONOS

Plano de execução da substituição da camada de entrada montada na **Fase 5**
([`ingresso-aws.md`](./ingresso-aws.md)) pela borda da Cloudflare, agora que existe domínio próprio.
Cobre também a parte de **domínio** da Fase 7 do roteiro de [`arquitetura-aws.md`](./arquitetura-aws.md) §13.

> **Estado:** **planejado, não executado.** Escrito em 22/08/2026.
> **Pré-requisito já cumprido:** domínio `gabrielmizael.com` comprado no **Cloudflare Registrar** em
> 22/08/2026 — a zona nasce ativa e com os nameservers da Cloudflare, então não há etapa de migração
> de DNS.
> **Hostname alvo:** `cronos.gabrielmizael.com`. A decisão de o apex ser o portfólio e cada projeto
> viver num subdomínio está analisada em
> [`../../docs/arquitetura-portfolio-subdominios.md`](../../docs/arquitetura-portfolio-subdominios.md).
> **O que este documento não faz:** não mexe em API, banco, front, imagens nem CI. A troca é
> inteiramente na borda.
> **Convenção:** o ID da conta aparece como `<conta>`; segredos, como `<...>`. O repositório é público.

---

## 0. O que muda e o que não muda

| Peça | Muda? |
|---|---|
| Task definitions / services `cronos-app`, `cronos-data`, `cronos-front` | ❌ intocados |
| Imagens no ECR (API, front, Caddy) | ❌ intocadas |
| CI dos três repositórios | ❌ intocado |
| Elastic IP `77.112.41.239` | ❌ **permanece** — vira a origem da Cloudflare |
| Parâmetro SSM `/cronos/edge/origin-secret` | ❌ **reaproveitado** — o mesmo header, outro injetor |
| Instância, cluster, security group (o recurso) | ❌ intocados |
| **Certificado Origin CA da Cloudflare** | ✅ **criar** (painel Cloudflare) |
| **Parâmetros SSM `/cronos/edge/origin-cert` e `/origin-key`** | ✅ **criar** |
| **`/etc/caddy/Caddyfile`** | ✅ **reescrever** (ganha um bloco `:443`, mantém o `:80` na transição) |
| **`/etc/caddy/certs/`** na instância | ✅ **criar** |
| **Task definition `cronos-edge`** | ✅ **nova revisão** (publica 443, monta os certificados) |
| **Prefix list gerenciada pelo cliente** com os IPs da Cloudflare | ✅ **criar** |
| **Regra de entrada 443 no Security Group** | ✅ **criar** (restrita à prefix list acima) |
| **Registros DNS na Cloudflare** | ✅ **criar** |
| **Regra 80 no SG + bloco `:80` do Caddy** | ⚠️ **remover na Etapa 9**, não antes |
| **Distribuição CloudFront `E2EHRXTE92L61B`** | ⚠️ **desabilitar na Etapa 9** — nunca apagar de imediato |
| `FRONTEND_URL`, `GOOGLE_*`, `SMTP_*` na task da API | ✅ **agora sim** — ver §11 |

Nenhuma alteração destrutiva até a Etapa 9. Os dois caminhos (CloudFront e Cloudflare) ficam vivos
**ao mesmo tempo** durante toda a execução, e é isso que torna a virada um registro de DNS e o
rollback outro registro de DNS.

### Custo

| Item | Custo/mês |
|---|---|
| Cloudflare (proxy, WAF gratuito, DNS, Bot Fight Mode, 1 regra de rate limit) | **US$ 0** |
| Certificado Origin CA (15 anos) | **US$ 0** |
| Prefix list gerenciada pelo cliente | **US$ 0** |
| Elastic IP | ~US$ 3,65 — **inalterado**, já pago hoje |
| CloudFront | US$ 0 antes, US$ 0 depois (estava dentro do free tier) |
| **Δ real da fase** | **US$ 0,00** |

> **A troca não economiza dinheiro.** O que ela compra é proteção que hoje não existe e uma borda em
> São Paulo. Se a motivação fosse custo, não haveria motivo para mexer.

---

## 1. O que se ganha, e o que se perde

O `ingresso-aws.md` §11.6 registrou "sem WAF, sem rate limiting" como decisão consciente, custeada em
~US$ 5-8/mês de AWS WAF num orçamento de ~US$ 25. A Cloudflare entrega uma versão reduzida disso de
graça — e é essa a razão da troca.

| Ganha | Detalhe |
|---|---|
| **WAF gratuito** | O *Cloudflare Free Managed Ruleset* já vem ligado: um subconjunto do ruleset completo, focado nas vulnerabilidades de maior impacto. **Não é customizável** e não inclui o OWASP Core Ruleset |
| **Bot Fight Mode** | Desafio computacional para tráfego com assinatura de bot conhecido. Também não é customizável nem escopável no plano gratuito |
| **1 regra de rate limiting** | Janela de contagem de **10 s** e bloqueio de **10 s** — o teto do plano gratuito. Atende parcialmente a §11.7 do `ingresso-aws.md` |
| **DDoS não medido** | Proteção L3/L4/L7 sem cota, em todos os planos |
| **Borda de São Paulo** | Resolve a §11.4 do `ingresso-aws.md`: o `PriceClass_100` do CloudFront **excluía** a borda GRU, e o público real é brasileiro |
| **Registrar + DNS + borda no mesmo lugar** | Menos superfície operacional; o TXT do Search Console e os registros do OAuth ficam ao lado do resto |
| **Analytics de borda** | Grátis, e hoje não existe nenhuma observabilidade da borda (§11.3) |

| Perde | Detalhe |
|---|---|
| **Prefix list gerenciada pela AWS** | `com.amazonaws.global.cloudfront.origin-facing` era mantida pela AWS. A lista de IPs da Cloudflare passa a ser **sua** para manter — envelhece em silêncio (§14.4) |
| **Políticas de cache granulares** | O CloudFront tinha `CachingDisabled` como padrão e `CachingOptimized` em `/_next/static/*`. A Cloudflare acerta o mesmo por padrão (não cacheia HTML), mas a granularidade fina exige Cache Rules |
| **Um provedor só no Terraform** | A Fase 8 passa a ter dois providers (`aws` e `cloudflare`) e dois lugares de estado |
| **Rollback trivial** | Hoje é "mexer na distribuição". Depois é "mexer em duas contas". Mitigado mantendo a distribuição **desabilitada**, não apagada |

---

## 2. A montagem escolhida, e as duas alternativas

```
Internet ──HTTPS──▶ Cloudflare ──HTTPS──▶ Caddy :443 ──▶ front :3000 ──▶ api :8080
                    (proxy, WAF)          (cert Origin CA)
                                          (exige X-Origin-Verify)
```

| Montagem | Por quê / por que não |
|---|---|
| **Cloudflare (proxy) → Caddy `:443` com Origin CA** | ✅ **Escolhida.** É o menor delta possível: troca o injetor do header e o terminador de TLS, e não toca em mais nada. A origem continua fechada (prefix list + header), o Caddy continua desacoplando borda de destino |
| **Cloudflare Tunnel (`cloudflared` na instância)** | ❌ **Não agora.** Fecha a entrada por completo — nenhuma porta aberta, nenhum SG, nenhum header. Em troca: mais um container, uma dependência de saída nova, e o *Authenticated Origin Pulls* deixa de ser possível (o túnel é só de saída, não há listener para apresentar certificado de cliente). **Não economiza o EIP**: a instância continua precisando de IPv4 público para *sair* (esta VPC não tem NAT) e a AWS cobra IPv4 público de qualquer forma. Vale reconsiderar quando o `user-data` da §14.1 existir |
| **DNS only (nuvem cinza)** | ❌ Perde tudo o que motivou a troca — WAF, DDoS, Bot Fight Mode — e ainda expõe o IP de origem. Além disso obrigaria o Caddy a emitir Let's Encrypt, com a porta 80 aberta ao mundo |

### O Caddy continua existindo — e agora por um quarto motivo

Os três da Fase 5 seguem valendo (valida o header, desacopla a borda do destino, é onde o TLS mora).
O quarto é novo: **é o Caddy que vai rotear por `Host`** quando o segundo projeto chegar num
subdomínio. Um bloco de site por hostname, sem tocar em DNS nem em borda. Ver
[`../../docs/arquitetura-portfolio-subdominios.md`](../../docs/arquitetura-portfolio-subdominios.md) §4.

---

## 3. A ordem de execução, e por que ela não dá downtime

O Caddy atende **`:80` e `:443` ao mesmo tempo** da Etapa 3 até a Etapa 9:

| Porta | Quem alcança | Termina TLS onde |
|---|---|---|
| `:80` | Só o CloudFront (prefix list da AWS) | No CloudFront |
| `:443` | Só a Cloudflare (prefix list nova) | No Caddy, com o Origin CA |

Enquanto os dois vivem, `https://d3c5d6t3539m1d.cloudfront.net` e `https://cronos.gabrielmizael.com`
servem o mesmo sistema. A virada é o momento em que você passa a divulgar o segundo — e o rollback,
até a Etapa 9, é voltar a usar o primeiro.

1. Certificado Origin CA (painel Cloudflare)
2. Certificado no SSM e na instância
3. `Caddyfile` com o bloco `:443`
4. Nova revisão da task `cronos-edge`
5. Prefix list da Cloudflare + regra 443 no SG
6. DNS e SSL na Cloudflare
7. Proteções gratuitas
8. As variáveis da aplicação que dependem do domínio
9. Verificação → e só então aposentar o CloudFront

---

## 4. Etapa 1 — o certificado Origin CA

No painel da Cloudflare: **SSL/TLS → Origin Server → Create Certificate**.

| Campo | Valor | Por quê |
|---|---|---|
| Tipo de chave | **ECDSA** | Muito menor que RSA — e o parâmetro SSM do tier Standard tem teto de **4 KB** por valor (§5) |
| Hostnames | `gabrielmizael.com`, `*.gabrielmizael.com` | Cobre o apex e todos os projetos de primeiro nível de uma vez. Um certificado só para todas as futuras origens |
| Validade | **15 anos** | É o padrão, e é aceitável porque este certificado **não é confiado publicamente** |

A chave privada aparece **uma única vez**. Copie as duas caixas antes de sair da página.

> **Por que Origin CA e não Let's Encrypt no Caddy:** o LE exigiria ou a porta 80 aberta ao mundo
> (desafio HTTP-01 — exatamente o que a prefix list existe para impedir) ou um token de API da
> Cloudflare guardado na instância para o DNS-01. O Origin CA é grátis, dura 15 anos e é confiado
> **só pela Cloudflare** — que é precisamente a única cliente que esta origem deve ter.
>
> **Consequência esperada:** `curl https://<EIP>` de fora falha com erro de certificado. Isso é o
> desenho funcionando, não um defeito.

---

## 5. Etapa 2 — guardar o certificado

Da sua máquina, com os dois PEMs salvos em arquivos temporários:

```bash
aws ssm put-parameter --region us-east-2 --name /cronos/edge/origin-cert --type SecureString --value file://origin.pem --description "Certificado Origin CA da Cloudflare para gabrielmizael.com"
```

**O que faz:** grava o certificado como `SecureString`, no mesmo prefixo `/cronos/edge/*` que a
execution role já enxerga desde a Fase 5 §4.1. O `file://` evita colar PEM multilinha no shell.

```bash
aws ssm put-parameter --region us-east-2 --name /cronos/edge/origin-key --type SecureString --value file://origin.key --description "Chave privada do certificado Origin CA"
```

**O que faz:** o mesmo para a chave privada. **Apague os dois arquivos locais depois** — a chave não
tem por que sobreviver na sua pasta de trabalho.

> **Por que o SSM se o Caddy lê arquivo, e não variável de ambiente:** a diretiva `tls` do Caddy
> recebe **caminhos**, não conteúdo. Os arquivos vão para o disco da instância (abaixo), como já
> acontece com o `Caddyfile`. O SSM aqui é a **fonte da verdade**, para que o `user-data` da §14.1
> consiga reconstruir a borda sozinho quando o ASG trocar a máquina. Hoje é backup; depois vira
> provisionamento.

Na sessão SSM da instância (`aws ssm start-session --region us-east-2 --target i-0694b1330083a3450`),
crie o diretório e cole o certificado com um heredoc (`sudo tee /etc/caddy/certs/origin.pem`), depois
a chave em `/etc/caddy/certs/origin.key`. Colar aqui em vez de puxar do SSM é deliberado: a
**instance role** não tem permissão de ler `/cronos/*` — o padrão de `AccessDenied` documentado na
Fase 5 §4.2 vale para ela também —, e conceder essa permissão só para uma etapa manual seria ampliar
privilégio pelo motivo errado.

```bash
sudo chmod 644 /etc/caddy/certs/origin.pem && sudo chmod 600 /etc/caddy/certs/origin.key && sudo ls -l /etc/caddy/certs/
```

**O que faz:** ajusta as permissões — `644` no certificado (é público por natureza), `600` na chave —
e lista o diretório para confirmar que os dois arquivos existem antes de você seguir.

> **Atenção ao container:** o `caddy:2-alpine` roda como `root`, então `600` é legível de dentro. Se
> um dia a imagem passar a rodar sem privilégio, esta é a linha que quebra a borda com um erro de
> permissão que não menciona permissão.

---

## 6. Etapa 3 — o `Caddyfile` com os dois caminhos

Substitui o arquivo da Fase 5 §5. O bloco `:80` continua **idêntico** — é o caminho do CloudFront, e
ele só sai na Etapa 9.

```
{
	admin off
	auto_https off
}

# --- Caminho antigo: CloudFront. Removido na Etapa 9. ---
:80 {
	handle /_edge-health {
		respond "ok" 200
	}
	handle {
		@naoAutorizado not header X-Origin-Verify {env.ORIGIN_SECRET}
		respond @naoAutorizado "forbidden" 403
		reverse_proxy front:3000
	}
}

# --- Caminho novo: Cloudflare. ---
cronos.gabrielmizael.com:443 {
	tls /etc/caddy/certs/origin.pem /etc/caddy/certs/origin.key

	handle /_edge-health {
		respond "ok" 200
	}

	handle {
		@naoAutorizado not header X-Origin-Verify {env.ORIGIN_SECRET}
		respond @naoAutorizado "forbidden" 403

		reverse_proxy front:3000 {
			# A Cloudflare entrega o IP real do visitante em CF-Connecting-IP. Sem esta
			# linha, o X-Forwarded-For que chega na API acumula saltos e o rate limiting
			# por IP passa a contar todo mundo no mesmo balde. Ver §14.2 — isto reduz a
			# cadeia, mas não prova que ela ficou do tamanho que o `trust proxy` espera.
			header_up X-Forwarded-For {http.request.header.CF-Connecting-IP}
		}
	}
}
```

Pontos que importam:

- **`auto_https off` continua ligado.** Agora ele impede o Caddy de tentar ACME para
  `cronos.gabrielmizael.com` — o certificado é fornecido à mão pela diretiva `tls`. Sem isso, o Caddy
  tentaria emitir Let's Encrypt e falharia em loop, porque a porta 80 não está aberta ao mundo.
- **O bloco `:443` é nomeado pelo hostname**, não é `:443` solto. Requisição que chegue com outro
  `Host` não é atendida — e é exatamente esse mecanismo que hospedará o próximo subdomínio.
- **`{env.ORIGIN_SECRET}` é o mesmo segredo da Fase 5.** Quem passa a injetá-lo é uma Transform Rule
  da Cloudflare (§9), não mais o `CustomHeaders` da distribuição.
- **A sonda `/_edge-health` fica fora da checagem de header** nos dois blocos, pelo mesmo motivo de
  antes: ela precisa dizer "o Caddy está vivo", não "a borda me mandou algo válido".

---

## 7. Etapa 4 — nova revisão da task `cronos-edge`

Partindo do JSON da Fase 5 §6, três mudanças:

| Campo | Mudança |
|---|---|
| `portMappings` | acrescentar `{"containerPort": 443, "hostPort": 443, "protocol": "tcp"}` |
| `mountPoints` | acrescentar `/etc/caddy/certs` (`readOnly: true`) |
| `volumes` | acrescentar o `host.sourcePath` correspondente |

```bash
aws ecs register-task-definition --region us-east-2 --cli-input-json file://cronos-edge.json --query "taskDefinition.{Familia:family,Revisao:revision}" --output table
```

**O que faz:** registra a revisão nova. Não afeta nada até o `update-service` abaixo.

```bash
aws ecs update-service --region us-east-2 --cluster ec2-sistema-despesas --service cronos-edge --task-definition cronos-edge --force-new-deployment --query "service.{Servico:serviceName,TaskDef:taskDefinition}" --output text
```

**O que faz:** substitui a task em execução. Como o service já roda com `minimumHealthyPercent=0`
(porta fixa, instância única), o ECS **para a antiga antes de subir a nova** — há uma janela de alguns
segundos em que a borda não atende. É o mesmo comportamento de todo deploy desta stack.

```bash
aws ecs wait services-stable --region us-east-2 --cluster ec2-sistema-despesas --services cronos-edge
```

**O que faz:** bloqueia até o service estabilizar. Se não estabilizar, o disjuntor
(`deploymentCircuitBreaker`) reverte sozinho para a revisão anterior — e o motivo mais provável é
caminho errado de certificado no `Caddyfile`.

---

## 8. Etapa 5 — a prefix list da Cloudflare e a regra 443

A AWS **não** publica uma prefix list gerenciada para a Cloudflare. A lista é sua para criar e manter
— são 15 faixas IPv4, publicadas em <https://www.cloudflare.com/ips-v4>.

```bash
aws ec2 create-managed-prefix-list --region us-east-2 --prefix-list-name cloudflare-ipv4 --address-family IPv4 --max-entries 20 --entries Cidr=173.245.48.0/20 Cidr=103.21.244.0/22 Cidr=103.22.200.0/22 Cidr=103.31.4.0/22 Cidr=141.101.64.0/18 Cidr=108.162.192.0/18 Cidr=190.93.240.0/20 Cidr=188.114.96.0/20 Cidr=197.234.240.0/22 Cidr=198.41.128.0/17 Cidr=162.158.0.0/15 Cidr=104.16.0.0/13 Cidr=104.24.0.0/14 Cidr=172.64.0.0/13 Cidr=131.0.72.0/22 --query "PrefixList.{Id:PrefixListId,Nome:PrefixListName,Max:MaxEntries}" --output table
```

**O que faz:** cria a prefix list com as 15 faixas atuais e `MaxEntries=20`, deixando folga para a
Cloudflare anunciar faixas novas sem exigir recriação. **Anote o `Id`** (`pl-...`).

> **`MaxEntries` conta contra o limite de regras do Security Group** (padrão: 60) — a mesma armadilha
> da Fase 5 §7.1. Aqui são 20, contra as ~55 da prefix list do CloudFront: o SG fica com **mais**
> folga depois da troca, não menos.

```bash
aws ec2 authorize-security-group-ingress --region us-east-2 --group-id <SG_ID> --ip-permissions 'IpProtocol=tcp,FromPort=443,ToPort=443,PrefixListIds=[{PrefixListId=<PL_ID>,Description="Cloudflare proxy"}]'
```

**O que faz:** libera a 443 **exclusivamente** para os endereços da Cloudflare. Como na Fase 5, a
porta nunca é exposta a `0.0.0.0/0`, nem temporariamente: a verificação é feita por dentro da
instância e pelo hostname público.

> **IPv6:** só é necessário se você criar registro `AAAA` apontando para a instância. Não é o caso —
> o registro é `A`, para o Elastic IP.

---

## 9. Etapa 6 — DNS e SSL na Cloudflare

Tudo no painel, na zona `gabrielmizael.com`.

| Onde | O quê | Valor |
|---|---|---|
| **DNS → Records** | `A` | Nome `cronos`, conteúdo `77.112.41.239`, **Proxied (nuvem laranja)**, TTL Auto |
| **SSL/TLS → Overview** | Encryption mode | **Full (strict)** |
| **SSL/TLS → Edge Certificates** | Always Use HTTPS | On |
| | Minimum TLS Version | 1.2 |
| | HSTS | **deixe desligado por enquanto** — ver §14.8 |
| **Rules → Transform Rules → Modify Request Header** | Set static | Header `X-Origin-Verify` = `<o mesmo segredo do SSM>`, com filtro `hostname eq "cronos.gabrielmizael.com"` |

**A nuvem laranja é obrigatória**, não opcional. Cinza (DNS only) significa: o IP de origem fica
público, a 443 do SG não deixa ninguém passar, e nenhuma das proteções existe.

**Nunca `Flexible`.** Nesse modo a Cloudflare fala HTTP com a origem — que aqui está fechada — e o
sintoma clássico é loop de redirecionamento no navegador, sem nada nos logs da aplicação. **`Full`
sem strict** aceitaria certificado inválido na origem, o que anula metade do ponto de usar Origin CA.

> **A Transform Rule é o que amarra a origem a *esta* zona.** A prefix list libera **toda** a
> Cloudflare — qualquer cliente dela poderia apontar um proxy para o seu IP. É o mesmo raciocínio da
> Fase 5 §1, com outro nome no injetor.
>
> **Alternativa/complemento — Authenticated Origin Pulls (mTLS):** gratuito, e o Caddy suporta
> (`client_auth` dentro do bloco `tls`). Mas o certificado de cliente padrão da Cloudflare é
> **compartilhado entre todos os clientes dela** — prova "veio da Cloudflare", não "veio da sua
> zona". Ou seja: tem exatamente a mesma lacuna da prefix list, e por isso **não substitui o header**.
> Vale como camada extra, nunca como troca.

### Universal SSL cobre um nível de subdomínio — e só um

O certificado gratuito da borda cobre `gabrielmizael.com` e `*.gabrielmizael.com`. **Não cobre
`api.cronos.gabrielmizael.com`** nem qualquer outro segundo nível — isso exigiria o Advanced
Certificate Manager, que é pago. Consequência prática de desenho: **mantenha todo hostname a um nível
do apex**. Se o CRONOS um dia expuser a API num hostname próprio, o nome é `api-cronos.gabrielmizael.com`,
não `api.cronos.gabrielmizael.com`.

---

## 10. Etapa 7 — as proteções gratuitas

Ligue **uma de cada vez**, testando o login entre elas. É a única forma de saber qual delas quebrou o
que, e no plano gratuito nenhuma pode ser escopada por regra.

| Proteção | Onde | Observação |
|---|---|---|
| **Free Managed Ruleset** | Security → WAF → Managed rules | Já vem ligado. Não é customizável |
| **Bot Fight Mode** | Security → Bots | ⚠️ **Ligue por último.** No plano gratuito não pode ser pulado por regra nem escopado por rota |
| **Rate limiting (1 regra)** | Security → WAF → Rate limiting rules | Janela e bloqueio de 10 s. Aponte para as rotas de autenticação numa expressão só, já que a cota é de uma regra |
| **Security Level** | Security → Settings | `Medium` (padrão) resolve |
| **Cache** | Caching | **Não crie Cache Rule para HTML.** O padrão da Cloudflare já não cacheia HTML — que é justamente o motivo do `CachingDisabled` no CloudFront (servir página autenticada de um usuário para outro). `/_next/static/*` já é cacheado por extensão |

> **A regra de rate limit da Cloudflare é quebra-molas, não defesa.** Uma janela de 10 segundos não
> para um script paciente. A defesa real continua sendo a da aplicação — 10 cadastros por hora por
> IP, documentada em
> [`../../docs/problema-e2e-rate-limit-cadastro.md`](../../docs/problema-e2e-rate-limit-cadastro.md)
> — e ela depende de a API enxergar o IP certo, o que é exatamente a pendência da §14.2.

---

## 11. Etapa 8 — as variáveis que dependiam do domínio

É aqui que a Fase 7 do `arquitetura-aws.md` §13 se completa. Tudo abaixo estava esperando este
momento, e tudo mora na mesma task definition — faça **uma revisão só**.

| Onde | Variável | Valor |
|---|---|---|
| `cronos-app` (texto plano) | `FRONTEND_URL` | `https://cronos.gabrielmizael.com` |
| | `GOOGLE_CALLBACK_URL` | a URL pública real do callback |
| | `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `MAIL_FROM` | ver `ingresso-aws.md` §11.8 |
| `cronos-app` (`secrets`, SSM) | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `COOKIE_SESSION_SECRET`, `SMTP_PASSWORD` | idem |
| Google Cloud Console | Authorized domains | `gabrielmizael.com` — **o domínio registrável**, que cobre todos os subdomínios |
| | Authorized redirect URIs | a URI **exata**, com subdomínio e path. Sem curinga |
| Google Search Console | Verificação de propriedade | registro `TXT` na zona, com a **mesma conta Google** que é owner do projeto no GCP |

> **Os grupos são "tudo ou nada".** O `env.ts` da API recusa-se a subir com o grupo pela metade, e o
> sintoma é task que nunca fica `healthy`. Ver `ingresso-aws.md` §11.8.
>
> **O rewrite `/api/*` do front é resolvido em build-time** (§11.9) — e trocar o endereço público é
> exatamente o gatilho do bug de 20/08. Reveja
> [`../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md`](../../sistema-controle-despesas-front/docs/problema-rewrite-api-build-time.md)
> **antes** de virar o DNS, não depois.

---

## 12. Verificação

Na sessão SSM, antes de mexer em DNS:

```bash
curl -sk -o /dev/null -w "tls_local=%{http_code}\n" https://localhost/_edge-health
```

**O que faz:** prova que o Caddy está terminando TLS na 443 com o certificado montado. O `-k` é
obrigatório: o Origin CA não é confiado publicamente, e `localhost` não é o hostname do certificado.
Esperado `200`.

Depois de criar o registro DNS, da sua máquina:

```bash
curl -sI https://cronos.gabrielmizael.com | grep -iE "^(HTTP|server|cf-ray)"
```

**O que faz:** o teste de ponta a ponta. Esperado: `HTTP/2 200`, `server: cloudflare` e um `cf-ray` —
os dois últimos provam que o tráfego passou pelo proxy, e não por DNS direto.

```bash
curl -s -o /dev/null -w "%{http_code} -> %{redirect_url}\n" http://cronos.gabrielmizael.com
```

**O que faz:** confirma o `Always Use HTTPS`. Esperado `301` para o `https://`. Os cookies de sessão
são `secure` em produção — sem isso o login não funciona.

```bash
curl -s -o /dev/null -w "direto_no_ip=%{http_code}\n" --max-time 10 https://77.112.41.239 -k
```

**O que faz:** tenta furar a borda indo direto na origem. **Esperado: timeout** (a regra do SG só
aceita faixas da Cloudflare). Uma resposta aqui significa que a 443 ficou aberta a mais do que devia.

```bash
curl -s -o /dev/null -w "login=%{http_code}\n" https://cronos.gabrielmizael.com/login
```

**O que faz:** exercita uma página que passa por `layout.tsx` → `getCurrentUser()` → API. Esperado
`200`; `502` indicaria o front fora, `500` problema de `API_URL`.

### O teste do IP real (§14.2)

Force um `429` numa rota de autenticação e leia o que a API registrou:

```bash
aws logs tail /ecs/cronos-app --region us-east-2 --since 5m --format short | grep -i "ip"
```

**O que faz:** o `logSecurityEvent` da API registra o IP que ela enxerga. Se aparecer o **seu** IP
público, a cadeia de proxies está calibrada. Se aparecer um IP privado (`172.17.x.x`) ou um IP da
Cloudflare, o `trust proxy: 1` está contando errado — e o limitador de cadastro está colocando o
mundo inteiro no mesmo balde.

---

## 13. Etapa 9 — aposentar o CloudFront

**Só depois de a §12 passar inteira**, e de preferência alguns dias depois.

1. Remover o bloco `:80` do `Caddyfile` e registrar nova revisão da `cronos-edge`.
2. Revogar a regra 80 do Security Group (`revoke-security-group-ingress`, mesma prefix list do CloudFront).
3. **Desabilitar** a distribuição `E2EHRXTE92L61B` (`update-distribution` com `Enabled=false`) —
   **não apagar**. Uma distribuição desabilitada custa zero e volta ao ar em minutos.
4. Deletar de verdade só quando o domínio tiver semanas de estrada.

### Rollback

| Momento | Como voltar |
|---|---|
| Antes da Etapa 9 | Usar a URL do CloudFront. Nada a desfazer — os dois caminhos estão vivos |
| Depois da Etapa 9 | Reabrir a 80 no SG, restaurar o bloco `:80` (revisão anterior da task ainda existe), reabilitar a distribuição |
| Problema só de DNS | Trocar a nuvem laranja por cinza **não** resolve: com DNS only o SG bloqueia todo mundo. O caminho é o CloudFront |

---

## 14. Pendências e armadilhas

### 14.1 O `Caddyfile` **e o certificado** não sobrevivem à troca da instância

A dívida da Fase 5 §11.1 **dobrou de tamanho**: agora são três arquivos criados à mão em
`/etc/caddy/`. Se o ASG substituir a máquina, a borda sobe sem configuração **e** sem certificado — e
o sintoma passa a ser erro de TLS, que é ainda menos parecido com "faltou um arquivo".

A correção é a mesma e agora tem dois motivos: `user-data` no launch template escrevendo os três
arquivos no boot, puxando cert e chave do SSM (§5). Isso exige `ssm:GetParameter` para `/cronos/edge/*`
na **instance role** — permissão que hoje ela não tem.

### 14.2 `trust proxy: 1` × uma cadeia de três saltos

O `app.ts` da API declara `app.set('trust proxy', 1)` com o comentário "atrás de um load balancer".
A cadeia real é **CloudFront → Caddy → Next (rewrite `/api/*`) → API** — três saltos, não um. Se o
Next acrescenta o próprio endereço ao `X-Forwarded-For`, `req.ip` na API é o IP do **container do
front**, e o limitador de 10 cadastros/hora/IP vira um limite global de 10 cadastros por hora.

O `header_up` da §6 reescreve o `X-Forwarded-For` com o `CF-Connecting-IP`, o que encurta a cadeia,
mas **não prova** que ela ficou do tamanho que o `trust proxy` espera. Medir com o teste da §12 antes
de considerar resolvido. É uma pendência do repositório da API, não desta fase — mas a troca de borda
é o momento em que ela fica barata de verificar.

### 14.3 Bot Fight Mode não é escopável no plano gratuito

Ele não distingue rota nem pode ser pulado por regra. O fluxo do OAuth é navegação de navegador (deve
passar), e as chamadas SSR acontecem **dentro** da instância (nem passam pela Cloudflare), então o
risco é moderado — mas é real, e é por isso que ele entra por último.

### 14.4 A lista de IPs da Cloudflare é sua para manter

A prefix list do CloudFront era gerenciada pela AWS. Esta não. Se a Cloudflare anunciar uma faixa
nova, parte do tráfego começa a bater em `timeout` **sem nenhum erro do seu lado**. Vale um lembrete
no calendário a cada seis meses, ou automatizar junto com o Terraform da Fase 8.

### 14.5 Limite de 100 MB por requisição no plano gratuito

Irrelevante para o CRONOS hoje. Morde no dia em que existir upload de arquivo.

### 14.6 O Terraform da Fase 8 passa a ter dois providers

`aws` e `cloudflare`, com um token de API da Cloudflare para guardar em algum lugar. Não é um
problema — é escopo que apareceu.

### 14.7 O `origin-secret` continua sendo o mesmo de agosto

Trocar o segredo agora exige mexer em dois lugares (SSM e Transform Rule) em vez de dois (SSM e
distribuição). Mesma fricção, outro painel. Nada a fazer, só registrar.

### 14.8 HSTS é praticamente irreversível

Ligar o HSTS faz o navegador **memorizar** que este host é HTTPS-only, pelo tempo do `max-age`.
Se algo quebrar depois, você não consegue voltar para HTTP nem para um certificado ruim — os
visitantes que já carregaram o header ficam presos. Ligue só quando o domínio estiver estável, e
comece com `max-age` curto.

---

## 15. Referências

- [`ingresso-aws.md`](./ingresso-aws.md) — a Fase 5 que este documento substitui; §1 (a montagem antiga), §4.2 (o padrão de `AccessDenied`), §7 (SG e prefix list), §11 (as pendências herdadas)
- [`arquitetura-aws.md`](./arquitetura-aws.md) — §1.5 (cookies `secure`), §10 (rede e ingresso), §11 (domínio e TLS), §13 (Fase 7)
- [`../../docs/arquitetura-portfolio-subdominios.md`](../../docs/arquitetura-portfolio-subdominios.md) — por que o hostname é `cronos.gabrielmizael.com` e o que vem depois
- [`../../docs/problema-e2e-rate-limit-cadastro.md`](../../docs/problema-e2e-rate-limit-cadastro.md) — o limitador de aplicação que a §14.2 pode estar anulando
- [Cloudflare — Origin CA](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/)
- [Cloudflare — Authenticated Origin Pulls (mTLS)](https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/)
- [Cloudflare — Rate limiting rules (cotas por plano)](https://developers.cloudflare.com/waf/rate-limiting-rules/)
- [Cloudflare — Managed Rules](https://developers.cloudflare.com/waf/managed-rules/)
- [Cloudflare — Universal SSL e a limitação de um nível](https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/)
- [Cloudflare — faixas IPv4](https://www.cloudflare.com/ips-v4)
