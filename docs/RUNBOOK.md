# Runbook — npx-platform

Procedimentos operacionais do dia a dia. Última atualização: 2026-07-26.

## Credencial nativa após provisionamento (obrigatório desde 2026-07-29)

Toda instância nova deve sair com linha em `instance_credentials` (tela
`/credentials`). Se aparecer "Sem credencial" ou o NOC alertar categoria
`credenciais`:

1. NÃO recriar a stack se já houver dados (Chatwoot/BookStack).
2. Resetar/criar admin nativo no container (ver scripts
   `portal/scripts/recover-npx-creds.sh` + `persist-npx-creds.cjs` como
   referência da recuperação NPX).
3. Confirmar login real.
4. Upsert `InstanceCredential` + atualizar `docs/ACCESS.md` na mesma
   sessão.

O provisionamento self-service (`captureNativeCredential`) e o script
de vitrine já fazem isso automaticamente pra instâncias novas.


## Subir/derrubar um stack

Cada stack é independente (Traefik, Portainer, cada cliente):

```bash
cd /opt/npx-platform/traefik      && docker compose up -d   # ou down
cd /opt/npx-platform/portainer    && docker compose up -d
cd /opt/npx-platform/clients/demo && docker compose up -d
```

`docker compose down -v` remove volumes também — só usar se realmente
quiser apagar os dados (bancos, dashboards salvos, etc.).

## Adicionar um novo cliente

1. `mkdir -p /opt/npx-platform/clients/<nome>`
2. Copiar a estrutura de `/opt/npx-platform/clients/demo/docker-compose.yml`
   como referência (rede `internal` própria para banco, containers web na
   rede `edge`).
3. Labels Traefik obrigatórias em cada container que precisa ser roteado:
   ```yaml
   labels:
     - "traefik.enable=true"
     - "traefik.docker.network=edge"
     - "traefik.http.routers.<nome>-<servico>.rule=Host(`<servico>.<nome>.npxit.com.br`)"
     - "traefik.http.routers.<nome>-<servico>.entrypoints=websecure"
     - "traefik.http.routers.<nome>-<servico>.tls=true"
     - "traefik.http.routers.<nome>-<servico>.tls.certresolver=letsencrypt"
     - "traefik.http.services.<nome>-<servico>.loadbalancer.server.port=<porta>"
   ```
4. Criar o DNS do(s) subdomínio(s) novo(s) apontando para o IP público
   correto **antes** de subir (senão o Traefik vai tentar e falhar a
   emissão do certificado repetidamente até funcionar — inofensivo, mas
   gera ruído no log).
5. **Atualizar `docs/ACCESS.md` e `docs/ARCHITECTURE.md` na mesma sessão**
   — não é opcional, ver `CLAUDE.md`.

## Trocar Let's Encrypt de staging para produção

Já foi feito em 2026-07-12 (ver `docs/STATE.md`) — os 4 hosts (`traefik`,
`portainer`, `zabbix.demo`, `grafana.demo`) estão em produção. Passo a passo
para repetir (ex: se algum host novo precisar do mesmo processo, validando
staging antes de virar para produção):

1. Configurar o certresolver com `acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory`.
2. Subir/reiniciar o Traefik, confirmar emissão em staging via
   `openssl s_client -connect 127.0.0.1:443 -servername <host> | openssl x509 -noout -issuer -subject`
   — issuer deve conter `(STAGING)` e subject deve bater com o host.
3. Remover a flag `acme.caserver` do compose (produção é o padrão quando
   ausente).
4. **Zerar o `acme.json`** antes de reiniciar — senão o Traefik reaproveita
   os certificados de staging já válidos em vez de pedir novos à produção.
   O arquivo é `root:root 600` (criado pelo próprio Traefik), então zere via
   um container temporário como root:
   ```bash
   docker run --rm -v /opt/npx-platform/traefik/letsencrypt:/le alpine \
     sh -c "cp /le/acme.json /le/acme.json.staging-backup && : > /le/acme.json"
   ```
5. `docker restart traefik` e confirmar no log:
   `acmeCA=https://acme-v02.api.letsencrypt.org/directory` + `Register...`
   sem erro.
6. Disparar uma requisição para cada host (`curl -sk --resolve <host>:443:127.0.0.1 https://<host>/`)
   para forçar o pedido do certificado (Traefik pede sob demanda).
7. Validar com `curl` **sem `-k`** — se fechar sem erro de certificado, é
   produção real. Confirmar issuer sem `(STAGING)` via `openssl x509`.

## Verificar se um host está roteando corretamente

Sem depender de DNS público (útil para testar antes do DNS existir, ou a
partir da própria VM):

```bash
curl -sk -o /dev/null -w "%{http_code}\n" --resolve <host>:443:127.0.0.1 https://<host>/
```

Verificar qual certificado está sendo servido (self-signed de fallback vs
Let's Encrypt real):

```bash
echo | openssl s_client -connect 127.0.0.1:443 -servername <host> 2>/dev/null | openssl x509 -noout -issuer -subject
```

Se `issuer=CN = TRAEFIK DEFAULT CERT`, o Let's Encrypt ainda não emitiu para
aquele host (checar `docker logs traefik | grep -i acme`).

## Diagnosticar problema de emissão Let's Encrypt

```bash
docker logs traefik --tail 100 | grep -iE "acme|certificate|challenge"
```

Erros comuns:
- `DNS problem: NXDOMAIN` → o host não tem registro DNS público ainda.
- `Timeout during connect` / `Connection refused` → porta 80 não está
  alcançável de fora (checar FortiGate/firewall), Let's Encrypt precisa
  conectar na porta 80 (HTTP-01) para validar.
- Rate limit (`too many certificates`/`too many failed authorizations`) →
  esperar a janela do rate limit da Let's Encrypt passar; por isso staging
  é usado primeiro para não gastar tentativas de produção com configuração
  ainda não validada.

## Onde achar o slug (identificador técnico) de um tenant

Desde 2026-07-19, o slug é gerado automaticamente na criação do tenant
(normalizado a partir do nome) — nunca mais aparece pra preencher na
tela de criação (ver `docs/DECISIONS.md`). Continua **visível** em dois
lugares, pra qualquer humano da NPX (ou a IA por-tenant do roadmap)
conseguir localizar quando precisar fazer algo manual/técnico:

1. **Tela de tenants** (`/dashboard`, para super_admin) — coluna "Slug"
   na listagem.
2. **Tela de detalhe/edição do tenant** (`/tenants/<id>`) — mostrado logo
   abaixo do título, com o texto "Identificador técnico interno (slug)".

O slug é o que vira nome de container Docker (`<slug>-zabbix-server`,
etc.), label de subdomínio DNS (`<tipo>.<slug>.npxit.com.br`), e prefixo
de nome de objeto no FortiGate (`zabbix_<slug>_<porta>`) — é o dado que
qualquer procedimento manual (este runbook, `docker exec`, SSH no
FortiGate) precisa pra identificar qual recurso pertence a qual tenant.

## Adicionar/trocar uma senha de algum serviço

1. Gerar senha forte: `openssl rand -base64 18 | tr -d '/+=' | cut -c1-20`
2. Aplicar no serviço (variável de ambiente do compose, ou via API/UI do
   serviço, dependendo do caso).
3. **Atualizar `docs/ACCESS.md` imediatamente**, na mesma sessão — regra
   permanente, ver `CLAUDE.md`.

## SEMPRE atualizar a permissão do `grafana-reader` ao criar um host group novo no Zabbix

**Achado real, 2026-07-17 (onboarding MIP ENGENHARIA), que custou tempo real
de depuração — não pular este passo de novo.** O usuário de API dedicado
que o Grafana usa para consultar cada Zabbix de cliente (`grafana-reader`,
grupo Zabbix `API read-only (Grafana)`) **não tem permissão "todos os host
groups"** — tem uma lista explícita de `hostgroup_rights` (IDs de grupo
específicos). Um host group novo (ou um host movido para um group novo)
**fica invisível para o Grafana até esse group ser adicionado
explicitamente a essa lista**, mesmo que o group já exista e already tenha
hosts com dados reais sendo coletados no Zabbix.

**Sintoma:** painel do Grafana (tipo "Problems" do plugin Zabbix,
`queryType: "4"`) mostra **"No data"** para qualquer filtro de
group/host que não seja o wildcard puro `/.*/` — não é erro visível, não
aparece no `/api/ds/query` (que aliás retorna `"non-metrics queries are
not supported"` para esse tipo de query ao testar direto, então não serve
pra depurar isso — o teste real precisa ser visual, via screenshot do
dashboard renderizado). Fácil de confundir com "o painel está mal
configurado" quando na verdade é permissão no lado do Zabbix.

**Correção:**

```
usergroup.update
  usrgrpid: <id do grupo "API read-only (Grafana)" daquele Zabbix>
  hostgroup_rights: [...lista atual..., {"id": "<novo groupid>", "permission": "2"}]
```

(`permission: "2"` = read-only, mesmo padrão de todas as entradas
existentes). Repetir para cada host group novo criado — inclusive quando
só *move* um host existente para um group que ainda não está nessa lista
(caso real: mover SW20/23/25 de "Applications" — que tinha permissão —
para "MIP ENGENHARIA/BH-MG/Switches" — que não tinha — quebrou a
visibilidade desses 3 hosts no Grafana até a permissão ser corrigida).

**Regra permanente daqui pra frente:** todo host group novo criado em
qualquer Zabbix de cliente (convenção `<Cliente>/<Cidade-UF>/<Categoria>`
acima) deve ter sua permissão adicionada ao `grafana-reader` **na mesma
sessão**, antes de considerar o onboarding concluído — mesmo espírito da
regra de documentação imediata do `CLAUDE.md`.

## Convenção de agrupamento de hosts no Zabbix (por cliente)

Padrão adotado a partir de 2026-07-17 (caso real: MIP ENGENHARIA, unidade
do tenant FLUA TI) para **todo host novo, de qualquer cliente, daqui pra
frente**. Usa o suporte nativo do Zabbix a host groups aninhados (`/` no
nome cria hierarquia visível na árvore de grupos, sem precisar que o grupo
pai exista como objeto separado):

```
<Cliente>/<Cidade-UF>/<Categoria>
```

Exemplos reais:

- `MIP ENGENHARIA/BH-MG/Switches`
- `MIP ENGENHARIA/BH-MG/Impressoras`
- `MIP ENGENHARIA/BH-MG/Servidores VMware`

Regras:

1. `<Cliente>` é o nome do cliente final (pode ser diferente do nome do
   tenant no portal, quando o tenant representa uma revenda/matriz com
   várias unidades/clientes finais — caso da FLUA TI, que hospeda a MIP
   ENGENHARIA como um dos clientes monitorados).
2. `<Cidade-UF>` identifica o site físico — relevante quando um cliente
   tem mais de uma unidade/filial monitorada pelo mesmo Zabbix.
3. `<Categoria>` é o tipo de ativo (`Switches`, `Impressoras`,
   `Servidores VMware`, `Firewalls`, etc. — usar o nome mais específico
   que fizer sentido pro tipo de equipamento, não uma categoria genérica
   tipo "Network" ou "Applications").
4. Mover um host de grupo **nunca** deve alterar interface SNMP, template
   ou macros — é reorganização pura de agrupamento/visualização.
5. Ao criar os grupos via API, criar só os grupos-folha com o caminho
   completo (`hostgroup.create` com `name` já contendo os `/`) — o Zabbix
   monta a árvore visual sozinho a partir dos segmentos do nome, não é
   necessário criar `MIP ENGENHARIA` e `MIP ENGENHARIA/BH-MG` como grupos
   independentes.

## Containers e o que cada um faz (referência rápida)

| Container | Stack | Função |
|---|---|---|
| `traefik` | traefik | Reverse proxy / TLS |
| `docker-shim` | traefik | Proxy de rewrite do socket Docker (ver DECISIONS.md) |
| `portainer` | portainer | UI de administração Docker |
| `demo-zabbix-server` | clients/demo | Zabbix server |
| `demo-zabbix-web` | clients/demo | Zabbix web (nginx+php) |
| `demo-mysql` | clients/demo | Banco do Zabbix |
| `demo-grafana` | clients/demo | Grafana |

## Host novo num Zabbix com proxy group pode ficar sem proxy atribuído por alguns minutos

Achado real em 2026-07-18: ao criar um host novo com `monitored_by: 2`
(proxy group) via API, o campo `assigned_proxyid` pode ficar `"0"` por
alguns minutos até o grupo terminar de rebalancear — nesse estado,
nenhum `task.create` (check-now) executa, e o host parece "não
responder" mesmo que o equipamento esteja perfeitamente online. Não é
falha de conectividade do equipamento, é atraso interno do Zabbix.

**Como diferenciar de um equipamento realmente não respondendo:**

```
host.get { output: [host, assigned_proxyid], hostids: [...] }
```

Se `assigned_proxyid` for `"0"`, espere (na prática, minutos, não
segundos — recriar o host do zero costuma resolver mais rápido que
esperar o rebalanceamento automático) até virar um id real antes de
concluir que o SNMP/ICMP não respondeu. Só depois de `assigned_proxyid`
ter um valor real é que um `task.create` sem resposta significa de fato
"equipamento não respondeu".

Sintoma associado no log do `zabbix-server` (`docker logs
<container>-zabbix-server`): `Proxy group "<nome>" changed state from
online to degrading` seguido de volta a `online` alguns segundos depois
— coincide com a janela em que hosts novos ficam sem proxy atribuído.

## Volume `portal-uploads-data` (upload de branding) — passo único depois de criar/recriar

O container `portal` roda como usuário não-root (`user: "1000:1000"`
no `docker-compose.yml`). Um volume Docker nomeado NOVO é sempre
criado com o ponto de montagem `root:root` — o container não consegue
escrever nele até alguém ajustar a permissão uma vez, de fora. Achado
real (Fase 2 da sessão de validação profunda, 2026-07-26): upload de
logo/favicon (`/tenants/[id]/branding`) falhava com `EACCES: permission
denied, mkdir '/app/public/uploads/tenants'` até isso ser corrigido.

**Só precisa rodar isto uma vez, e só se o volume `portal_portal-uploads-data`
for recriado do zero** (primeira subida, ou depois de `docker compose down -v`,
ou migração pra host novo):

```bash
docker run --rm -v portal_portal-uploads-data:/data alpine chown -R 1000:1000 /data
```

Se o upload de branding voltar a dar `EACCES`, é sinal de que isso
precisa ser rodado de novo.


## Onboarding MIP automático quando o proxy voltar (FASE D3)

1. Não tente religar `FLUA-Proxy-01` desta VM — é infra do cliente FLUA.
2. O cron do `suporteti` roda a cada 5 min:
   `python3 /opt/npx-platform/scripts/mip-proxy-watcher.py`
3. Estado: `cat /opt/npx-platform/var/mip-onboard/mip-onboard-watcher.json`
   - `waiting_proxy` — ainda offline
   - `running` / `failed_will_retry` — tentando
   - `completed` — onboarding SNMP já rodou; não reexecuta (use
     `--force` no watcher pra forçar)
4. Manual: `python3 .../mip-onboard-ativos.py --check-proxy` e
   `--apply` quando fresco.
5. Hosts novos usam **sempre** o proxy group `FLUA-Proxy-Group`, nunca
   o proxy individual (FASE D2).

## Assistente de IA por tenant (FASE G)

- Config chave/modelo: `/settings/ai` (só ADMN)
- Chat: `/tenants/<id>/ai` (quem tem acesso ao tenant + ver instâncias)
- Teste de isolamento: `python3 scripts/test-ai-tenant-isolation.py`


## Volume de anexos da IA (`portal-ai-uploads`)

Portal roda como uid 1000. Depois de criar o volume pela primeira vez:

```bash
docker run --rm -v portal_portal-ai-uploads:/data alpine chown -R 1000:1000 /data
```

Sem isso, upload toma `EACCES` em `/app/data/ai-uploads`.

## Chatwoot (tipo de catálogo, 2026-07-28)

Stack: `chatwoot/chatwoot:v3.16.0` + `pgvector/pgvector:pg16` + Redis 7.
Provisionamento self-service cria SuperAdmin via `AccountBuilder`
(`rails runner`). Login: `suporteti@npxit.com.br` + senha compartilhada
(`docs/ACCESS.md`). Se o volume de storage for recriado, não há passo
`chown` especial (app roda como root na imagem oficial). Atualizar pin
de versão: `compose-templates.ts` (`chatwoot` + `chatwoot-sidekiq`).

## Sessão 41 (2026-07-30) — operações novas

### Docker Socket Proxy (kopia)
```
cd /opt/npx-platform/backup/docker-socket-proxy && docker compose up -d
cd /opt/npx-platform/backup/kopia && docker compose up -d
# Agent deve ter DOCKER_HOST=tcp://npx-docker-socket-proxy:2375 e NÃO montar docker.sock
```

### Redis do portal (rate limit)
```
cd /opt/npx-platform/portal/redis && docker compose up -d
# portal precisa REDIS_URL=redis://portal-redis:6379 e estar na mesma network
```

### Isolamento lateral / rede tenant
Após provisionar (ou migrar) um tenant fora da `edge`:
```
docker network connect {slug}_internal traefik
docker network connect {slug}_internal portal
```
(automatizado em `ensurePlatformOnTenantNetwork` para fluxos novos).

### Rebuild / recreate do portal (sessão 42)
```
cd /opt/npx-platform/portal
docker compose build portal && docker compose up -d portal
# compose só anexa edge + portal_internal — religar internas:
for n in backup_internal demo_internal felixti_internal flua_internal \
  npx-zabbix_internal npx_internal valid1_internal validnivel2_internal; do
  docker network connect "$n" portal 2>/dev/null || true
done
```
PDFs comerciais: bind `portal/public/generated-pdfs` → `/app/public/generated-pdfs`;
servidos por `GET /generated-pdfs/[file]` (não depender só de static do Next).

### Destinos nuvem
Ver `docs/BACKUP-CLOUD-DESTINATIONS.md` (nativo Kopia vs rclone).
