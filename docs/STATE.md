# Estado atual — npx-platform

Última atualização: 2026-07-12 (sessão longa: brand-kit, publicação docs,
branding, monitoramento da própria NPX, NOC, templates — ver seção
"Sessão de branding/publicação/observabilidade" mais abaixo para o
progresso fase a fase).

## Resumo do que está no ar

| Serviço | Container(s) | URL | TLS |
|---|---|---|---|
| Traefik | `traefik`, `docker-shim` | traefik.local (LAN) / **traefik.npxit.com.br** (WAN) | self-signed (LAN) / **Let's Encrypt produção** (WAN) |
| Portainer | `portainer` | **portainer.npxit.com.br** | **Let's Encrypt produção** |
| Portal multi-tenant | `portal`, `portal-db` | **admn.npxit.com.br** | **Let's Encrypt produção** |
| Zabbix (demo) | `demo-zabbix-server`, `demo-zabbix-web`, `demo-mysql` | **zabbix.demo.npxit.com.br** | **Let's Encrypt produção** |
| Grafana (demo) | `demo-grafana` | **grafana.demo.npxit.com.br** | **Let's Encrypt produção** |
| Zabbix (FLUA TI) | `flua-zabbix-server`, `flua-zabbix-web`, `flua-mysql` | **zabbix.flua.npxit.com.br** | **Let's Encrypt produção** |
| Grafana (FLUA TI) | `flua-grafana` | **grafana.flua.npxit.com.br** | **Let's Encrypt produção** |
| GLPI (FLUA TI) | `flua-glpi`, `flua-glpi-mysql` | **glpi.flua.npxit.com.br** | **Let's Encrypt produção** |

## Sessão de branding/publicação/observabilidade (2026-07-12) — progresso fase a fase

Sessão longa com 7 fases (0 a 6). Cada fase é atualizada aqui **ao final
dela**, não só no final da sessão inteira — se a sessão for interrompida,
esta seção diz exatamente onde parou.

**Fase 0 — brand-kit NPX IT — concluída.**
Manual de identidade visual (PDF, 16 páginas) e papel timbrado (docx)
lidos por completo (texto extraído + páginas renderizadas como imagem para
capturar cores/tipografia exatas — sem `pdftotext`/`poppler` disponível no
host, processado via container Python com `pymupdf`/`python-docx`).
Brand-kit estruturado em `/opt/npx-platform/portal/brand/npxit.json`
(cores com origem CMYK/Pantone documentada, tipografia com fallback web,
regras de uso incorreto, dados de contato oficiais confirmados no
timbrado). Logos copiados para
`/opt/npx-platform/portal/public/brand/npxit/` (estrutura definitiva,
6 arquivos: light/dark/full-light/full-dark/icon/favicon — a pasta
`FILES/` original não foi apagada, mas nada deve depender dela daqui pra
frente). Documentado em `docs/portal/BRANDING.md`.

**Fase 1 — publicação da documentação — concluída.**
`/opt/npx-platform` agora é um repositório git próprio (remote `origin` →
`admn`, privado). `/opt/npx-platform/docs-publish/` é um repositório git
separado e independente (remote → `platform-docs`, público), listado no
`.gitignore` do repo raiz para os dois nunca se misturarem.
`scripts/publish-docs.sh` sincroniza só a lista fechada de docs
sanitizados + roda checagem heurística de segredo antes de cada push;
`scripts/backup-source.sh` faz backup completo (com segredos reais,
decisão consciente do usuário — ver `docs/DECISIONS.md`) para o `admn`.
Ambos os scripts rodados com sucesso nesta sessão. Log de sessão publicado
em `docs-publish/logs/2026-07-12-sessao-branding-publicacao.md`.

**Nota operacional para sessões futuras:** o `admn` já tinha 3 commits de
teste de uma sessão anterior (verificação de acesso git) quando este novo
histórico foi criado — foi feito merge com `--allow-unrelated-histories`
em vez de force-push, então nada foi perdido, mas o histórico do `admn`
tem uma raiz "dupla" por causa disso. Não é um problema funcional, só uma
curiosidade do histórico se alguém for investigar depois.

**Fase 2 — branding por tenant — concluída (com limites honestos).**
Matriz real de capacidades investigada e **testada ao vivo** contra o
stack `flua`: GLPI suporta logo+cor nativamente (`custom_css_code` da
Entity — confirmado funcionando), Zabbix e Grafana OSS **não** suportam
logo/cor (Zabbix não tem essa capacidade nativa; Grafana é Enterprise-only
— confirmado, sem a seção `[white_labeling]` no `defaults.ini`). Os três
suportam favicon via volume mount de arquivo estático (achado bom: o do
Grafana não precisa de rebuild de imagem, ao contrário do que se
imaginava). Tema claro/escuro nativo e testado em Zabbix e Grafana.
Implementado em `portal/src/lib/branding.ts` +
tela `/tenants/<id>/branding`. Estrutura de pasta
`clients/<tenant>/branding/` criada (referência em `clients/flua/`) com
validação de arquivo em `portal/src/lib/branding-upload.ts` — a tela de
upload em si fica para quando o provisionamento self-service existir (ver
`docs/ROADMAP.md`). Detalhes completos em `docs/portal/BRANDING.md`.

**Limite não-bloqueante:** a tela de aplicar branding foi validada por
compilação + carregamento (200), não por teste de ponta a ponta via
navegador nesta sessão (o form usa Server Action client-side, não é
trivial reproduzir via curl puro). Recomendo confirmar visualmente antes
de anunciar a clientes.

**Fase 3 — Zabbix da própria NPX — concluída (falta só o DNS/cert real).**
Stack dedicada em `/opt/npx-platform/monitoring/npx-zabbix/` (containers
`npx-mysql`, `npx-zabbix-server`, `npx-zabbix-web`, `npx-zabbix-agent`) —
isolada, não é tenant de cliente nenhum. Senha padrão trocada.

Host `Docker-Host-suporteti` criado com os templates oficiais **"Linux by
Zabbix agent"** e **"Docker by Zabbix agent 2"**, mais 3 itens/triggers
customizados (TCP: Traefik 443, Portainer 9000, Postgres do portal 5432).

**Validado com dados reais coletados (não só deploy):**
- CPU/RAM/disco do **host de verdade** (não do container): 2.05% CPU,
  33.9% de 15.6GB RAM, disco real via bind-mount `/hostfs` (263GB total,
  7.9% usado) — confirmado que reflete o host, não o container isolado.
- **19 containers descobertos automaticamente** via LLD do template
  Docker (demo-*, flua-*, npx-*, portainer, portal, portal-db, traefik,
  docker-shim), todos com `Status: running`.
- Traefik, Portainer e Postgres do portal: os 3 checks TCP retornaram "1"
  (up).

**Acesso sensível autorizado explicitamente pelo usuário** (mesmo padrão
do `docker-shim`): o container `npx-zabbix-agent` monta
`/var/run/docker.sock:ro` (visibilidade de containers) e `/:/hostfs:ro`
(disco real do host) — sem porta exposta, sem escrita. Ver
`docs/DECISIONS.md`.

**Pendente (não bloqueante):** `zabbix-master.npxit.com.br` ainda não tem
DNS criado — Traefik está servindo com certificado self-signed de
fallback (`TRAEFIK DEFAULT CERT`), confirmado via `openssl s_client`.
Assim que o DNS existir (mesmo processo já usado para os outros hosts:
`dig @8.8.8.8` para confirmar, depois só esperar o Traefik emitir),
o certificado real sai sozinho, nenhuma ação adicional necessária.

**Fase 4 — Grafana NOC por tenant + mestre NPX — pendente.**

**Fase 5 — biblioteca de templates v1 — pendente.**

**Fase 6 — fechamento — pendente.**

## Portal de gestão multi-tenant — Fase 1 (fundação) — concluída

Movido de `docs/ROADMAP.md` ("Portal — fundação (auth + modelo de
tenants)") para cá, 2026-07-12. Detalhes técnicos completos em
`docs/portal/ARCHITECTURE.md`.

- Next.js 14 (App Router) + TypeScript + Prisma + Postgres próprio
  (`portal-db`, rede `internal` isolada, não reaproveita banco de cliente).
- Modelo de dados: `tenants` (hierarquia via `parent_tenant_id`), `users`
  (papel `super_admin`/`gestor`/`tecnico`), `instances` (referência a
  onde a instância mora e onde estão as credenciais — nunca a senha em
  si).
- Seed rodado: tenant raiz **NPX IT**, tenant filho **FLUA TI**, as 5
  instâncias já existentes (zabbix/grafana/glpi da FLUA + zabbix/grafana
  do demo) apontando para o tenant certo, e o primeiro usuário
  `super_admin`.
- Autenticação (login, JWT em cookie httpOnly, bcrypt) e autorização
  (isolamento entre tenants) **testadas via requisições HTTP reais**
  nesta sessão, não só lidas no código — ver `docs/portal/ARCHITECTURE.md`
  para o que foi validado.
- No ar em **https://admn.npxit.com.br**, certificado Let's Encrypt
  produção válido até 2026-10-10.

**Pendência que não é bloqueante:** SMTP do Office 365 (fluxo "esqueci
minha senha") está com `SMTP_USER`/`SMTP_PASSWORD` vazios em
`/opt/npx-platform/portal/.env` — aguardando o usuário fornecer a senha de
aplicativo. Até lá, o token de reset é gerado normalmente, só o e-mail não
sai.

Certificados emitidos em 2026-07-12, válidos até **2026-10-10** (90 dias,
padrão Let's Encrypt — Traefik renova automaticamente antes de expirar,
nenhuma ação manual necessária em condições normais).

## Cliente FLUA TI — status por fase

**Fase 1 (stack + certificados + senha) — fechada.**
Zabbix + Grafana + GLPI no ar, nomeados `flua-*`, rede `internal` própria
isolada (só web/frontend tocam `edge`). Certificados reais emitidos para
`zabbix.flua` e `grafana.flua`. Senha padrão do Zabbix (`Admin`/`zabbix`) já
trocada — ver `docs/ACCESS.md`.

**Fase 2 (Zabbix↔Grafana) — fechada, com uma ressalva.**
Plugin oficial `alexanderzobnin-zabbix-app` (v6.4.1) instalado e habilitado.
Datasource "Zabbix" criado e testado: health check retornou
`Zabbix API version 7.0.28`. Usuário de API dedicado `grafana-reader`
(role "User role", grupo com permissão só-leitura em todos os host groups) —
testado: lê hosts, não consegue criar usuários. Dashboard inicial
"FLUA - NOC Overview" criado (`/d/flua-noc-overview`), com painel de
problemas ativos (contagem + tabela) e hosts monitorados.

**Ressalva:** o tipo de query "Problems"/"Number of problems" desse plugin
só funciona através do frontend do Grafana (JS/React), não através da API
genérica `/api/ds/query` — confirmei isso tentando validar o painel via
curl e recebendo `"non-metrics queries are not supported"` mesmo com dados
reais de problema presentes no Zabbix. Isso é uma característica
arquitetural do plugin (metrics vs non-metrics usam caminhos internos
diferentes), não um defeito do dashboard. **Ação recomendada:** confirmar
visualmente no navegador (login em grafana.flua.npxit.com.br) que o painel
"Problemas Ativos" renderiza corretamente antes de considerar 100%
validado.

**Fase 3 (Zabbix↔GLPI) — fechada, com um desvio do padrão oficial.**
O webhook oficial da Zabbix para GLPI (`glpi_legacy_api=true`) exige um
token pessoal (`glpi_user_token`) que o GLPI **não expõe via API** (só
aparece na tela "Remote access keys" da própria UI, e mesmo tentando via
`_reset_api_token` na API o valor plaintext nunca é retornado). Como não há
acesso a browser neste ambiente, escrevi um **webhook customizado** (JS)
que faz a mesma coisa usando Basic Auth (usuário/senha do
`zabbix-integration`) contra `/apirest.php/initSession` — mais simples e
sem essa dependência de UI. Testado ponta a ponta com sucesso: problema de
teste no Zabbix → ticket criado no GLPI (ticket id 2, "Problem: Teste
integracao GLPI..."). Um "API client" também precisou ser criado no GLPI
via SQL direto (não existe CLI para isso) — autorizado explicitamente pelo
usuário nesta sessão, ver `docs/DECISIONS.md`.

**Pendência menor:** configurei a "recovery operation" da Action do Zabbix
para adicionar um followup no ticket quando o problema é resolvido, mas o
teste não confirmou esse disparo (o problema resolveu corretamente no
Zabbix, mas nenhum alerta de recovery apareceu no log). A criação de
ticket (o requisito principal) está confirmada funcionando; o followup
automático de resolução fica como próximo passo a debugar, não é
bloqueante para a entrega de hoje.

**Item de teste deixado no ambiente:** item trapper `teste.glpi.trap` +
trigger "Teste integracao GLPI: valor de teste recebido" no host
"Zabbix server" do stack FLUA — serve para re-testar a integração a
qualquer momento com
`docker exec flua-zabbix-server zabbix_sender -z 127.0.0.1 -p 10051 -s "Zabbix server" -k teste.glpi.trap -o 1`.
Pode ser removido quando não for mais necessário.

**GLPI exposição pública — feita em 2026-07-12.** Labels Traefik
adicionadas ao serviço `glpi` (mesmo padrão de zabbix-web/grafana: Host,
entrypoint websecure, tls.certresolver=letsencrypt), mapeamento de porta
`127.0.0.1:8082` removido (acesso agora só via Traefik/DNS). DNS
confirmado via `dig glpi.flua.npxit.com.br @8.8.8.8` → `187.110.164.126`
antes de recriar o container. Certificado de produção emitido de primeira
(sem precisar de staging — mesma conta/resolver ACME já validada pelos
outros hosts do domínio), válido até 2026-10-10. Validado com `curl` sem
`-k` → 200, sem erro de certificado.

## Fase 1 (Let's Encrypt) — fechada

Sequência que destravou (DNS que antes não existia foi criado apontando
para `187.110.164.126`):

1. DNS confirmado propagado via `dig @8.8.8.8` para os 4 hosts.
2. Restart do Traefik em **staging** → certificados `(STAGING) ...` emitidos
   e confirmados via `openssl s_client` para os 4 hosts (subject batendo
   com cada host, issuer Let's Encrypt staging).
3. Removida a flag `--certificatesresolvers.letsencrypt.acme.caserver`
   (produção é o padrão quando ausente).
4. `acme.json` (que só tinha certificados de staging) foi zerado — backup
   preservado em `letsencrypt/acme.json.staging-backup` — para forçar
   reemissão pela CA de produção em vez de reaproveitar os certs de
   staging já válidos.
5. Restart do Traefik → log confirmou `acmeCA=https://acme-v02.api.letsencrypt.org/directory`
   (produção) + `Register...` (nova conta ACME de produção criada) sem
   erros.
6. Validado com `curl` **sem `-k`** para os 4 hosts — todos fecharam TLS
   confiando na cadeia do sistema (200/302/401, sem erro de certificado).
   `openssl x509 -issuer` confirmou `O = Let's Encrypt` sem o prefixo
   `(STAGING)` em todos os 4.

### Divergência de IPs (anotada, não bloqueante)

O DNS aponta para `187.110.164.126`, que **não é** o IP de saída desta VM
(`187.110.164.122`, via `curl ifconfig.me`) — isso é esperado em um
FortiGate com IP de WAN diferente para inbound (port-forward) vs outbound
(NAT), e o próprio sucesso da emissão via HTTP-01 (que exige o FortiGate
rotear `.126:80` até este host) confirma que o roteamento inbound está
funcionando corretamente. Não é uma pendência, só um detalhe de rede válido
para constar.

## Pendências conhecidas (não bloqueantes)

- Senha padrão do Zabbix (`Admin`/`zabbix`) ainda não foi trocada — ver
  `docs/ACCESS.md`.
- `docs/portal/` continua vazio, sem uso definido ainda.
- Cofre de senhas (Vault/Bitwarden) ainda não avaliado — ver
  `docs/DECISIONS.md` para o critério de quando priorizar isso.
- `letsencrypt/acme.json.staging-backup` pode ser removido quando não for
  mais necessário como histórico (não é sensível — só contém certificados
  de staging, que nenhum navegador confia).
