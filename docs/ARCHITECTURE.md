# Arquitetura — npx-platform

Última atualização: **2026-07-30 (sessão 42 — PDF/branding/audit;
sessão 41 — isolamento `*_internal` + socket-proxy + portal-redis)**.
O diagrama ASCII abaixo ainda mostra o modelo antigo “tudo na edge”; o
**comportamento atual** está nas seções “Convenções (2026-07-30)” e em
`docs/portal/ARCHITECTURE.md` / `docs/STATE-SESSAO-42.md`.

## Visão geral

Um único host Docker (VM `suporteti`) roda um reverse proxy (Traefik) na
frente de todos os serviços. O FortiGate da rede encaminha as portas 80/443
do IP público para este host; o DNS de `*.npxit.com.br` é gerido via
Microsoft 365 (nameservers `ns[1-4].bdm.microsoftonline.com`).

```
Internet
   │  (FortiGate: 80/443 -> este host)
   ▼
┌─────────────────────────────────────────────────────────┐
│ Host Docker (suporteti)                                  │
│                                                            │
│  rede "edge" (externa, compartilhada por todos os serviços│
│  que precisam ser roteados publicamente)                  │
│                                                            │
│  ┌───────────┐     ┌──────────────┐                       │
│  │docker-shim│◄────┤   Traefik    │◄── :80 / :443 (host)  │
│  │(nginx,    │ unix│ v3.5         │                        │
│  │ sem rede) │sock │ - ACME (LE)  │                        │
│  └─────┬─────┘     │ - dashboard  │                        │
│        │            └──────┬───────┘                       │
│   /var/run/docker.sock      │ roteia por Host()             │
│   (real, ro)                 │                                │
│                               ▼                                │
│        ┌──────────────────────────────────────────┐            │
│        │ rede edge                                  │           │
│        │  portainer          demo-zabbix-web  demo-grafana      │
│        │  (docker.sock rw)   │                 │                │
│        │                     │                 │                │
│        │              rede "internal" (stack demo, isolada)     │
│        │              demo-zabbix-server ── demo-mysql          │
│        └──────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘
```

## Componentes

### Traefik (`/opt/npx-platform/traefik/`)
- Reverse proxy único de entrada. Entrypoints `web` (80) e `websecure` (443).
- Provider Docker: `exposedByDefault=false` — só containers com
  `traefik.enable=true` são roteados. Aponta para o `docker-shim` via socket
  Unix, não para o socket real.
- Certresolver `letsencrypt` (HTTP-01, entrypoint `web`), **em produção**
  desde 2026-07-12 — ver `docs/STATE.md`.
- Dashboard protegido por basicauth, acessível em `traefik.local` (LAN,
  self-signed) e `traefik.npxit.com.br` (WAN, Let's Encrypt).
- `docker-shim`: nginx que reescreve requisições ao Docker API para
  contornar um bug de negociação de versão entre o client do Traefik e o
  dockerd deste host. Ver `docs/DECISIONS.md`.

### Portainer (`/opt/npx-platform/portainer/`)
- UI de administração do Docker do host. Roteado via Traefik em
  `portainer.npxit.com.br`.
- Fala direto com `/var/run/docker.sock` (rw) — não usa o shim, pois seu
  client negocia a versão da API corretamente.

### Clientes (`/opt/npx-platform/clients/<nome>/`)
- Cada cliente é um stack Compose isolado, com sua própria rede `internal`
  para banco de dados (não exposta) e containers web na rede `edge` (para
  serem roteados pelo Traefik).
- Cliente **`demo`** (zabbix.demo.npxit.com.br, grafana.demo.npxit.com.br) —
  stack de demonstração/validação, não é cliente real de produção.
- Cliente **`flua`** (FLUA TI) — primeiro cliente real. Zabbix + Grafana +
  GLPI todos roteados publicamente via Traefik (GLPI ganhou rota pública em
  2026-07-12; antes disso ficava só na rede `internal` + porta
  `127.0.0.1:8082`, que foi removida ao expor). Grafana está em `edge`
  (rota pública) **e** em `internal`
  (fala direto com `flua-zabbix-web` via rede Docker, sem precisar sair para
  a internet e voltar pelo Traefik) — é a única exceção deliberada à regra
  "só web/frontend toca edge, DB fica em internal": aqui `internal` serve
  também para tráfego backend-a-backend dentro do mesmo cliente.
- Integração Zabbix→GLPI: webhook customizado (media type "GLPI (custom
  webhook)") rodando dentro do processo `zabbix-server`, chamando a API
  REST do GLPI via rede `internal` (`http://flua-glpi/apirest.php`). Ver
  `docs/DECISIONS.md` para por que não foi usado o webhook oficial da
  Zabbix.

### `portal/` (`/opt/npx-platform/portal/`)
- Já não é um diretório vazio — é o portal de gestão multi-tenant
  (Next.js + Prisma + Postgres próprio, containers `portal`/`portal-db`),
  hoje o maior componente do sistema: autenticação, hierarquia de
  tenants (raiz ADMN), provisionamento self-service de instâncias,
  permissões granulares, credenciais cifradas e o protótipo de IA.
  Roteado via Traefik em `admn.npxit.com.br`, na rede `edge` (mesma
  convenção dos demais serviços). Arquitetura completa e atual em
  `docs/portal/ARCHITECTURE.md` — não duplicada aqui de propósito, para
  evitar drift. Pontos novos (2026-07-28): chat de IA por tenant em
  `/tenants/[id]/ai` (isolamento lógico; chave em `/settings/ai`);
  watcher MIP em `scripts/mip-proxy-watcher.py` + estado em
  `var/mip-onboard/` (dispara onboarding quando `FLUA-Proxy-01` volta).

### `scripts/` (automação operacional)
- `mip-onboard-ativos.py` — hosts Zabbix MIP sempre no proxy group
  `FLUA-Proxy-Group` (nunca proxy individual).
- `mip-proxy-watcher.py` — cron `*/5` do usuário `suporteti`.
- `test-ai-tenant-isolation.py` — prova de isolamento lógico da IA.

### Backup granular por instância (`/opt/npx-platform/backup/`) — Fase 1, 2026-07-27; endurecido 2026-07-30
- Motor **Kopia**, complementar ao Acronis (que já cobre a VM inteira via
  backup diário completo — nunca mexido por este subsistema). Componentes
  em `backup_internal` (+ `portal_internal` onde o portal precisa falar):
  - `npx-kopia-server` — Repository Server; storage local por default;
    destinos S3/B2/Azure/GCS/SFTP/WebDAV nativos; OneDrive/GDrive via
    rclone (`docs/BACKUP-CLOUD-DESTINATIONS.md`).
  - `npx-docker-socket-proxy` (Tecnativa) — única face Docker do agente;
    endpoints mínimos; sem mount de `docker.sock` no agent.
  - `npx-kopia-agent` — API HTTP :8090; dump lógico + snapshot/restore;
    `DOCKER_HOST=tcp://npx-docker-socket-proxy:2375`.
- Granularidade de identidade: **um "usuário" Kopia por TENANT** (não
  por instância) — ver `prisma/schema.prisma::TenantBackupConfig`.
- Detalhe completo em `docs/portal/ARCHITECTURE.md`.

### Portal Redis (`portal/redis/`) — 2026-07-30
- Redis dedicado da plataforma para rate limiting (`rate-limiter-flexible`),
  **isolado** dos Redis dos Chatwoots de tenant.

## Convenções (2026-07-30 — isolamento lateral)

- **Apps de tenant** (web + DB) ficam só em `{slug}_internal`.
- **Traefik** e **portal** são conectados a cada `{slug}_internal`
  (`ensurePlatformOnTenantNetwork` no provisionamento).
- Labels: `traefik.enable=true` + `traefik.docker.network={slug}_internal`.
- A rede `edge` permanece para Traefik entrypoint / serviços de plataforma
  que ainda a usam; **não** é mais o caminho app↔app entre tenants
  (evidência sessão 41: lateral FLUA↔Felix bloqueado após migração).
- Bancos nunca na `edge`.
- TLS: sempre via `tls.certresolver=letsencrypt` para hosts WAN reais; o
  `traefik.local` interno continua em `tls=true` simples (certificado
  default self-signed do Traefik), como acesso de fallback para LAN.
- Sessão 42: stacks `demo` / `npx` / `validnivel2` / `monitoring/npx-zabbix`
  migradas para `*_internal` (inventário: só traefik/portal/portainer/
  npx-zabbix-agent na `edge`).

## Comercial PDF + auditoria (2026-07-30 — sessão 42)

- Lib `portal/src/lib/commercial-pdf.ts` (pdfkit) gera orçamento e
  relatório executivo em `/app/public/generated-pdfs` (bind mount no host).
- Download via App Router `GET /generated-pdfs/[file]` (standalone não
  indexa arquivos criados após o boot).
- Tabela SQL `commercial_audit` (ticket, time entry, sale item, quote,
  PDF export, `instance.start|stop|restart`).
- Branding público: `lib/auth-branding.ts` no login (`?tenant=`) e no
  e-mail de reset.

## NOC interno + vitrine NPX (2026-07-28)

- **NOC ADMN**: rota portal `/noc` + coletor `lib/noc/collect.ts` (containers
  Portainer, VIP TCP, DNS/TLS, Kopia, integrações, centrais, FortiGate NPX).
- **Uptime Kuma NPX**: stack em `clients/npx/` (`npx-uptime-kuma`), status
  page `/status/entrega` (checagem externa HTTP das URLs públicas).
- **Vitrine**: `npx-bookstack` (KB), `npx-chatwoot` (inbox site + AgentBot),
  agente `lib/vitrine/agent.ts` → webhook `/api/vitrine/chatwoot-hook` →
  GLPI (`npx-glpi`) para ação de conta. Marca: env `VITRINE_BRAND_NAME`.
