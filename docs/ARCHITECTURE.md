# Arquitetura — npx-platform

Última atualização: 2026-07-12 (GLPI da FLUA TI exposto publicamente).

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
- Diretório reservado, ainda vazio nesta sessão — não usado até agora.

## Convenções

- Todo serviço que precisa ser roteado publicamente entra na rede externa
  `edge` e ganha labels `traefik.enable=true` +
  `traefik.docker.network=edge` (necessário sempre que o container também
  está em outra rede, para desambiguar).
- Bancos de dados ficam em rede `internal` própria do stack do cliente,
  nunca na rede `edge`.
- TLS: sempre via `tls.certresolver=letsencrypt` para hosts WAN reais; o
  `traefik.local` interno continua em `tls=true` simples (certificado
  default self-signed do Traefik), como acesso de fallback para LAN.
