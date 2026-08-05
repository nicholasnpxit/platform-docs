# LEIA ISSO PRIMEIRO — orientação rápida (atualizado 2026-07-30, sessão de consolidação)

> Este bloco existe pra qualquer sessão nova (Claude Code, Cursor, ou
> outra ferramenta) conseguir se situar em segundos, sem precisar ler
> o arquivo inteiro (3200+ linhas) antes de fazer qualquer coisa. Não
> substitui a releitura completa de `CLAUDE.md`/`docs/ROADMAP-MACRO.md`
> exigida no início de toda sessão — é um resumo de orientação, não a
> fonte de verdade (o resto deste arquivo, cronológico, é a fonte real).

**O que este produto é, resumido:** plataforma self-service de
instâncias de TI prontas para uso (não MSP, não consultoria) — catálogo
de 8 apps (Zabbix, Grafana, GLPI, Vaultwarden, Uptime Kuma, BookStack,
Chatwoot, CrowdSec — conferir `docs/ROADMAP-MACRO.md` seção 6 pro estado
exato de cada um, catálogo evolui), hierarquia fixa ADMN → cliente
pagante (MSP ou empresa) → cliente final dele, assistente de IA por
tenant (isolamento lógico testado, isolamento físico por VM ainda
pendente — ver seção 10 do macro), NOC interno da NPX (monitora só a
entrega da própria plataforma, nunca a rede do cliente), backup granular
por instância via Kopia. Meta comercial: R$150k/mês bruto até
dez/2026 — ver `docs/ROADMAP-MACRO.md` seção 0 pro contexto completo.

**O que está rodando NESTE EXATO MOMENTO (verificado ao vivo, não só
lido em doc — confira `docs/ACTIVE-SESSION.md` pra confirmação em tempo
real, pode ter mudado desde que isso foi escrito):** Em 2026-08-05:
IA multi-chat/auditoria OK; MSP block + NOC-lite + migração FLUA→MIP OK;
**frontend `vsadmnfront` em modo preparação** (Docker+Traefik+discovery
HTTP, **sem cutover DNS/VIP**); **limite IA L1 auto + sub-alocação L2 OK**.

**Bug seletor Cliente preso (2026-07-30): CORRIGIDO em 2026-08-03** — ver
seção Fase 0 abaixo. Causa: soft nav + Server Action + redirect com
origem interna atrás do Traefik.

**Os 4 grandes temas que entram a seguir** (detalhados por completo em
`docs/ROADMAP-MACRO.md`, seções novas — **nada disso foi implementado
ainda, só planejado/documentado**):
1. **Ferramenta de migração/onboarding** (prioridade alta, "pra ontem")
   — cliente migrar de ambiente externo pra plataforma sem perder dado.
2. **Relatório de segurança real** — consolidar o que já existe em
   `docs/SECURITY.md` num relatório executivo (implementado × lacuna).
3. **Segregação de infraestrutura** — frontend em preparação
   (`vsadmnfront`); cutover DNS/VIP **ainda não**; banco dedicado
   (`vsadmndb`) ainda fora.
4. **Estudo de capacidade de hardware** — dimensionar pra meta de 500
   clientes em 6 meses, com base em uso real medido, não estimativa.

---

# Estado atual — npx-platform

Última atualização: **2026-08-05 — monstro Parte 5 (VIP temp + checklist)**



## 2026-08-05 — SERVICE-ACCOUNTS Parte 6 — CONCLUÍDA

- **A** (token GitHub): continua ação manual do responsável — confirmado.
- **B** (UID portal): **fora**, sem autorização explícita — não mexido.
- **C** (MIP `docker exec`): **fechado** — API via
  `https://zabbix.flua.npxit.com.br/api_jsonrpc.php`; `svc-mip-watcher`
  removido do grupo `docker`; `--check-proxy` OK.

## 2026-08-05 — Cutover checklist + VIP temp front (Parte 5) — CONCLUÍDA

Prompt: `PROMPT-CURSOR-monstro-ui-escopo-pendencias-2026-08-05.md` Parte 5.
Checklist: `docs/CUTOVER-FRONTEND-CHECKLIST.md` reescrito:
1) fechar 80/443 da app no **mesmo evento** do cutover (não opcional);
2) etapa VIP temporário **antes** do VIP de produção.

VIP temp aplicado: `187.110.164.124` (`vsa10_80`/`vsa10_443`) →
`172.16.11.155`. Evidência:
`docs-publish/validation/frontend-vip-temp-2026-08-05/` (HTTP/HTTPS 200
whoami). **Produção `.126` não alterada.**

## 2026-08-05 — Frontend `vsadmnfront` preparação (SEM cutover) — CONCLUÍDA

Prompt: `PROMPT-CURSOR-frontend-migracao-e-ia-credito-2026-08-05.md` Parte A.
Evidência: `docs-publish/validation/frontend-prep-2026-08-05/`.
Checklist cutover (não executar): `docs/CUTOVER-FRONTEND-CHECKLIST.md`.

1. Docker CE 29.7.1 + compose no `vsadmnfront` (172.16.11.155).
2. Traefik `traefik-front` em `/opt/npx-front/` — **só** `providers.http`
   (+ file vazio), **sem** docker.sock remoto.
3. Portal: `GET /api/internal/traefik-discovery` (token
   `TRAEFIK_HTTP_PROVIDER_TOKEN`), porta LAN `172.16.11.150:3099→3000`.
4. Validação: Host `front-prep.test.npx.internal` via front → whoami
   `172.16.11.150:19080` HTTP/HTTPS 200. Cert atual = Traefik DEFAULT
   (LE público exige VIP/DNS pro front — fica no checklist de cutover).
5. Prova negativa: TCP 2375/2376 da app **fechados** a partir do front.
6. **DNS público e VIP FortiGate de produção NÃO foram alterados.**

## 2026-08-05 — Limite de gasto IA (L1 auto + subalocação L2) — CONCLUÍDA

Prompt: mesma data, Parte B. Evidência:
`docs-publish/validation/ia-limite-credito-2026-08-05/`.

1. `PlatformSettings.aiDefaultTenantLimitUsd` (default 5) + UI em
   `/settings/ai`.
2. `createTenantAction`: tenant nível 1 (pai ADMN) chama
   `ensureTenantOpenRouterKey` automaticamente.
3. Subtenants: tela `/tenants/[id]/ai-credits` aloca fatia do limite do
   pai; `resolveChatApiKey` bloqueia L2 sem chave (nunca fallback
   plataforma).
4. `provisionTenantAiKeyForm` continua com `requireTenantAccess`.
5. Validação: L1 teste nasceu com limit=5; L2 bloqueado até alocar US$1.

## 2026-08-05 — Migração real FLUA→MIP ENGENHARIA (§2) — CONCLUÍDA

Prompt: `PROMPT-CURSOR-msp-instancias-hierarquia-2026-08-04.md` §2.
Evidência: `docs-publish/validation/msp-migracao-flua-mip-2026-08-05/` (+ ensaio em
`msp-migracao-ensaio-2026-08-05/`).

1. **Ensaio** `mig-ensaio-msp`→`mig-ensaio-final`: volume com marcador
   `ENSAIO-MARKER-ZBX-HIST-88421` sobreviveu ao rename.
2. **Kopia fresco** imediatamente antes: zabbix=`cac05ea9…`,
   grafana=`9d6487d4…`, glpi=`881dcc3c…`.
3. **Stack** movida p/ `clients/mip-engenharia/` via
   `scripts/migrate-tenant-stack.sh --keep-hosts` — containers/volumes
   `mip-engenharia-*`; Host Traefik `*.flua.npxit.com.br` mantidos.
4. Portal: 3 `Instance` + credenciais agora em MIP; FLUA sem instâncias.
5. Prova: Zabbix login suporteti, **38 hosts** (igual ao antes), HTTP 200
   zabbix/grafana/glpi; porta 12051 inalterada no host.
6. Compose antigo da FLUA renomeado p/
   `docker-compose.yml.migrated-to-mip-2026-08-05`. Volumes `flua_*`
   antigos **não** foram purgados (rede de segurança; podem limpar depois).

## 2026-08-05 — Tela gestão MSP + NOC-lite (§3) — CONCLUÍDA

Prompt: `PROMPT-CURSOR-msp-instancias-hierarquia-2026-08-04.md` §3.
Evidência: `docs-publish/validation/msp-gestao-noc-lite-2026-08-05/`.

- `/tenants/[id]/clientes` ganhou painel NOC-lite (snapshot NOC filtrado
  à árvore do MSP + `stack_health_issues` escopado) e status por cliente.
- Menu MSP já apontava pra `/tenants/{id}/clientes`; `/clientes` ADMN
  continua ADMN-only (redirect confirmado).
- Validado logado como técnico FLUA (screenshot real).

Próximo: §2 migração FLUA→MIP (ensaio descartável → backup Kopia → real).

## 2026-08-05 — MSP não hospeda instância nele mesmo (§1) — CONCLUÍDA

Prompt: `PROMPT-CURSOR-msp-instancias-hierarquia-2026-08-04.md` §1.
Evidência: `docs-publish/validation/msp-block-instancia-2026-08-05/`.

- `canCreateInstance(..., accountType)` recusa `MSP` (ADMN bypass).
- Server Action + páginas `/instances/new` revalidam no banco.
- UI: aba `+ Criar`, botão da lista, NewMenu e sidebar escondem no MSP.
- Validado: técnico FLUA sem criar; `/new` → banner; MIP ainda cria; ADMN na FLUA ainda cria.

Próximo nesta sessão: §3 tela gestão MSP / NOC-lite; depois §2 migração FLUA→MIP.

## 2026-08-05 — IA: multi-chat, página Automação, auditoria real — CONCLUÍDA

Prompt: `PROMPT-CURSOR-ia-chat-historico-auditoria-2026-08-04.md`.
Evidência: `docs-publish/validation/ia-chat-historico-auditoria-2026-08-04/`.

- Schema `AiChatThread` 1:N (unique `tenantId+userId` removida; colunas
  `titulo`, `resumo_compacto`). SQL aditivo no Postgres — **não**
  `prisma db push --accept-data-loss` (schema do repo ainda diverge de
  tabelas comerciais no DB).
- Actions: `list/create/delete` threads; `send`/`load` com `threadId`;
  compactação em `lib/ai/chat-history.ts` quando > `AI_HISTORY_MODEL_LIMIT`.
- UI: `/tenants/[id]/ai` fullscreen (`AiChatWorkspace`); menu lateral
  **Automação**; FAB só atalho + Novo chat + Ver histórico.
- Auditoria: `/tenants/[id]/ai/auditoria` (ADMN/gestor) + tool
  `consultar_auditoria_ia`. Prompt: boas práticas de TI OK; nunca restore
  de backup.

Validado ao vivo (NPX IT, ADMN): 2+ threads; compactação com
`resumo_compacto` preenchido + UI COMPACTOK; auditoria com linhas
legíveis; IA consultou o log.

## 2026-08-04 — IA visão multimodal + confirmação perigosa + docs — CONCLUÍDA

Prompt: `PROMPT-CURSOR-ia-visao-doc-confirmacao-2026-08-04.md`.
Evidência: `docs-publish/validation/sessao-ia-visao-doc-confirmacao-2026-08-04/`.

**Item 1 (urgente):** imagens anexadas passam como `image_url` data-URI
no `runChatTurn`/`OpenRouter` (não mais só `textExtract` vazio). UI com
miniatura. Validado: modelo leu `CODIGO-VISAO-NPX-88421` + números da
imagem de prova.

**Item 2:** card vermelho `dangerous` com 2 etapas; botão envia frase
reforçada sem digitar; `needsMoreConfirmation` também gera card.

**Item 3:** `Instance.docsMarkdown`/`docsAtualizadoEm` + tool
`gerar_documentacao_instancia` (mutação com confirmação; escopo só app
do tenant) + render em `/tenants/[id]/docs`.

## 2026-08-04 — IA chat UX + bug dashboard Grafana — CONCLUÍDA

Prompt: `PROMPT-CURSOR-ia-chat-ux-e-bug-dashboard-2026-08-04.md`.
Evidência: `docs-publish/validation/sessao-ia-chat-ux-dashboard-2026-08-04/`.

**Item 4 (crítico, feito primeiro):** `criar_dashboard_grafana` criava
dashboard com `"uid":"${DS_ZABBIX}"` — painéis sem dado. Corrigido em
`lib/ai/app-tools.ts` (resolve placeholder via datasource real +
validação pós-save). Dashboard FLUA/MIP
`be205f52-d292-4e95-9bd3-6638c3eea5fa` apagado/recriado; screenshot
mostra contagens reais por host group (Impressoras/Switches/etc.).

**Itens 1–3:** `AiAssistantDrawer` — `onPaste` (imagem + texto ≥900 →
anexo); tools/JSON só com “Modo técnico” (ADMN); card de confirmação
com botão Confirmar (`pendingConfirmations` estruturado em
`chat.ts`/actions). Validado Playwright (`ux-log.txt`).

## 2026-08-04 — Redesign UI v2 (layout compartilhado + abas cápsula) — CONCLUÍDA

Prompt: `PROMPT-CURSOR-redesign-ui-v2-2026-08-04.md`. Evidência:
`docs-publish/validation/sessao-redesign-ui-v2-2026-08-04/`
(`01-tab-y.txt`: Geral Y=141 = Instâncias Y=141, delta=0;
close-up da cápsula ativa com `shadow-sm`; cor FLUA
`rgb(237, 50, 55)`; 390/768/1440/1920 sem overflow horizontal).

**Queixa pós-v1:** ainda “formulário Excel”; abas não pareciam abas;
trocar aba “carregava janela nova” (barra pulava de posição).

**Entrega:**
- `TabNav.tsx` reescrito (cápsula no trilho — código do prompt).
- `app/tenants/[id]/layout.tsx` — título + slug + abas + card de
  conteúdo fixos; `{children}` só troca o miolo. Active via
  `TenantTabNavClient` + `usePathname()` (mesmo padrão de
  `SidebarNav`).
- Páginas sob `/tenants/[id]/*` sem `AppShell`/`TenantTabNav`/
  título duplicado.
- Composição Geral: `grid-cols-[1fr_320px]` (form + lateral).
- `ContentGrid` padrão alinhado a esse grid.

Portal rebuildado (`npx-portal:latest`) e redes `_internal`
religadas.

## 2026-08-04 — Redesign UI v1 (abas + uso do espaço) — SUPERSEDED pela v2

Prompt: `PROMPT-CURSOR-redesign-ui-2026-08-04.md`. Evidência:
`docs-publish/validation/sessao-redesign-ui-2026-08-04/` (20 PNGs +
log responsivo). Responsável rejeitou visualmente; corrigido na v2
acima.

**Problema original:** cards `max-w-lg` em ~39 telas + navegação
“link · link” parecendo índice de manual (print Editar tenant FLUA).

**Entrega v1 (parcialmente mantida):** `PageContainer`/`FormStack`,
AppShell `max-w-7xl`, Cards sem teto estreito — a casca de abas e o
layout por página foram substituídos na v2.

## 2026-08-04 — Watchdog Zabbix server — CONCLUÍDA

Prompt: `PROMPT-CURSOR-zabbix-watchdog-2026-08-04.md`. Evidência:
`docs-publish/validation/sessao-zabbix-watchdog-2026-08-04/`.

**Incidente real (FLUA):** `flua-zabbix-server` Up mas sem processar
dado (~5 min) após "MySQL server has gone away" / inactivity — UI
mostrava “servidor não está rodando”; Docker/Portainer verdes. Restart
manual resolveu. Agora automatizado.

**Entrega:**
- `scripts/zabbix-server-watchdog.py` — atividade real via
  `MAX(clock)` em `history_uint` (+ log `gone away`/`inactivity`);
  `docker restart` 1× com cooldown 30 min; se falhar →
  `restart_failed=1` (trigger High).
- Cron suporteti `*/5` com `--apply --send-zabbix`.
- NOC mestre (`Docker-Host-suporteti`): 4 itens trapper
  `npx.zabbix_watchdog.*` + 4 triggers (stuck, age>15min, restart
  failed, auto-restart info).
- Prevenção: `wait_timeout`/`interactive_timeout=28800` explícitos no
  template MySQL Zabbix + compose dos tenants existentes (já era o
  default MySQL 8; documentado). Sem `DBReconnect` — deprecado no
  MySQL 8.0.34+ / Zabbix 7.

**Validado:** 6/6 servers saudáveis (incl. FLUA); force freeze em
`valid1` → detectou age=48s, restart OK, evento NOC “reiniciou tenant
automaticamente”.

## 2026-08-03 — Provisionamento resiliente — CONCLUÍDA

Prompt: `PROMPT-CURSOR-provisionamento-resiliente-2026-08-03.md`.
Evidência: `docs-publish/validation/sessao-provisionamento-resiliente-2026-08-03/`.

**Incidente real (felixti):** fila `glpi`/`glpi-2`/`glpi-3` em
`provisionando` após restart do portal no meio do health check (até
10 min). `rollback()` limpava Docker mas o placeholder só era apagado
no `.then()` em memória — sumia com o processo. Reconcile `--apply`
removeu `glpi-3` + containers/volumes; felixti ficou só com
zabbix/grafana/uptime_kuma/chatwoot ativos.

**Entrega:**
- Botão **Cancelar provisionamento** (flag `cancelamento_solicitado_em`,
  check entre etapas, `cleanupFailedPlaceholder` com log real).
- Debounce via `useFormStatus` + bloqueio server-side se já há
  `provisionando` do mesmo tipo; segunda instância só com checkbox
  `allowExtraInstance`.
- Catálogo de falhas (`provisioning-lifecycle.ts`) + retry 1× de health
  (+5 min) se o container existe.
- Script `scripts/provisioning-reconcile.py` + cron suporteti `*/10`
  (`--min-age-minutes 15`). Loop in-process no portal desligado (sem
  python3/docker.sock no container).
- Validado: felixti limpo; cancel E2E; restart+reconcile limpa
  bookstack de teste em valid1; duplicidade e `ja-existe-tipo` com
  banner.

## 2026-08-03 — WhatsApp self-service (permissão + instruções) — CONCLUÍDA

Prompt: `PROMPT-CURSOR-whatsapp-self-service-2026-08-03.md`. Evidência:
`sessao-whatsapp-selfservice-2026-08-03/`.

- Permissão: `canManageUsersInTenant` (admin do próprio tenant), não mais
  só ADMN. Validado com `teste1@teste.com` em valid1; acesso a flua
  redireciona `/dashboard`.
- Instruções passo a passo na própria GUI + aviso token 24h vs System
  User + sandbox; link para `/manuais/whatsapp` (9 páginas: 3 roles ×
  3 idiomas).

## 2026-08-03 — WhatsApp Cloud API — AUTOMACAO PRONTA (E2E Graph aguarda sandbox)

Prompt: `PROMPT-CURSOR-whatsapp-integracao-2026-08-03.md`. Evidencia:
`docs-publish/validation/sessao-whatsapp-2026-08-03/`.

**Pronto:** GUI credencial/templates em `/tenants/<id>/integrations/whatsapp`,
token AES-GCM, webhook Meta + relay autenticado, wire Zabbix/Grafana/Chatwoot,
hook GLPI em `createGlpiTicketForTenant`. Webhook challenge e auth do relay
validados com stub. UI Playwright OK.

**Aguardando (nao e bloqueio tecnico de codigo):** credenciais do numero
sandbox Meta em `/opt/npx-platform/whatsapp/.env` (token + phone_number_id +
WABA + destinatario de teste) para enviar mensagem real e submeter template
com status PENDING/APPROVED da Meta. Business Verification e aprovacao
humana de template continuam so do lado da Meta.

## 2026-08-03 — Fase 6 CrowdSec/Pi-hole — AVALIADA, NÃO ATIVADA

Evidência: `sessao-fechamento-fase6-2026-08-03/`. Traefik sem bouncer;
FortiGate sem helpers VPN. Preferência do prompt: não forçar.

## 2026-08-03 — Fase 5 Trial/Demo — CONCLUÍDA (escopo reduzido, sem VM nova)

Evidência: `sessao-fechamento-fase5-2026-08-03/`. 30 dias, 1×/produto,
autolimpeza real (stop+pausado; remove `teste-*` após 7d). UI ADMN +
checkbox no provisionamento. Cron 07:15. Sem cartão (decisão aberta).

## 2026-08-03 — Fase 3 backup agendado — CONCLUÍDA

Evidência: `sessao-fechamento-fase3-2026-08-03/`. Snapshot Kopia real via
cron path; UI ADMN em `/backups/admin`.

## 2026-08-03 — Fase 2 inadimplência e-mail — CONCLUÍDA

Evidência: `sessao-fechamento-fase2-2026-08-03/`. E-mails 5/15/25 Brevo OK;
suspensão real de `teste-dunning-*-web`; auditoria `dunning_events`.

## 2026-08-03 — Fase 1 Chatwoot IA comercial/técnica + handoff — CONCLUÍDA

Evidência: `docs-publish/validation/sessao-fechamento-fase1-2026-08-03/`.
Agentes estendem `lib/vitrine/agent.ts`: comercial (só `campaigns` ativas),
técnico (BookStack/catálogo/manuais/KB, sem tools), handoff Chatwoot com
token humano + histórico intacto, identificação por e-mail→tenant.
TOTAL_LLM_CALLS=**0**.



## 2026-08-03 — Fase 4 manutenção automática de disco — CONCLUÍDA

Prompt: `docs/PROMPT-CURSOR-manutencao-disco.md` (via
`PROMPT-CURSOR-crm-e-fechamento-2026-08-03.md` Fase 4). Evidência:
`docs-publish/validation/sessao-disco-fase4-2026-08-03/`.

**Entregue:**
- Camada 1: `scripts/lib/test_cleanup_guard.py` +
  `scripts/lib/test-cleanup-guard.sh` + inventário de scripts de teste
  (`11-audit-cleanup-camada1.txt`).
- Camada 2: `scripts/docker-maintenance.py` (dry-run default / `--apply`;
  modos orphans|cache|images|all; carência órfãos 2h; Docker 29
  `--max-used-space=15GB` + `until=48h`; imagens `until=6h`; cruzamento
  obrigatório com `tenants` via `portal-db`).
- Cron `suporteti`: */20 orphans+zabbix; 03:15 mode all.
- Zabbix `Docker-Host-suporteti`: 4 itens trapper + triggers (watchdog
  2h, build cache >25GB, suspicious min 1h, disco /hostfs >80%/90%) +
  trigger auxiliar de validação imediata. PROBLEM real confirmado
  (`07-triggers-PROBLEM.json`).
- Grafana uid `npx-docker-maintenance` (+ latest data Zabbix com valores
  reais — painéis timeseries do plugin ainda “No data” em alguns filtros;
  fonte de verdade dos valores: API/Latest data).
- Validação 7/7: dry-run, positivo, negativo 3×, grace, Zabbix, NOC/
  Grafana, docs.

Disco agora: **29%** (`df`); build cache após 1ª limpeza `--mode cache`:
**16.65GB** (antes 22.15GB; teto alvo 15GB — cron 03:15 continua).

## 2026-08-03 — Fase 0 CRM comercial interno — CONCLUÍDA

Prompt: `docs/PROMPT-CURSOR-crm-e-fechamento-2026-08-03.md` (começar por
Fase 0 + Fase 4). Evidência:
`docs-publish/validation/sessao-crm-fase0-2026-08-03/` (prints 01–09 +
`summary.txt` + tenant descartável `teste-crm-1785765133`).

**O que entrou (estende o comercial existente, não substitui):**
- Tabelas raw SQL: `campaigns`, `campaign_sale_items`, `contracts`,
  `contract_lines`; colunas `tenants.commercial_status` e
  `platform_settings.commercial_mrr_goal_cents`.
- Lib `portal/src/lib/crm.ts` + actions `portal/src/app/crm/crm-actions.ts`.
- UI ADMN: `/crm` (lista N1 + painel meta MRR vs R$150k),
  `/crm/campaigns`, `/crm/tenants/[id]`, contratos novos,
  edição de `/sales/items/[id]`. Nav: CRM + itens de venda.
- Validação E2E Playwright: campanha 10% → contrato (item
  `Monitoramento Zabbix (teste)` R$100 → MRR R$90 após campanha) →
  status `inadimplente` na lista → barra de meta 0,1% de R$150.000.
  Produto extra `Zabbix Managed CRM 1500` seedado via SQL (o POST de
  criar item em `/sales/items/new` nesta sessão limpou o cookie
  `npx_session` no browser headless — campanha/contrato/status OK após
  re-login; editar item existente permanece funcional — ver DECISIONS).

**Fora de escopo respeitado:** sem gateway de pagamento, sem Kanban,
sem UID portal/`suporteti`, sem domínio/WhatsApp/OAuth.

**TOTAL_LLM_CALLS (Fase 0 CRM):** 0.

---

## 2026-08-03 — Fase 3.6–3.9 — CONCLUÍDAS

Evidência: `docs-publish/validation/sessao-onda-fase3-3.6-3.9-2026-08-03/`.

**3.6 Métricas por tenant:** `/tenants/<id>/metrics` agrega CPU/mem/disco
só dos containers do tenant (Portainer — mesmo escopo das cards). Nunca
host inteiro do `npx-zabbix`. Print `01-metrics.png`.

**3.7 Export logs:** `GET /api/tenants/.../instances/.../logs/export`
(text/plain attachment, ~500 linhas/container). Link na InstanceCard.
Sample `02-logs-export-sample.txt`.

**3.8 Widgets:** `/tenants/<id>/board` — catálogo health/backups/IA/GLPI,
reordenar, persistido em `platform_kv` por usuario+tenant. Print
`03-widget-board.png`.

**3.9 SSO GLPI:** formulário em `/tenants/<id>/sso` (provider `glpi`);
`applyGlpiSso` sobe oauth2-proxy em `sso.<dominio-glpi>` e **não**
coloca ForwardAuth no Host principal (login local permanece). Ativação
live IdP depende de credencial OIDC do tenant — UI+mecanismo prontos;
FLUA compose confirma router `glpi.flua.npxit.com.br` sem oauth2 ate
ativar (`05-glpi-local-login-intact.txt`).

TOTAL_LLM_CALLS(3.6–3.9)=**0**.

**Validação final isolamento IA (determinístico):** evidência em
`docs-publish/validation/sessao-onda-fase3-final-isolation-2026-08-03/`
(`validate-ia-hierarquia` sem LLM).

**TOTAL_LLM_CALLS onda inteira (Fases 0–3):** Fase0=0, Fase1=0, Fase2=1,
Fase3=0 → **1** chamada real de LLM no total da rodada.

## 2026-08-03 — Fase 3.5: destino Kopia rede/NAS (sem OAuth) — CONCLUÍDA

Prompt: §3.5. Evidência:
`docs-publish/validation/sessao-onda-fase3-3.5-2026-08-03/`.

**UI ADMN** `/backups/destination`: filesystem (path de mount NFS/SMB no
host — Kopia não tem SMB nativo) + SFTP nativo + objeto cloud. Validação
rejeita `rclone_*` até OAuth. Troca arquiva em `previousDestinations`;
agent `POST /destination` atualizado. Teste: save
`data-nas-test` → previous list → restore `data` (`00-log.txt` PASS).
OneDrive/GDrive **não** ativados (fora de escopo). TOTAL_LLM_CALLS=**0**.

## 2026-08-03 — Fase 3.4: proxies Zabbix pelo painel — CONCLUÍDA

Prompt: §3.4. Evidência:
`docs-publish/validation/sessao-onda-fase3-3.4-2026-08-03/`.

**UI:** `/tenants/<id>/instances/<zabbixId>/proxies` — listar/criar/remover
via API Zabbix 7 (`proxy.get/create/delete`) como `suporteti`. Link na
InstanceCard só quando `tipo=zabbix`.

**Validação (valid1):** create `?created=1&proxyid=2` + delete
`?deleted=1` (prints 02/03). TOTAL_LLM_CALLS(3.4)=**0**.

## 2026-08-03 — Fase 3.3: certificado próprio por instância — CONCLUÍDA

Prompt: `docs/PROMPT-CURSOR-onda-grande-2026-08-02.md` (§3.3).
Evidência: `docs-publish/validation/sessao-onda-fase3-3.3-2026-08-03/`.

**Mecanismo:** file provider Traefik já existente (`traefik/dynamic` +
`TRAEFIK_DYNAMIC_DIR` no portal). Upload `.crt`+`.key` → validação
`crypto.X509Certificate` (CN/SAN + `notAfter`) → arquivos em
`dynamic/certs/<instanceId>/` + YAML `tls-custom-<id>.yml` → remove
label `tls.certresolver=letsencrypt` do router (senão ACME compete).
Campo `Instance.tlsMode` (`letsencrypt`|`custom`). UI na InstanceCard;
ações `uploadCustomCertAction` / `clearCustomCertAction`.

**Isolamento real (2+ hosts simultâneos):**
- `uptime-kuma-2.valid1.npxit.com.br` → issuer/subject =
  `CN=uptime-kuma-2.valid1.npxit.com.br` (self-signed de teste)
- `admn.npxit.com.br` → Let's Encrypt (YR2) intacto
- `grafana.flua.npxit.com.br` → Let's Encrypt (YR1) intacto

TOTAL_LLM_CALLS(3.3)=**0**.

## 2026-08-03 — Fase 3.1: domínio próprio self-service — CONCLUÍDA (LE E2E bloqueado)

Prompt: `docs/PROMPT-CURSOR-onda-grande-2026-08-02.md` (§3.1).
Evidência: `docs-publish/validation/sessao-onda-fase3-3.1-3.2-2026-08-03/`.

**UI:** modo "Dominio proprio do cliente" em
`ServiceAndDomainPicker` + checklist DNS A → `187.110.164.126`.

**Gate server-side:** `dnsPointsToNpxWan` em create
(`provisionInstanceAction`) e update (`updateInstanceDomainAction`) —
placeholders `*.example` / domínio-base ofuscado isentos. Validado:
submit com `example.com` →
`?error=dns-nao-aponta&detail=DNS resolve para …, esperado 187.110.164.126`
(print `04-dns-gate-reject.png`).

**Bloqueio E2E Let's Encrypt real:** não há subdomínio descartável com
DNS controlável nesta sessão (regra 4 do prompt — não inventar/comprar
domínio). Mecanismo Traefik+LE já existe; falta só DNS real apontando
pro WAN pra fechar HTTPS de ponta a ponta. TOTAL_LLM_CALLS(3.1)=**0**.

## 2026-08-03 — Fase 3.2: domínio-base ADMN configurável — CONCLUÍDA

Prompt: §3.2. Evidência: mesma pasta `sessao-onda-fase3-3.1-3.2-2026-08-03/`.

**Schema:** `PlatformSettings.deliveryDomainBase` (coluna SQL
`delivery_domain_base`). Tela ADMN `/settings/platform` com checklist
DNS coringa. Fallback: env `OBFUSCATED_DELIVERY_DOMAIN` /
`instancias-teste.example`. Instâncias antigas **não** são tocadas;
`suggestObfuscatedDomainAsync()` usa o valor do banco nas novas.

**Validação:** save idempotente (`02-platform-settings-saved.png`);
sugestão ofuscada termina em `.instancias-teste.example`
(`05-obfuscated-suggestion.png`). Valor continua placeholder — **não**
comprar domínio real. TOTAL_LLM_CALLS(3.2)=**0**.

**Nota build:** apóstrofo em string de `delivery-domain.ts` (`Let's`)
quebrava o webpack; corrigido. Builds anteriores “ok” estavam em
cache sem a rota `/settings/platform`.

## 2026-08-03 — Fase 2: pendências wizard auditor — CONCLUÍDA

Prompt: `docs/PROMPT-CURSOR-onda-grande-2026-08-02.md` (Fase 2).
Evidência: `docs-publish/validation/sessao-onda-fase2-2026-08-03/`.

**Novas tools:** `auditar_bookstack`, `auditar_nextcloud`, `auditar_chatwoot`.
**Uptime Kuma aprofundado:** lê `kuma.db` via `execInContainer` —
`monitor-sem-notificacao` (todos os monitores), `intervalo-excessivo`,
`tls-expiry-desabilitado`.

**Validação determinística (`01-deterministic.json`):** Kuma plantou
monitor sem notificação e detectou; BookStack (npx) token OK + finding
real `suporteti-sem-admin-bookstack`; Chatwoot (valid1) OK; Nextcloud
**SKIP** — nenhuma instância ativa na plataforma (tool implementada;
retorna erro claro se ausente).

**Busca externa C.5.2:** não implementada (opcional).

**LLM:** TOTAL_LLM_CALLS = **1** (wizard chama `auditar_chatwoot` primeiro).

## 2026-08-03 — Fase 1: central de manuais do portal — CONCLUÍDA

Prompt: `docs/PROMPT-CURSOR-onda-grande-2026-08-02.md` (Fase 1).
Evidência: `docs-publish/validation/sessao-onda-fase1-2026-08-03/`.

**Arquitetura:** manuais do **produto** (como usar o portal) vivem em
`/manuais` dentro do portal — herdam branding do shell. BookStack da
vitrine **não** foi “pintado” por tenant (decisão do prompt). Conteúdo
genérico na tabela `manual_pages` (sem campo tenant), 12 áreas × 3
idiomas = **36** páginas. Imagens reais em `portal/public/manuals/`
(tenants de validação valid1/validnivel2 — zero dado de cliente
pagante).

**Visibilidade:** ADMN vê 3 níveis; usuário nível 1 vê nivel1+nivel2;
nível 2 só nivel2 (`07-roles.txt`).

**Validação:** screenshots ADMN / EN / ES / FLUA contexto /
`teste1@teste.com` / `gestorn2@teste.com` + detalhe com imagem.

**TOTAL_LLM_CALLS (Fase 1) = 0** (conteúdo escrito direto, sem chat de
IA).

**Nota schema:** `manual_pages` criada via SQL (`CREATE TABLE IF NOT
EXISTS`) — `prisma db push` recusou por drift de tabelas SQL antigas
fora do schema (não usamos `--accept-data-loss`).

## 2026-08-03 — Fase 0: bug "preso na visão Cliente" — CONCLUÍDA

Prompt: `docs/PROMPT-CURSOR-onda-grande-2026-08-02.md` (Fase 0).
Evidência: `docs-publish/validation/sessao-onda-fase0-2026-08-03/`.

**Causa raiz confirmada (não era só cache genérico):**
1. Toggle Plataforma/Cliente usava Server Action + soft navigation sem
   `revalidatePath('/', 'layout')` (o seletor de idioma já fazia isso) —
   Router Cache podia manter o shell antigo até hard refresh.
2. Após rebuild/restart do `portal`, IDs de Server Action mudam e o form
   soft falha até hard refresh — casa com o relato (~4 min + Ctrl+Shift+R).

**Correção:**
- `tenant-switch/actions.ts`: `revalidatePath('/', 'layout')` + log
  `[nav-mode]`.
- UI do toggle/picker em `SidebarNav` passou a usar **GET
  `/api/nav-mode`** (redirect 303 full document) — imune a ID de Server
  Action e a Router Cache stale.
- Redirect usa `PORTAL_URL` / `x-forwarded-*` (nunca `localhost:3000` do
  container atrás do Traefik — isso derrubava a sessão no primeiro teste).

**Validação real (Playwright):**
- 3 ciclos Cliente ↔ Plataforma imediatos (`00-log.txt` PASS).
- Troca de tenant (FLUA) + volta Plataforma sem hard refresh.
- Após `docker restart portal`, cookies reutilizados: click Plataforma →
  `/noc` imediato; Cliente → picker (`00-post-restart.txt` PASS).
- Screenshots: `02-mode-cliente.png`, `05-back-plataforma.png`,
  `07-after-restart-plataforma.png`.

**TOTAL_LLM_CALLS (Fase 0) = 0**

## 2026-08-02 — IA hierárquica / memória / wizard / KB + MSP (Parte A+B)

Prompt: `docs/PROMPT-CURSOR-ia-hierarquica-msp.md`.
Evidência: `docs-publish/validation/sessao-ia-hierarquica-2026-08-02/`.

### Parte A — IA
- **Hierarquia:** `ToolContext` + `resolveOperableTenantId` /
  `listar_subtenants` / `targetTenantId` nas app-tools. Ator nível 1
  (FLUA) owns-check positivo em MIP; negativo em VALID1; MIP não opera FLUA
  (`01-deterministic.txt`).
- **Memória:** SSR em `/tenants/[id]/ai` + `AppShell` passa
  `initialMessages` ao drawer; load pega as **últimas** 50 (não as mais
  antigas). Screenshot `04-ai-history-ssr.png` (`memoria=true`).
- **Limite modelo:** últimas **40** mensagens por turno
  (`AI_HISTORY_MODEL_LIMIT`) — ver DECISIONS.
- **Tempo real / escopo / anti-vazamento:** system prompt reforçado em
  `lib/ai/chat.ts`.
- **Wizard:** botão "Assistente guiado" no drawer (`09-drawer-wizard.png`).
- **KB agregada:** tabela `ai_knowledge_base` **sem** coluna tenant;
  curadoria ADMN em `/settings/ai/knowledge`; seed NVR/SNMP; injeção no
  prompt via `lib/ai/knowledge.ts` (`08-knowledge.png`).

### Parte B — MSP
- **Branding N2:** MIP locked + readonly herdado
  (`05-branding-mip-locked.png`, `locked=1`).
- **Cota diminuir:** banner `needsConfirm` ao salvar max &lt; usados
  (`07-quota-confirm.png`); instâncias **não** apagadas.
- **Consumo árvore:** bloco rollup na tela de cotas
  (`06-quotas-rollup.png`).
- **Profundidade:** create sob MIP continua `profundidade-maxima`.
- **Grupos N2:** `/tenants/[id]/groups` já usa `canManageUsersInTenant` +
  `hasAccessToTenant` — MSP com acesso ao filho gerencia grupos do filho.

### LLM (economia de token)
**TOTAL_LLM_CALLS = 3** (só estas, reais, via OpenRouter — ver
`02-llm-calls.txt`):
1. A.5 wizard (1 pergunta por vez)
2. A.6 off-topic (recusa clima)
3. A.6 probe cross-tenant (não confirma FLUA fora do escopo)

Hierarquia/memória/KB/cota/branding validados **sem** LLM.

---

## 2026-08-02 — Migração/onboarding externo + IA real por tenant

Evidência: `docs-publish/validation/sessao-migracao-ia-2026-08-02/`.

### Parte 1 — Migração
- Pesquisa: `docs/templates/MIGRACAO-{zabbix,grafana,glpi,vaultwarden,uptime-kuma,bookstack,chatwoot,crowdsec}.md`
- Manuais: `MANUAL-MIGRACAO-*.md` + screenshot Zabbix real (`manual-zabbix-host-list.png`)
- Agente: `migration-agent/npx-migration-agent.sh` — run real empacotou
  `npx-agent-test.tar.gz` (detectou zabbix/grafana/kuma/vw/glpi/chatwoot)
- UI: `/tenants/[id]/migration` (screenshot `04-migration-ui-valid1.png`);
  download agente autenticado (`npx-migration-agent.sh` baixado via UI)
- E2E: export host `npx-migra-descartavel` do demo →
  `configuration.import` no valid1 → host presente
  (`02-import-valid1.txt`: `import {"result":true}`, `hosts_after` com hostid 10683)
- Chatwoot/CrowdSec dump full: fora do self-service v1 (documentado)

### Parte 2 — IA app-tools
- Tools: `criar_host_zabbix`, `criar_dashboard_grafana`,
  `criar_categoria_glpi`, `configurar_entidade_glpi`, `ler_documentacao_tenant`
- E2E API (mesmo padrão suporteti/API das tools) em
  `03-ai-tools-e2e.txt`: host valid1 10684; Grafana dash uid
  `e38ce67a-…` HTTP 200; GLPI ITILCategory id=1 HTTP 201; DS Zabbix FLUA ok
- Screenshots: `04-grafana-ai-dash-render.png`
- Isolamento: `03-isolation-base.txt` PASS; owns cross-tenant 0
- VM dedicada IA (MACRO §10) **continua pendente**

---

## 2026-07-30 — Sessão 42 (fechamento fases 1–6)

Ver `docs/STATE-SESSAO-42.md`. Evidência:
`docs-publish/validation/sessao-42-2026-07-30/`.

| # | Tema | Status |
|---|------|--------|
| 1 | Migração edge → `*_internal` (demo/npx/validnivel2/npx-zabbix) | ✅ |
| 2 | PDF 1-clique orçamento + relatório executivo (pdfkit) | ✅ |
| 3 | i18n comercial EN/ES + enforce | ✅ |
| 4 | Backups NPX sem “Carregando…” residual | ✅ |
| 5 | Branding login (`?tenant=`) + e-mail reset | ✅ |
| 6 | `commercial_audit` (ticket/tempo/venda/quote/PDF/ops) | ✅ |

Residual conhecido (não bloqueia 1–6): OAuth rclone; DNS
`grafana-master.npxit.com.br`; i18n-debt residual fora do comercial (~28
arquivos isentos).

---

## 2026-07-30 — Sessão 41 (execução longa 1–41)

Ver detalhe + pendências honestas em `docs/STATE-SESSAO-41.md`.
Evidência: `docs-publish/validation/sessao-41-2026-07-30/`.

Resumo executivo: socket-proxy ✅ · isolamento lateral ✅ · Redis rate
limit ✅ · crédito IA bloqueio real ✅ · 2× uptime_kuma no valid1 ✅ ·
ticket GLPI 535 ✅ · BookStack produto+manuais ✅ · Chatwoot msg real ✅ ·
backups FLUA sem “Carregando” ✅ · seletor idioma com globo ✅ · picker
com busca ✅. Residual: DNS grafana-master, OAuth rclone, i18n debt das
telas comerciais, warn pontual backups NPX.

---

## 2026-07-29 — FASES 0–5 (auditoria + i18n enforce + IA multi-tenant + perfil + IPs + roadmap)

## 2026-07-29 — Sessão enforcement (FASES 0–5) — fechamento

### FASE 0 — Auditoria de realidade — CONCLUÍDA
Evidência: `docs-publish/validation/audit-reality-2026-07-29/`.
Ver tabela na seção abaixo (honesta).

### FASE 1 — i18n com enforcement no build — CONCLUÍDA (com dívida explícita)
- **Enforcement:** `portal/scripts/i18n-enforce.cjs` no `prebuild` — falha o
  build se string PT (diacríticos) ou denylist crítica aparecer em JSX fora
  de `i18n-debt.txt` / `i18n-messages.ts`.
- **Selftest:** `I18N_ENFORCE_SELFTEST=1` → EXIT 1 (prova: enforcement não é
  só cosmético). Crowdin/Lokalise adiados (custo/ops); checker local
  escolhido — ver `docs/DECISIONS.md`.
- **Telas críticas traduzidas de verdade:** NOC, backups admin, créditos IA
  ADMN/tenant, access IPs, etc.
- **Teste visual:** 27 rotas × 3 idiomas = 81 screenshots,
  `docs-publish/validation/fase1-i18n-enforce-2026-07-29/` (`evidence.json`
  `ok: true`, `leaks: 0` na denylist). Residual: 24 arquivos em
  `i18n-debt.txt` (usuários, formulários longos) ainda podem mostrar PT no
  corpo — **não** declarar 100% PT-free.
- **Causa raiz extra do falso EN:** `getRequestLocale` priorizava
  `User.locale` sobre o cookie — cookie `en` sozinho não mudava a UI.
  Corrigido: cookie vence; seletor grava cookie+User. Spot EN pós-fix:
  NOC/Backups/Credits em inglês (`spot2_en_*.png`).

### FASE 2 — IA em todo tenant + bypass cobrança — CONCLUÍDA
- Bypass marcado: `lib/ai/billing-bypass.ts` (`AI_BILLING_BYPASS_TEMP=true`)
  + banner UI `ai.billingBypassBanner`.
- Ferramenta `abrir_chamado_glpi` no chat do portal (GLPI **do tenant**, via
  `suporteti` + URL interna). Pré-requisito: tenant com GLPI ativo.
- Evidência E2E FLUA + Felix (≠ NPX): listar + diagnosticar + anexo 424242 +
  histórico — `docs-publish/validation/fase2-ai-all-tenants-2026-07-29/`
  (`ok: true`). Ticket GLPI FLUA HTTP 201 id=528 —
  `glpi-ticket-flua.json`.

### FASE 3 — Perfil links sociais — CONCLUÍDA
- Stub OAuth removido da UX; validação de formato de domínio + HTTP ≠ 404
  (`lib/social-links.ts`). Discussão OAuth real → `docs/ROADMAP.md`.

### FASE 4 — IPs WAN permitidos — CONCLUÍDA (VPN cancelada)
- Tela `/tenants/[id]/access`; campo `Tenant.allowedWanCidrs`; default
  vazio = sem restrição.
- Camada: **Traefik IPAllowList via labels Docker** (não FortiGate) —
  evidência valid1 deny 403 / allow 200 / open 200 em
  `docs-publish/validation/fase4-ip-allowlist-2026-07-29/evidence.json`.

### FASE 5 — Só ROADMAP — CONCLUÍDA
Itens registrados (sem implementar): revisão Traefik ponto único;
auditoria de segurança completa / anti-cópia.

---

## 2026-07-29 — FASE 0 Auditoria de realidade (obrigatória, ao vivo)

Evidência: `docs-publish/validation/audit-reality-2026-07-29/`
(`audit-report.md`, `audit-report.json`, `en-text-dump.json`, screenshots
`en_*.png`, `ai_*.png`, `menu_*.png`).

### Tabela resumo (honesta)

| Item | Veredicto (auditoria FASE 0) | Atualização pós FASES 1–4 |
|---|---|---|
| Assistente IA fora ADMN/NPX | **FUNCIONA** (listar) | **E2E completo** FLUA+Felix: listar+diagnóstico+anexo+histórico (`fase2-ai-all-tenants-2026-07-29`) |
| IA → chamado GLPI no chat portal | **NÃO EXISTIA** | **EXISTE** `abrir_chamado_glpi` (GLPI do tenant). Ticket FLUA id=528 HTTP 201 |
| Créditos IA / cobrança | Tela real; saldo não bloqueava | Bypass **marcado** (`AI_BILLING_BYPASS_TEMP`); banner UI; capacidades plenas |
| Perfil LinkedIn/etc. | Decorativo + stub OAuth | Validação formato+HTTP; OAuth real no ROADMAP |
| i18n EN/ES | Menu OK / conteúdo PT | Títulos críticos traduzidos + enforce no build; dívida em `i18n-debt.txt` |
| IPs WAN / VPN | (não existia; VPN cancelada) | `/tenants/[id]/access` + Traefik IPAllowList; default aberto |
| Menu Acronis upsell | Upsell sem produto | Mantido explícito em `/upsell?feature=` |

**Regra nova desta sessão:** nada “concluído” sem função E2E; upsell precisa permanecer claramente temporário/sem fingir produto.

---

## 2026-07-29 — Sessão redesenho ADMN FASES 0–10 (fechamento) — histórico

Evidência visual consolidada:
`docs-publish/validation/redesenho-admn-2026-07-29/` (`evidence.json`,
30+ screenshots PT/EN/ES, extras credentials/instances/upgrade/MSP,
mobile). BookStack: `fase5-bookstack-2026-07-29.json`. Backup FASE 8:
`fase8-portal-db-backup-2026-07-29.txt`. Hierarquia FASE 9:
`fase9-flua-mip-2026-07-29.json`. PDF FASE 10:
`fase10-doc-teste-msp.pdf` + `fase10-evidence-2026-07-29.txt`.

### FASE 0 — CONCLUÍDA
`docs/REDESENHO-ADMN-ACRONIS.md` lido integral e copiado em
`docs/DECISIONS.md` (`---BEGIN/END docs/REDESENHO-ADMN-ACRONIS.md---`).

### FASE 1 — i18n (conteúdo das páginas, não só menu) — CONCLUÍDA com checagem
- Dicionário `portal/src/lib/i18n-messages.ts` + legacy em `i18n.ts`
  (PT/EN/ES); cookie `npx_locale` + preferência `User.locale`.
- Telas-chave instrumentadas (credentials, clientes, dashboard titles,
  instances, security, profile, upsell, shell).
- Anti-regressão: `portal/scripts/check-i18n-hardcoded.cjs` → **OK**
  (`docs-publish/validation/fase1-i18n-check-2026-07-29.txt`).
- Teste real Playwright: 10 rotas × 3 idiomas + 11 extras × 3 idiomas;
  `evidence.json` com `ok: true`, `leaks: []` (zero strings PT da lista
  de vazamento em EN/ES nas rotas cobertas).
- **Residual conhecido:** textos longos de ajuda/erros fiscais e alguns
  corpos de formulário ainda em PT (não títulos da lista proibida).
  Ampliar dicionário em iterações futuras.

### FASE 2 — UI imediata — CONCLUÍDA (OAuth real bloqueado)
- Copiar user/senha em `/credentials` sem Revelar + auditoria
  (`copy_user`/`copy_password`).
- Header: `+ Novo` + idioma discreto (sigla) + `UserMenu` (foto/iniciais).
- `/settings/profile` (nome, cargo, bio, foto URL, redes).
- `/tenants/[id]/company` mesmos campos empresa + stub verificação.
- **Bloqueio OAuth:** sem `client_id`/`secret` LinkedIn/Instagram no
  `.env` — stub grava timestamp local e mensagem `oauth.pending`
  (não fingir verificação social real).

### FASE 3 — Menu Acronis — CONCLUÍDA (núcleo)
- Sidebar plataforma: categorias Clientes, Monitoramento, Caixa de
  entrada, Relatórios, Tarefas, Vendas, Minha empresa, Integrações,
  Configurações + Conta (upsell visível onde ainda não há produto →
  `/upsell?feature=`).
- MSP: **Cliente → Instâncias** (`/tenants/[id]/clientes` primeiro;
  sem lista solta de instâncias no menu MSP).
- Lista `/clientes`: sparkline 7d, tipo conta, 2FA mode, modo cobrança
  derivado, pastas; botão `+ Novo` contextual.
- Janela de manutenção editável no tenant (JSON `maintenanceWindow`).
- Evidência: `clientesHasSparkline: 7`, `clientesHasNewMenu: 1`,
  screenshots desktop/mobile.

### FASE 4 — Pastas cosméticas — CONCLUÍDA
- Model `ClientFolder`; criar pasta na UI `/clientes`; filtro por pasta;
  assign em MSP clientes. Pastas de teste “Região Sul” / “Prioridade
  Alta”; `clientesHasFolders: true`. **Não** afeta authz/isolamento.

### FASE 5 — BookStack — CONCLUÍDA
API real em `https://docs.npx.npxit.com.br` — 6 livros (uso + implantação
× PT/EN/ES), 39 entradas (páginas). Evidência
`fase5-bookstack-2026-07-29.json` (`ok: true`). Exemplos:
- https://docs.npx.npxit.com.br/books/manuais-de-uso-pt-br
- https://docs.npx.npxit.com.br/books/deployment-templates-en

### FASE 6 — CONCLUÍDA
Entrada em `docs/ROADMAP.md`: customização de dashboard por widget —
**não implementado**.

### FASE 7 — FINAL vs MSP — CONCLUÍDA (cobrança automática NÃO)
- Schema `accountType`, `whiteLabelEnabled`, `brandingFeeMonthlyCents`.
- Tenants teste: `teste-final` (FINAL), `teste-msp` (MSP).
- FINAL: CTA upgrade / intenção (`ResellerIntent`); sem criar subtenant.
- MSP: white label incluso; tela Meus clientes.
- Criação de tenant: campo tipo sob ADMN; filho de MSP sempre FINAL.
- Evidência screenshots extra `*__extra_final-up*` / `*__extra_msp-cli*`.

### FASE 8 — Backup Kopia Postgres portal — CONCLUÍDA (pré-requisito 9)
- Snapshot ID: `2b971ba7367b04a2ef7608b3bcd57e6d`
- Audit: `65a1d5be-b9e0-4045-8dc9-21383722e4d8`, tenant `admn`
- `docs-publish/validation/fase8-portal-db-backup-2026-07-29.txt`

### FASE 9 — FLUA MSP + MIP ENGENHARIA (só registro) — CONCLUÍDA com impasse visual de hosts
**Feito (só banco do portal, zero docker em apps FLUA):**
- FLUA `accountType=MSP`, `whiteLabelEnabled=true`
- Subtenant `mip-engenharia` / “MIP ENGENHARIA” filho da FLUA (`FINAL`)
- Instâncias FLUA inalteradas (`instancesUnchanged: true` —
  zabbix/grafana/glpi mesmas URLs)
- UI: MIP aparece sob FLUA (`fase9MipVisible: true`,
  `fase9-flua-clientes.png`)

**IMPASSE (não executado — regra de ouro):** fazer hosts/dashboards de
rede da MIP *dentro* do Zabbix/Grafana da FLUA aparecerem “agrupados”
como se fossem do subtenant MIP no painel exigiria metadados **dentro**
das apps (host groups/tags/folders Grafana) **ou** recriar/mover
instâncias — ambos tocariam produção de cliente. Opções documentadas
para decisão humana futura:
1. Só rótulo organizacional no portal (“hosts MIP vivem no Zabbix da
   FLUA”) — seguro, já refletido pela hierarquia de tenant sem mexer
   nas apps.
2. Tags/grupos dentro do Zabbix/Grafana da FLUA — muda config da app
   em produção → **exige confirmação humana explícita**.
3. Instância Zabbix própria da MIP — reprovisionamento → **proibido
   sem confirmação**.

Nada da lista de proibições da FASE 9 foi executado.

### FASE 10 — Doc com branding do tenant — CONCLUÍDA
- Análise estrutural dos PDFs de referência + gerador
  `portal/scripts/generate-tenant-doc.cjs`
- Artefato teste: `fase10-doc-teste-msp.html` + `.pdf` (59 194 bytes)
  com cor `#0f766e` / displayName “Teste MSP” (não branding FLUA/NPX
  fixo).

### Portal rebuild
Imagem `npx-portal:latest` rebuild/redeploy nesta sessão após as
mudanças de shell/menu/i18n/account-type.

---

## 2026-07-29 — Sessão redesenho ADMN (FASE 0 ok) — histórico início

**FASE 0 — CONCLUÍDA:** `docs/REDESENHO-ADMN-ACRONIS.md` lido **inteiro**
(106 linhas) e copiado integralmente para `docs/DECISIONS.md` (marcadores
`---BEGIN/END docs/REDESENHO-ADMN-ACRONIS.md---`, integridade verificada).

**Entendimento confirmado (antes de implementar):**
- Correções imediatas: copiar em credentials, menu usuário+idioma, perfil.
- Menu ADMN em ~10 categorias estilo Acronis; MSP nunca vê instância solta
  (sempre Cliente → Instâncias); sparkline 7d; +Novo; manutenção; upsell.
- Pastas só cosméticas; BookStack manuais 3 idiomas; roadmap widget;
  tipo FINAL vs MSP; backup Kopia antes de hierarquia FLUA/MIP;
  FASE 9 só registro visual — **proibido** tocar containers/apps FLUA;
  geração de doc com branding do tenant.

**Regra de ouro desta sessão:** produção de cliente = parar, documentar,
seguir — nunca adivinhar.

Fases 1–10 em execução a seguir.

---

## 2026-07-29 — Recovery credenciais (NOC “ativo sem credencial”)

Evidência: `docs-publish/validation/missing-creds-recover-2026-07-29/` (`login-evidence.txt`, `persist.txt`, `passwords.env` chmod 600).

**10/10** instâncias que o NOC sinalizava sem `InstanceCredential` recuperadas (login real + upsert portal + ACCESS.md):

- **felixti:** zabbix, grafana, glpi, uptime_kuma, chatwoot
- **valid1:** zabbix, vaultwarden, uptime_kuma, chatwoot
- **validnivel2:** zabbix

Checagem pós-recovery: zero instâncias ativas (não-platform-root) sem credencial.

---

## 2026-07-29 — Credenciais das 5 instâncias NPX (confiança)

Evidência: `docs-publish/validation/npx-creds-2026-07-29/` (`login-evidence.txt`, `persist.txt`, `passwords.env` chmod 600).

**Problema:** `/credentials` do tenant NPX mostrava "Sem credencial cadastrada" para Zabbix, GLPI, BookStack, Uptime Kuma e Chatwoot; responsável sem acesso.

**FASE 2 (recuperação, sem recriar):** logins nativos confirmados — Admin (Zabbix), glpi (GLPI), admin@npxit.com.br (BookStack/Chatwoot), admin (Kuma). Gravados em `InstanceCredential` + `docs/ACCESS.md`. Conta `suporteti` mantida (senha compartilhada); GLPI suporteti foi resetado pra bater com `SUPORTETI_PASSWORD`.

**FASE 3 (causa raiz):** `captureNativeCredential` no provisionamento + persistência obrigatória + alerta NOC `credenciais` + regra em `CLAUDE.md`. Portal rebuild/redeploy nesta sessão.

**Aberto:** outras instâncias ativas de outros tenants ainda podem estar sem `InstanceCredential` (NOC agora alerta) — recuperação pontual conforme aparecer.

---

## 2026-07-29 — Seletor modo Cliente × Dashboard (evidência)

Evidência: `docs-publish/validation/tenant-switch-2026-07-29/`.

**Antes:** picker “FLUA TI” mas lista misturava URLs flua+demo+felix+npx
(`tenantScopeFilter` ADMN → `{}`); em paralelo, fechar o dropdown no
`onClick` cancelava o Server Action (cookie não trocava).

**Depois (3+ tenants):** FLUA → só `*.flua.*`; Tulio → só `felixti*` /
felix; NPX → demo/npx; volta FLUA. Clique humano no picker OK.
Decisão completa em `docs/DECISIONS.md`.

---

## 2026-07-29 — BUG login lento + /clientes seletor (evidência)

Evidência: `docs-publish/validation/login-clientes-bugs-2026-07-29/`.

### BUG 1 — login ~18s → ~0,7s
- **Antes:** click→ready ≈ **18,2–18,9s** (5 runs); POST `/login` ~18s; reload `/dashboard` ~18s; `/noc` ~200ms.
- **Causa raiz:** visão executiva ADMN em `/dashboard` fazia até **12× `getContainerStats`** no Portainer (~1–2s cada, sequencial). O login redireciona para `/dashboard` (self-fetch RSC do Server Action) e herdava o custo. Hierarquia de tenants (7) e queries de `/clientes` **não** estavam no path do login. Coletor NOC compete por Portainer e agrava, mas sozinho o dashboard já custava ~18s.
- **Correção:** card de saúde lê `noc_snapshots` (mesmo cache do NOC); zero Portainer no path do dashboard/login.
- **Depois:** click→ready ≈ **661–909ms**; `/dashboard` ≈ **193–250ms**.

### BUG 2 — seletor de tenant em /clientes
- **Veredicto: design, não bug de re-render.** `/clientes` é visão de **plataforma** (todos os nível 1); não depende do tenant ativo.
- **UX:** em rotas de plataforma o seletor fica oculto; banner “Visão de plataforma inteira — seletor de tenant não se aplica nesta tela”; copy da página reforça.

---

## Sessão 2026-07-28 (noite) — 7 fases (performance NOC, bug Cliente, gestão)

Evidência: `docs-publish/validation/noc-clientes-2026-07-28/` (+ `evidence.txt`).

### FASE 1 — NOC cache + coletor background — CONCLUÍDA
- Página `/noc` **só lê** `noc_snapshots` (`getCachedNocSnapshot`); nunca coleta ao vivo.
- Coletor: `lib/noc/cache.ts` + `src/instrumentation.ts` (loop ~90s, `PlatformSettings.nocCollectIntervalSeconds`).
- UI: “Atualizado há Xs/min” + timestamp + duração da coleta (`data-testid=noc-age`).
- **Medição real:** coleta background ≈ **60s**; carga da tela ≈ **200–230ms** (5 amostras). Antes a página esperava a coleta inteira.

### FASE 2 — crash aba Cliente — CONCLUÍDA
- **Causa 1 (Application error):** `export { NAV_MODE_COOKIE }` re-exportado de `'use server'` file (`tenant-switch/actions.ts`) — Next.js exige só async functions. Removido; cookie vive em `lib/nav-mode.ts`.
- **Causa 2 (modo Cliente “não pega”):** cookie `npx_nav_mode=tenant` gravava, mas `AppShell` forçava de volta a Plataforma quando o seletor ficava vazio / dependia só de `accessibleTenantIds`. ADMN agora carrega **todos** os tenants `isPlatformRoot=false` no picker.
- Teste: 3× Plataforma↔Cliente + troca entre 6 tenants — sem Application error; sidebar Cliente com Dashboard/Instâncias. Screenshots `fase2-*.png`.

### FASE 3 — filtros NOC — CONCLUÍDA
- Checkbox “Só alertas/erros”; agrupar por categoria / cliente / serviço.
- Checks enriquecidos com `tenantNome`/`service` no coletor.

### FASE 4 — tela Clientes — CONCLUÍDA
- Rota `/clientes` (ADMN): tabela nível 1 com nome, doc, instâncias, subtenants, cotas, retenção, status; busca/ordenação/filtro; CSV filtrado via `/tenants/export?ids=`.

### FASE 5 — ações em massa — CONCLUÍDA
- Seleção múltipla + retenção backup nos **existentes** (`bulkSetBackupRetentionAction`) vs **padrão futuro** (`defaultBackupRetentionDays` em `PlatformSettings`) — conceitos separados.
- Confirmação explícita com contagem; auditoria em `bulk_action_audit`.

### FASE 6 — menu sem “Ações” — CONCLUÍDA
- Seção **Clientes** (Lista + Criar tenant); export CSV só na tela; Visão sem lista antiga `/tenants` no menu.
- Toggle Plataforma/Cliente polido (ring/sombra); forms do toggle sem `display:contents`.

### FASE 7 — pendências anteriores
- **DNS público:** `uptime|docs|chat.npx.npxit.com.br` → `187.110.164.126`; HTTPS com cert válido (`ssl_verify_result=0`) **sem** `--resolve`. Status page `/status/entrega` 200; Chatwoot `sdk.js` 200.
- **Naming / fusão de raízes:** não há investigação dedicada ainda aberta. Já resolvido antes: (a) ADMN `isPlatformRoot` vs raízes de cliente (2026-07-26); (b) multi-instância com `containerPrefix`/`slug` (2026-07-27). Nada pendente bloqueante.
- **OpenRouter Broadcast:** segue roadmap (não desta sessão).

### Schema novo
- `noc_snapshots`, `bulk_action_audit`; colunas `default_backup_retention_days`, `noc_collect_interval_seconds` em `platform_settings`.

---

## Menu lateral redesenhado (2026-07-28)

Toggle Plataforma/Cliente (ADMN), seletor hierárquico com subtenants
recuados, seções Visão/Clientes/Configuração (categoria **Ações removida**
na mesma noite — ver bloco acima). Evidência visual anterior:
`docs-publish/validation/nav-sidebar-2026-07-28/`; evidência atualizada:
`docs-publish/validation/noc-clientes-2026-07-28/`.


## VIPs FortiGate no NOC — falso negativo por NAT hairpin (2026-07-28)

**Veredicto: situação 1 (não-crítica).** Os "fail: timeout" do NOC eram
probe do *portal* (atrás do FortiGate) contra o IP WAN público
`187.110.164.126` — NAT hairpin. De fora, as portas reais estão **OPEN**.

Evidência literal: `docs-publish/validation/vip-hairpin-2026-07-28/evidence.txt`
- LAN/localhost 12051/12052/12056: OK
- WAN self-probe: TIMEOUT
- `api.networktools.dev` → 12051/12052/12056 **OPEN** (~140ms)

Correção: `collect.ts` probeia `NPX_HOST_LAN_IP`; 12053/12054 Liberada
no PORT-REGISTRY (órfãs).


## NOC interno + vitrine (2026-07-28) — evidência literal

### Escopo permanente
NOC interno monitora **só o que a NPX entrega** (nunca rede/roteador do
cliente). Regra em `CLAUDE.md`. Visão ADMN: `https://admn.npxit.com.br/noc`
(sidebar "NOC interno", só `canManageTenants` / `isAdmn`).

### Coletor + página `/noc`
- Código: `portal/src/lib/noc/collect.ts`, `portal/src/app/noc/page.tsx`.
- Playwright ADMN (2026-07-28): título "Saúde do que entregamos"; snapshot
  `ok=40`, `fail=8`, `unknown=5` (gerado `2026-07-28T22:43:30Z`).
  Screenshot: `docs-publish/validation/noc-vitrine-2026-07-28/noc-interno.png`.

### Uptime Kuma (tenant NPX) + status page
- Containers: `npx-uptime-kuma` (saudável). URL pública:
  `https://uptime.npx.npxit.com.br` — **DNS criado** (2026-07-28 noite);
  HTTPS com cert válido sem `--resolve` (status `/status/entrega` 200;
  API status-page 200).
- Status page publicada: `/status/entrega` — título "Status — o que
  entregamos"; `published=true`; grupo "Serviços públicos" com 13
  monitores HTTP (portal, GLPI NPX, Zabbix/Grafana demo+FLUA, Portainer,
  Traefik, etc.).
- Evidência antiga (com `--resolve`): mantida em
  `docs-publish/validation/noc-vitrine-2026-07-28/`. Evidência pública
  nova: `noc-clientes-2026-07-28/evidence.txt`.

### Vitrine: BookStack + Chatwoot + agente → GLPI
- **BookStack** `npx-bookstack`: livro "Catálogo de produtos" com 8 páginas
  (Zabbix, Grafana, GLPI, BookStack, Vaultwarden, Uptime Kuma, Nextcloud,
  Chatwoot). Search API `query=Zabbix` → page Zabbix.
- **Chatwoot** `npx-chatwoot`: Account "NPX", inbox "Chat site",
  WebWidget token `Z9ent1Jpcg3ymc18G5oZpqcT`, AgentBot "Vitrine Agent"
  com `outgoing_url=https://admn.npxit.com.br/api/vitrine/chatwoot-hook`.
  WhatsApp: **não** nesta fase (depende API Meta).
- **Agente** `portal/src/lib/vitrine/agent.ts` + rota
  `/api/vitrine/chatwoot-hook` (middleware público; auth por
  `VITRINE_WEBHOOK_SECRET`). Lê BookStack (fallback catálogo); se pedido
  exigir ação de conta → abre Ticket no GLPI (não executa ação sozinho).
- **GLPI NPX**: categorias ITIL (1 Informações, 2 Ação de conta, 3 Técnico);
  API client `vitrine-agent` (App-Token cifrado com GLPIKey); login API via
  Basic `suporteti`+senha.
- **E2E literal**:
  - GET agent `q=O que e o Zabbix…` → reply com conteúdo do catálogo/KB,
    `mode=kb_answer` (`docs-publish/validation/noc-vitrine-2026-07-28/vitrine-kb.json`).
  - GET agent `q=Quero contratar o Zabbix…` → `openedTicket=true`,
    `ticketId=2`, modo `open_ticket` (`vitrine-ticket.json`).
  - GLPI DB: tickets `#1` (teste API) e `#2` (agente), categoria 2.

### Grafana dashboard NOC INTERNO
- Importado em `npx-grafana` uid `npx-noc-interno` (tags `noc-interno`,
  `admn-only`). JSON fonte: `monitoring/npx-noc-interno-dashboard.json`.
  URL: `https://grafana.demo.npxit.com.br/d/npx-noc-interno/…`

### OpenRouter Broadcast
**Não implementado nesta sessão** — esforço desproporcional agora.
Requer Grafana Cloud **ou** Tempo/OTLP self-hosted + destino Broadcast no
OpenRouter. Registrado em `docs/DECISIONS.md` / roadmap.

### Bloqueio DNS (Azure / Microsoft 365) — ATUALIZADO 2026-07-28 noite
Registros A → `187.110.164.126` **já criados** para:
- `uptime.npx.npxit.com.br`
- `docs.npx.npxit.com.br`
- `chat.npx.npxit.com.br`
HTTPS público com Let's Encrypt válido confirmado (sem `--resolve`).
`grafana-master`/`zabbix-master` podem seguir no padrão antigo se ainda
sem DNS próprio.

### Segredos / env
- `clients/npx/.vitrine-secrets.json` (chmod 600)
- `clients/npx/.vitrine-bookstack-api.json` (chmod 600)
- Variáveis `VITRINE_*` em `portal/.env` + `portal/docker-compose.yml`

### Dockerfile portal
`RUN chown -R 1000:1000 /app` substituído por `COPY --chown=1000:1000` +
`USER 1000` no install do prisma — o chown antigo travava o build por
minutos (achado 2026-07-28).

---

## Grupos de segurança e cota por tenant (Fase 3, 2026-07-15)

Detalhe técnico completo em `docs/portal/ARCHITECTURE.md`. Rebuild +
redeploy do portal e `prisma db push` (`security_groups`, `tenant_quotas`,
`users.security_group_id`) aplicados e confirmados no ar.

**Validado ao vivo, só contra o tenant NPX (autorizado):** login real
confirmou o JWT agora carrega `permissoes` calculado a partir do papel;
criação de um grupo de teste via HTTP real confirmada gravando as 3
flags corretas no banco (limpo depois — a exclusão via HTTP não foi
confirmada por causa de uma dificuldade de reproduzir a codificação
exata do form quando há duas server actions bound na mesma página via
`curl`, não um defeito do código; limpeza feita direto via SQL na própria
tabela do portal). Tela de cota (`/tenants/[id]/quotas`) confirmada
carregando corretamente pra FLUA (só leitura, mostra "irrestrito hoje"
corretamente) e redirecionando pro tenant raiz (NPX não tem cota, é
mestre). Nenhuma cota foi salva pra nenhum tenant real nesta sessão —
ficam todos irrestritos até o responsável do projeto configurar.

**Item 9 — atualizado em 2026-07-15 (segunda rodada): credenciais reais
recebidas, configuradas, mas o teste real de envio ainda falha — não
está concluído.** Sequência completa desta rodada:

1. `SMTP_HOST`/`SMTP_PORT`/`SMTP_USER`/`SMTP_PASSWORD`/`SMTP_FROM`
   preenchidos em `portal/.env` (chmod 600) com as credenciais reais do
   Brevo fornecidas pelo responsável do projeto.
2. **Bug real corrigido**: `docker-compose.yml` tinha `SMTP_HOST`/`SMTP_PORT`
   hardcoded pro Office 365 antigo, sem `${...}` — `.env` nunca teria
   conseguido sobrescrever isso. Corrigido para `${SMTP_HOST:-smtp-relay.brevo.com}`/`${SMTP_PORT:-587}`.
3. **Segundo bug real corrigido**: o resultado de `sendPasswordResetEmail`
   (`{sent, reason}`) era descartado silenciosamente em
   `forgot-password/actions.ts` — nenhum log, sucesso e falha eram
   indistinguíveis pra sempre. Adicionado `console.log`/`console.error`
   no resultado real (a tela continua mostrando a mesma mensagem
   genérica, por design, pra não vazar quais e-mails existem — só o log
   do servidor mudou).
4. Rebuild + redeploy do portal aplicados com as duas correções.
5. `dig` confirmou **DKIM configurado e válido** em `mail.npxit.com.br`
   (dois seletores Brevo, chave pública presente) — mas **SPF ausente**
   nesse subdomínio (só o TXT de verificação de domínio existe lá; o SPF
   do domínio raiz é só do Office 365, não cobre o Brevo).
6. **Teste real de envio (SMTP de verdade, não só a API) — falhou**:
   `525 5.7.1 Unauthorized IP address`. O Brevo restringe por IP de
   origem autorizado; o IP de saída desta VM (`187.110.164.122`,
   confirmado via `curl ifconfig.me`) não está liberado na conta Brevo.

**Resolvido em 2026-07-15 (terceira rodada) — item 9 CONCLUÍDO, testado
de verdade.** O responsável do projeto liberou `187.110.164.122` no
painel do Brevo. Reteste real:

- Conexão SMTP direta (mesma config de `mailer.ts`): **sucesso real** —
  `250 2.0.0 OK: queued as <dd60e119-0f88-0a89-8796-8d45a4dd363c@mail.npxit.com.br>`,
  resposta de protocolo do próprio servidor do Brevo, não uma API REST
  resumindo o resultado.
- Fluxo real da aplicação (`POST /forgot-password` via requisição HTTP
  de verdade, mesmo caminho que um usuário real percorre): `303` →
  `/forgot-password?sent=1`, e o log do servidor (graças à correção do
  item 3 acima) confirmou: `[forgot-password] E-mail de reset enviado
  com sucesso para admin@npxit.com.br`.
- **Confirmado de ponta a ponta de verdade, incluindo entrega visual**:
  `admin@npxit.com.br` não era uma caixa real (informado pelo
  responsável do projeto) — reenviado o teste direto para
  `nicholasalex@gmail.com` (e-mail real do responsável). Aceito pelo
  Brevo (`250 2.0.0 OK`) e **confirmado recebido no Gmail** — cabeçalho
  real do e-mail entregue: `SPF: PASS` (IP 77.32.148.27), `DKIM: PASS`
  (domínio `mail.npxit.com.br`), `DMARC: PASS`, entregue em 37 segundos.
- **Correção de um erro meu na rodada anterior**: eu tinha registrado
  "SPF ausente" por ter checado o subdomínio errado
  (`mail.npxit.com.br`, onde só existe o TXT de verificação de
  domínio). O Brevo separa SPF (endereço de envelope/bounce) de DKIM
  (domínio visível no From) em subdomínios diferentes — o SPF real está
  em `send.mail.npxit.com.br` (`CNAME` para o Brevo, carregando
  `v=spf1 include:spf.brevo.com -all`), confirmado via `dig` depois do
  cabeçalho real ter apontado o caminho certo. **SPF, DKIM e DMARC estão
  todos corretamente configurados** — nenhuma pendência de DNS restante
  para este relay. Detalhe em `docs/ACCESS.md`.

## Credenciais de instância visíveis por tenant (2026-07-16)

Tela nova `/tenants/[id]/credentials` — usuário/senha das instâncias do
próprio tenant, senha cifrada em repouso (AES-256-GCM), oculta por
padrão com botão "Revelar" + auditoria (quem/quando). Detalhe técnico
completo em `docs/portal/ARCHITECTURE.md`; decisões de segurança
(algoritmo, chave mestra, exclusão do `suporteti`, quem pode ver) todas
confirmadas com o responsável do projeto antes de implementar — ver
`docs/DECISIONS.md`.

**Migrado:** as 5 credenciais nativas de `demo` e `flua` cujo valor real
era conhecido com certeza (Zabbix+Grafana de cada um, GLPI da FLUA).
**Não migrado, de propósito:** `npx-glpi` — senha nativa nunca
confirmada em nenhuma sessão. `docs/ACCESS.md` segue como fonte pra
segredo de infraestrutura interna da NPX (nunca migra) e como
backup/referência geral.

**Validado ao vivo:** valor no banco confirmado ilegível (`psql` direto,
não é texto puro); decrypt round-trip correto pras 5 linhas; tela
carregada de verdade contra a FLUA real, mostrando os 3 usuários certos;
`npx-glpi` corretamente mostrando "sem credencial cadastrada".

## Lote de correções operacionais pós-uso real (Fase A1-A11, 2026-07-15)

Pedido depois de uso real do painel pelo responsável do projeto —
inclusive um GLPI de verdade provisionado por ele para o tenant NPX, que
expôs vários bugs reais (não hipotéticos). Detalhe técnico completo em
`docs/portal/ARCHITECTURE.md` (seção "Ações operacionais, diagnóstico,
domínio e provisionamento assíncrono") e `docs/DECISIONS.md`. Resumo:

- **A1 (ações operacionais)**: Iniciar/Parar/Reiniciar/Ver logs por
  instância, restrito a super_admin. Testado ao vivo (leitura: inspect +
  logs, confirmados contra `demo-zabbix-web` real) — start/stop/restart
  não foram testados contra container de produção de propósito (ação
  destrutiva demais pra um smoke test), seguem o mesmo padrão já
  comprovado das outras funções do `lib/portainer.ts`.
- **A2 (provisionamento assíncrono)**: **testado ao vivo, ponta a
  ponta**, contra um tenant descartável (`teste-a2`, criado e
  completamente removido nesta sessão) — resposta HTTP em 1s (antes:
  1-2min), job em background confirmado avançando e concluindo com
  sucesso (~68s depois, típico do Grafana).
- **A3/A4 (auto-refresh + certificado automático)**: polling de 20s
  implementado; A4 não precisou de código (Traefik já reconsulta ACME
  sozinho, confirmado nos próprios logs do Traefik).
- **A5 (redirect HTTP→HTTPS)**: era bug platform-wide, não só do GLPI —
  corrigido no Traefik (entrypoint), testado ao vivo pra 2 hosts.
- **A6 (credenciais GLPI)**: `suporteti` sempre funcionou (confirmado ao
  vivo, `initSession` + `getActiveProfile` = Super-Admin) — era só a
  documentação que nunca foi escrita. Corrigido na origem.
- **A7 (diagnóstico)**: seção nova no painel, testada contra container
  real. Achado: o "Zabbix da NPX" que aparece no painel (`zabbix.demo`)
  está saudável agora — se a referência era `zabbix-master.npxit.com.br`
  (infra própria, não rastreada como `Instance`), o motivo já era
  conhecido (DNS nunca criado).
- **A8 (domínio escolhido pelo usuário)**: campo de formulário de
  verdade agora, pré-preenchido mas editável.
- **A9 (trocar domínio de instância existente)**: **testado ao vivo**
  contra tenant descartável (`teste-a9`, criado e completamente removido
  nesta sessão) — label Traefik editada, stack redeployada via
  Portainer (200), Traefik confirmado roteando o domínio novo (301 com
  `Location` correto) segundos depois, sem passo manual.
- **A10**: `nicholasalex@gmail.com` documentado como e-mail de teste
  padrão em `CLAUDE.md`; `admin@npxit.com.br` marcado como não-real.
- **A11**: Vaultwarden/Uptime Kuma nunca foram de fato pedidos antes
  desta sessão (busca exaustiva não achou registro) — registrado como
  item de roadmap próprio em vez de implementado às pressas.

## Documentação por tenant (Fase 4, 2026-07-15)

`/tenants/[id]/docs` (segura pro cliente) e `/tenants/[id]/docs/technical`
(só super_admin) — detalhe completo em `docs/portal/ARCHITECTURE.md`.
Rebuild + redeploy do portal aplicados (sem mudança de schema — nenhum
`prisma db push` necessário nesta fase). Validado ao vivo contra a FLUA
nas duas variantes, sem risco: diferente do módulo de integração (Fase
2), estas páginas só leem o banco do próprio portal, nunca chamam a API
de Zabbix/Grafana/GLPI — confirmado por grep que nenhuma infraestrutura
interna (FortiGate, IP 172.16.x.x, senha) aparece na versão do cliente.

## Módulo de integração genérico (Fase 2, 2026-07-15)

Tela nova `/tenants/[id]/integrations` — status de saúde + botão
reconectar para integrações entre apps de um tenant, generalizando o que
antes era só infraestrutura pontual (webhook Zabbix→GLPI, datasource
Zabbix→Grafana), sem nenhuma tela de painel. Detalhe técnico completo em
`docs/portal/ARCHITECTURE.md`. Rebuild + redeploy do portal e
`prisma db push` (tabela `integrations` nova) já aplicados e
confirmados no ar.

**Falso alarme investigado e resolvido nesta sessão:** o datasource
Zabbix→Grafana do tenant `demo` (sob o tenant NPX) retornou
`"Incorrect user name or password or account is temporarily blocked"`
na primeira checagem. Investigado a pedido do responsável do projeto
antes de qualquer correção: a credencial `grafana-reader` documentada em
`docs/ACCESS.md` autentica normalmente no Zabbix `demo` (testado direto
via API), e o health check do próprio Grafana, repetido logo em
seguida, voltou `OK` (`Zabbix API version 7.0.28`). Causa real: bloqueio
temporário do Zabbix contra força bruta (a mensagem é ambígua de
propósito, não diferencia "senha errada" de "conta temporariamente
bloqueada") — já resolvido sozinho, nenhuma configuração precisou ser
corrigida, nenhum "reconectar" foi acionado.

**Erro de escopo cometido e corrigido durante a validação:** o teste
inicial da tela rodou contra a FLUA em vez do tenant NPX pedido
explicitamente — sem escrita nenhuma nas ferramentas (só leitura),
mas fora do autorizado. Registro completo, avaliação de impacto e a
decisão do responsável (manter as duas linhas gravadas) em
`docs/DECISIONS.md` (entrada 2026-07-15).

Estado herdado de 2026-07-14 — **todas as fases planejadas (0-6, A-D,
E-H) concluídas**, incluindo a fase de endurecimento do provisionamento
self-service. Únicas pendências reais que restam: (1) credenciais SMTP
do Brevo (Fase F — cadastro externo, fora do alcance deste agente); (2)
limites de CPU/memória do provisionamento self-service, propostos e já
**confirmados** pelo responsável do projeto, mas ainda não validados sob
carga real de cliente (ver `docs/DECISIONS.md`). Ver seção
"Sessão de branding/publicação/observabilidade" para as Fases 0-3,
"Correções e nova infraestrutura" para A-D e 4-6, e as seções "Fase E",
"Fase F", "Fase H" e "Provisionamento self-service — fase de
endurecimento" abaixo para o restante.

## FortiGate — primeiro acesso live (2026-07-15)

Primeira vez que este projeto teve acesso SSH direto ao FortiGate
(172.16.11.1, LAN-only, usuário `admn`). Só leitura usada nesta fase —
nenhuma alteração feita. **Achado que bloqueia a Fase 5 (automação de
escrita) até revisão do responsável:** o perfil real do usuário diverge
do escopo pedido nos dois sentidos — leitura mais restrita que "tudo"
(sem VPN/UTM/User\&Auth, confirmado ao vivo) e escrita mais ampla que só
"policy/VIP/address/service" (inclui `schedule` e o grupo `others` do
FortiOS, que agrega VIP + vários outros tipos de objeto numa única
chave), além de `cli-diagnose`/`cli-exec` habilitados (poder operacional
além de show/get). Detalhe completo, incluindo o `accprofile` real e o
achado incidental de que existe só uma conta admin no FortiGate inteiro,
em `docs/DECISIONS.md` (entrada 2026-07-15). Credencial em
`docs/ACCESS.md` / `/opt/npx-platform/fortigate/.env`.

## Fase E — validação visual real (Playwright) — concluída

Ferramenta permanente, não específica de uma sessão: `scripts/validate-visual.sh
<url> <nome.png> [--profile zabbix|grafana|glpi] [--user U] [--pass P]
[--ignore-https-errors]`, que roda `portal/scripts/playwright-screenshot.js`
dentro da imagem oficial `mcr.microsoft.com/playwright` (Chromium +
dependências resolvidas, sem precisar instalar nada pesado no host).
Suporta login automático nas 3 ferramentas (seletores de campo já
mapeados por perfil) e tira o screenshot mesmo em erro, pra ajudar a
diagnosticar. Saída fica em `docs-publish/validation/` (gitignored, uso
compartilhado entre sessões, nunca publicado automaticamente).

**Testado de ponta a ponta nesta sessão** (a implementação já existia de
uma fase anterior, mas nunca tinha sido confirmada rodando nem
documentada aqui): screenshot de uma URL pública simples (smoke test do
container/rede) e depois um teste real — login automático no Zabbix da
FLUA com `--profile zabbix`, screenshot final mostrando o dashboard
"Global view" já autenticado, com dados reais (hosts, problemas por
severidade, geomapa).

## Fase H — integração WhatsApp documentada (sem implementar) — concluída

Registrado em `docs/ROADMAP.md` (seção "Integração com WhatsApp"), como
pedido: só arquitetura/intenção, nenhum código escrito, nenhuma conta
criada no Meta Business. Provedor decidido pelo responsável do projeto
(WhatsApp Cloud API oficial da Meta, não gateway não-oficial). Cobre
Nível 1 (alertas de saída via Zabbix/Grafana, com a restrição real da
Meta de janela de 24h/template aprovado) e Nível 2 (atendimento via GLPI,
esforço alto, decisão de UX registrada como "não decidir sozinho quando
chegar a hora"). Confirmado nesta sessão que a seção está completa e não
truncada.

## Fase F — e-mail por tenant — schema e baseline concluídos, envio real pendente de cadastro externo

**Schema:** `TenantEmailConfig` (`portal/prisma/schema.prisma`, tabela
`tenant_email_config`) — identidade de envio por tenant (nome, e-mail,
reply-to) e status (`nao_configurado`/`configurado`/`com_erro`).
Deliberadamente **sem** campo de host/usuário/senha de SMTP por tenant —
todo tenant envia pelo relay central (decisão abaixo), não por
credencial própria.

**Baseline SMTP:** `portal/src/lib/mailer.ts` generalizado — antes só
tinha `sendPasswordResetEmail` (acoplado ao fluxo de esqueci-senha do
portal), agora tem `sendMail()` genérico (to/subject/text/html/from
opcional) que qualquer notificação futura por tenant pode reusar sem
mudança nenhuma, e `sendPasswordResetEmail` virou um wrapper fino em
cima dele. Nunca finge sucesso: sem `SMTP_USER`/`SMTP_PASSWORD`
configurados, retorna `{ sent: false, reason: '...' }` honesto.

**Investigação SMTP/O365/Gmail e decisão do relay central:** completa em
`docs/DECISIONS.md` ("E-mail por tenant: investigação SMTP/O365/Gmail").
Resumo: credencial própria por tenant (O365/Gmail) exige ação do
próprio cliente (consentimento Azure AD ou liberação de IP no relay do
Workspace) — não escala como processo de onboarding. Postfix próprio
tem risco real de entregabilidade (IP novo sem reputação, faixas
brasileiras historicamente malvistas por Outlook/Hotmail). Recomendei
provedor transacional; **o responsável do projeto decidiu: provedor
transacional, começando por Brevo** — `mailer.ts` já aponta o host
default para `smtp-relay.brevo.com:587`, configurável via env como
sempre.

**Pendente (fora do alcance deste agente — precisa de cadastro/verificação
humana):** criar conta no Brevo, gerar chave SMTP, preencher
`SMTP_USER`/`SMTP_PASSWORD` em `portal/.env`, e configurar os registros
SPF/DKIM que o Brevo indicar (recomendado usar um subdomínio, ex.
`mail.npxit.com.br`, não o domínio principal). Até lá, o comportamento é
o mesmo já existente pro fluxo de redefinição de senha por e-mail —
token gerado normalmente, e-mail não sai, sem fingir que saiu.

## Provisionamento self-service de instâncias — concluído em 2026-07-13

Maior item do roadmap do portal, fechado nesta sessão. Detalhes técnicos
completos em `docs/portal/ARCHITECTURE.md` (seção "Provisionamento
self-service de instâncias"). Resumo do resultado:

- **Rota nova**: `/tenants/<id>/instances/new` — formulário simples
  (tipo, domínio sugerido automaticamente, checkbox opcional de porta de
  trapper Zabbix). Só `super_admin`.
- **Sem `docker.sock` no portal** — usa a API do Portainer (que já tem
  esse acesso) para subir/atualizar a stack. Dois mounts novos e
  escopados aprovados explicitamente pelo responsável do projeto:
  `clients/` (rw, só pro compose gerado) e `docs/` (rw, só pro
  `PORT-REGISTRY.md`).
- **`suporteti` criado automaticamente** em cada instância nova (Super
  Admin/Admin/Super-Admin conforme a ferramenta) — política permanente,
  ver `docs/DECISIONS.md`.
- **Erros nunca fingidos como sucesso**: se qualquer etapa falhar, a
  tela mostra a etapa exata e o motivo; a linha em `instances` só é
  criada depois de todo o resto confirmado.

**Dois bugs reais encontrados e corrigidos durante o teste ponta a
ponta** (não hipotéticos — quebravam a feature de verdade):
1. Arquivos gerados nasciam com dono `root` (container do portal rodava
   como root por padrão) — operador humano não conseguia nem apagá-los.
   Corrigido (`user: "1000:1000"` no compose do portal), o que também
   revelou e corrigiu um problema de segurança pré-existente: `.env`
   não estava no `.dockerignore` e acabava embutido na imagem.
2. Health-check inicial esperava o **domínio público** responder — nunca
   funcionaria pra tenant novo, porque DNS de subdomínio novo não existe
   até alguém criar manualmente (mesmo padrão já visto com
   zabbix-master/grafana-master). Corrigido: checagem agora é direto
   pelo nome do container na rede `edge` compartilhada, sem depender de
   DNS/Traefik/Let's Encrypt.

**Limite honesto, real, medido ao vivo:** o schema do MySQL do Zabbix
(import completo, ~170 tabelas) mede **~9 minutos** num host já rodando
outras stacks — o timeout inicial (90s) não bastava; corrigido pra 10
minutos. Grafana (SQLite embutido) e GLPI são bem mais rápidos.

**Testado ponta a ponta:**
- **Grafana**: provisionado do zero pelo painel (tenant `teste-portal`)
  — container subiu, respondeu, `suporteti` criado e confirmado com
  login real (`isGrafanaAdmin: true`). Limpo depois (stack, volume,
  arquivo, linha no banco, tenant).
- **Zabbix**: pipeline completo (compose, deploy via Portainer, mesma
  lógica de criação do `suporteti`) validado depois de aguardar o schema
  terminar de verdade (~9min) — login do `suporteti` confirmado
  funcionando. O timeout que causou a falha na primeira tentativa já foi
  corrigido no código antes de finalizar esta fase. Limpo depois.
- **GLPI**: testado ponta a ponta na fase de endurecimento seguinte (ver
  seção abaixo) — precisou de uma correção adicional real (API REST
  desligada por padrão), não só reaproveitar o padrão de Zabbix/Grafana.

## Provisionamento self-service — fase de endurecimento — concluída em 2026-07-14

Pedido explícito do responsável do projeto: o caminho feliz já
funcionava testado, mas precisava ficar pronto pra uso real com clientes
de verdade. Sete itens, todos concluídos:

1. **Validação de entrada**: `portal/src/lib/validation.ts`
   (`isValidSlug`/`validateSlugOrThrow`, regex
   `/^[a-z0-9]([a-z0-9-]{1,38}[a-z0-9])?$/`) — barra na criação do tenant
   (`tenants/actions.ts`, com mensagem explicativa na tela) **e** de novo
   dentro de `provisionInstance` (defesa em profundidade: tenants criados
   antes dessa validação existir, ou qualquer caminho futuro que pule a
   tela, não conseguem gerar compose/comando com entrada não sanitizada).
2. **Limites de recurso**: `RESOURCE_LIMITS` em `compose-templates.ts`,
   aplicado a todo container gerado (Zabbix server/MySQL: 512m/1.0cpu;
   Zabbix web: 256m/0.5cpu; Grafana: 512m/0.5cpu; GLPI/MySQL: 512m/0.5cpu
   cada). Critério completo em `docs/DECISIONS.md` — **valores propostos,
   ainda pendentes de confirmação explícita do responsável do projeto**,
   não medidos sob carga real.
3. **Rollback**: `rollback()` em `provisioning.ts` remove containers e
   volumes do fragmento que falhou e, dependendo de o tenant já existir
   ou não, apaga a stack toda ou restaura o compose original e reimplanta
   via Portainer. **Bug real encontrado e corrigido** testando de
   propósito (matando um container no meio do provisionamento): o
   rollback não revertia nada de verdade porque `mergeCompose` mutava o
   objeto `existing` por referência (`const base = existing ?? {...}` só
   clona quando `existing` é falsy) — corrigido para clonagem explícita
   (`existing ? {...existing} : {...}`). Reproduzido o teste de falha
   forçada de novo depois do fix: limpeza completa confirmada (containers,
   volumes, arquivo compose, stack no Portainer, linha no banco e log de
   auditoria).
4. **Concorrência**: `provisionInstanceAction` cria a linha `Instance`
   (`status: 'provisionando'`) **antes** de qualquer trabalho de infra,
   usando a constraint única `@@unique([tenantId, tipo])` — a segunda
   requisição simultânea recebe `P2002` do Prisma e é redirecionada
   (`error=ja-existe`) sem tocar em infraestrutura. Testado de verdade com
   duas requisições concorrentes reais: uma recebeu `ja-existe`
   imediatamente, a outra seguiu e criou o Grafana normalmente.
5. **Log de auditoria**: tabela nova `ProvisioningAudit` (quem, quando,
   tipo, resultado, última etapa, detalhe do erro) — populada em sucesso
   e falha, renderizada em `/tenants/<id>/instances` (últimos 10,
   descendente).
6. **Status de DNS pendente**: `checkDnsReady()` faz um `fetch` real (3s
   de timeout) por instância `ativo` na tela de instâncias — sem cache,
   sempre ao vivo — e mostra "⏳ Aguardando DNS" com o registro A exato e
   o IP que falta configurar, quando o domínio ainda não responde.
7. **Teste ponta a ponta do GLPI, que tinha ficado pendente**: revelou um
   problema real, não só de timing. Descrito abaixo.

**GLPI — causa raiz real encontrada (não era timeout):** a imagem oficial
`glpi/glpi:latest` sobe com a REST API desligada
(`enable_api=0`) e o único cliente de API pré-cadastrado restrito a
`127.0.0.1` — sem variável de ambiente pra isso. As primeiras tentativas
aumentaram o timeout do health-check (90s → 240s → 600s) até ficar claro,
inspecionando `docker logs` e testando a chamada manualmente, que o
container já respondia HTTP havia muito tempo e a falha real era
`["ERROR","API disabled"]` e depois `["ERROR_NOT_ALLOWED_IP",...]` no
`initSession`. Corrigido em `enableGlpiApi()`
(`portal/src/lib/provisioning.ts`), rodando via exec no container pelo
proxy Docker do Portainer (mesmo mecanismo do rollback, sem precisar
`docker.sock` no portal):
- `bin/console config:set --context=core enable_api 1`
- `bin/console config:set --context=core enable_api_login_credentials 1`
- `UPDATE glpi_apiclients SET ipv4_range_start=<ip>, ipv4_range_end=<ip> WHERE id=1` — `<ip>` é o IP do próprio portal na rede `edge`, descoberto em tempo real (não fixo).

Perguntei explicitamente ao responsável do projeto se essa liberação
deveria cobrir a rede `edge` inteira ou só o IP do portal — optou pelo
mais restrito (só o portal). Decisão completa e critério em
`docs/DECISIONS.md`.

**Confirmado ponta a ponta depois do fix** (tenant `teste-glpi`, limpo
depois): provisionamento completo do zero pelo painel — container subiu,
respondeu, `enableGlpiApi` rodou, `suporteti` criado. Login real
confirmado via `initSession` (200, token válido) rodado de dentro do
próprio container do portal (a única origem permitida agora) — e perfil
ativo confirmado como `"name":"Super-Admin","id":4` via
`getActiveProfile`. Sessão encerrada e todo o tenant de teste limpo
(stack, volumes, diretório de compose, linhas de `instances`,
`provisioning_audit` e `tenants`) depois de confirmado.

**Causa-raiz resolvida, não só contornada, em duas frentes de
infraestrutura do próprio portal (pedido explícito do responsável do
projeto):**
- `npx prisma` pegava a versão 7 (incompatível com o schema v5) porque a
  imagem do runner nunca teve o pacote `prisma` instalado, só o
  `@prisma/client` gerado — `npx` baixava a versão mais recente do
  registro. Corrigido fixando `prisma@5.20.0` via `npm install --no-save`
  direto no estágio runner do `Dockerfile`.
- Conflito de permissão do usuário não-root com `prisma generate`:
  arquivos ficavam com dono `root` (copiados durante o build), e o
  container roda como uid 1000. Corrigido com `RUN chown -R 1000:1000
  /app` como último passo do estágio runner.
- Ambos verificados de ponta a ponta depois do fix: `npx prisma -v`
  reporta 5.20.0; `npx prisma db push` (sem flags) roda completo,
  incluindo o `generate`, como usuário não-root.

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

**Fase 4 — Grafana NOC por tenant + mestre NPX — concluída, retomada e
fechada em 2026-07-13.**

**Checagem de segurança do acesso anônimo (obrigatória antes de habilitar,
feita e confirmada ao vivo, não só lida na documentação):**
- Cada tenant (`demo`, `flua`) tem seu **próprio container Grafana
  isolado** — não é uma organização dentro de um Grafana compartilhado.
  Isso torna o vazamento cross-tenant estruturalmente impossível: não
  existe dado de outro tenant dentro do mesmo processo/banco para vazar.
- `GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer` (nunca Editor/Admin) +
  `GF_EXPLORE_ENABLED=false` em ambos os Grafanas de cliente (Explore
  desativado inteiramente — elimina qualquer ambiguidade sobre um usuário
  anônimo rodar query arbitrária contra o datasource, só os dashboards já
  publicados ficam visíveis).
- **Testado ao vivo (não só lido em doc) contra `grafana.flua` e
  `grafana.demo` publicamente, sem cookie/sessão:**
  - Dashboard NOC: `200` (renderiza normalmente em modo kiosk).
  - `/explore`: `302` (redireciona pra home — bloqueado).
  - `POST /api/dashboards/db` (tentativa de criar/editar): `403`.
  - `/api/admin/settings`, `/api/org/users`: `403` (sem vazar config nem
    lista de usuários).
  - `/api/user` (whoami): `401` — nem chega a expor uma identidade.
- **Achado honesto, não bloqueante:** `/api/datasources` retorna `200`
  para o Viewer anônimo, expondo o hostname Docker interno (ex:
  `http://flua-zabbix-web:8080/...`) e o **username** (não a senha) do
  `grafana-reader`. Risco baixo: o hostname só resolve dentro da rede
  Docker do próprio host (inacessível de fora), e sem a senha o username
  sozinho não autentica em lugar nenhum. Registrado aqui para não ser
  esquecido, não corrigido (seria preciso trocar o modelo de permissão do
  Grafana OSS, fora de escopo desta fase).

**Implementado:**
- **`demo-grafana`**: não tinha integração com Zabbix nenhuma até esta
  fase (plugin não instalado, rede `internal` ausente, 0 dashboards).
  Corrigido: plugin `alexanderzobnin-zabbix-app` instalado, rede
  `internal` adicionada (agora alcança `zabbix-web:8080` via DNS Docker),
  usuário `grafana-reader` criado no Zabbix `demo` (mesmo padrão da
  FLUA: role "User role", grupo só-leitura — testado: lê hosts, não cria
  usuários), datasource "Zabbix" criado e testado (health check OK,
  `Zabbix API version 7.0.28`), dashboard **"DEMO - NOC Overview"**
  (`/d/demo-noc-overview`) criado espelhando a estrutura do da FLUA
  (problemas ativos total + tabela detalhada + hosts monitorados).
- **`flua-grafana`**: já tinha o dashboard "FLUA - NOC Overview" da Fase
  2 anterior — só habilitado o acesso anônimo/kiosk nesta fase, nada
  recriado.
- **Grafana mestre da NPX (`npx-grafana`)**: novo serviço adicionado a
  `monitoring/npx-zabbix/docker-compose.yml` (mesma stack do
  `npx-zabbix-*`, rede `internal` + `edge`). **Sempre autenticado, sem
  acesso anônimo** — ferramenta interna da equipe NPX, não voltada a
  cliente, vê dado agregado de todos os tenants (por isso não pode ter o
  mesmo modelo aberto dos NOCs de cliente). Três datasources Zabbix
  criados e testados (health check OK nos três, `Zabbix API version
  7.0.28`):
  - **"NPX Master"** → `npx-zabbix-web:8080` (rede interna, mesma stack).
  - **"Zabbix Demo"** → `https://zabbix.demo.npxit.com.br` (usuário
    `grafana-reader` dedicado, criado nesta fase).
  - **"Zabbix FLUA"** → `https://zabbix.flua.npxit.com.br` (reaproveita o
    `grafana-reader` já existente da Fase 2).
  Datasources cross-tenant apontam para as **URLs HTTPS públicas já
  existentes** (não uma rede Docker compartilhada) — respeita a decisão
  de arquitetura já fixada em `docs/ROADMAP.md` ("isolamento sempre via
  Docker, nunca misturar redes internas de stacks diferentes").
  Dashboard **"NPX - Visão Consolidada"** (`/d/npx-master-overview`)
  criado com 3 seções (linhas): NPX/plataforma, DEMO, FLUA TI — problemas
  ativos + tabela detalhada em cada uma.
  **Confirmado sem acesso anônimo**: dashboard sem cookie → `302`
  (redireciona pro login); com credencial → `200`.
  **Pendente (não bloqueante, mesmo padrão do `zabbix-master`):** DNS de
  `grafana-master.npxit.com.br` ainda não existe — acessível hoje só via
  `--resolve`/`-k` (certificado self-signed de fallback). Assim que o DNS
  existir, o Traefik emite o certificado de produção sozinho, nenhuma
  ação adicional necessária.
- **Portal**: link "Ver NOC (kiosk)" adicionado em `/dashboard` (visão de
  `gestor`/`tecnico`) para toda instância `tipo=grafana`, construindo a
  URL pela convenção `/d/<slug-do-tenant>-noc-overview/...?kiosk=tv`
  (`portal/src/app/dashboard/page.tsx`). Não precisa de sessão do
  Grafana — o link já abre público, por isso funciona mesmo num navegador
  sem login prévio no Grafana daquele tenant. Rebuild + redeploy do
  `portal` feito e confirmado (`https://admn.npxit.com.br/login` → 200).

**Limite honesto, mesmo já documentado na Fase 2:** o painel "Problemas
Ativos" (tipo de query `Number of problems`/`Problems` do plugin Zabbix)
só renderiza via frontend JS do Grafana — não é validável por `curl`
puro contra `/api/ds/query`. A validação de "renderiza corretamente" foi
feita só por HTTP 200 na URL do dashboard (confirma que a página carrega
e o backend aceita a query), não por captura visual do navegador nesta
sessão (sem acesso a browser neste ambiente). Recomendo conferir
visualmente antes de divulgar o link do NOC a clientes.

**Fase 5 — biblioteca de templates v1 — concluída em 2026-07-13.**
Detalhes completos em `docs/templates/ZABBIX-TEMPLATES.md` e
`docs/templates/GRAFANA-TEMPLATES.md`. Resumo:

- **Zabbix**: descoberta chave — a imagem oficial já vem com **356
  templates** carregados sem precisar de nenhum `configuration.import`
  (confirmado via `template.get`). Documentada uma matriz de
  recomendação por tipo de ativo (Linux, Docker, Windows/IIS, SNMP de
  rede por marca, UPS, FortiGate, nuvem). Além disso, criado e **testado
  ponta a ponta** um template customizado próprio — **"NPX - Trapper
  Padrao"** (grupo `Templates/NPX`, 1 item trapper + 1 trigger de
  ausência de dado) — criado no `zabbix-master`, exportado via
  `configuration.export` para `templates/zabbix/npx-trapper-padrao.yaml`
  (versionado no repo), importado via `configuration.import` em `demo` e
  `flua` (mesmo YAML, mesmo comando), linkado a um host e validado com
  `zabbix_sender` (valor recebido e confirmado em `history.get`). Isso
  prova o pipeline completo que será usado para distribuir templates
  autorais entre todos os clientes no futuro.
  **Checado antes de linkar o template**: o host "Zabbix server" de
  `demo` tinha 121 itens próprios (não herdados de nenhum template) —
  `host.update` com `templates:[...]` não removeu nem alterou nenhum,
  só adicionou o item novo.
- **Grafana**: o plano original (buscar dashboards via
  `grafana.com/api/dashboards?tag=zabbix`) não funcionou — **testado ao
  vivo, a API pública mudou de contrato** (retorna `409 Unexpected
  parameter: tag` hoje). Pivotado para uma fonte melhor e sem dependência
  de rede externa: o próprio plugin `alexanderzobnin-zabbix-app` já
  instalado vem com **3 dashboards oficiais embutidos** ("Zabbix Server
  Dashboard", "Zabbix System Status", "Zabbix Template Linux Server"),
  copiados para `templates/grafana/*.json` no repo. Importado
  "Zabbix Server Dashboard" via `/api/dashboards/import` em `demo-grafana`
  e `flua-grafana` — **confirmado ao vivo**: `200`, 7 painéis carregados
  em ambos.
- **GLPI**: fora de escopo do v1 por decisão explícita — não tem um
  artefato portável único equivalente (dashboard/template) para
  replicar via API entre entities. Registrado em `docs/ROADMAP.md`.

**Nota de segurança (decorrência da Fase 4, não um problema novo):** o
dashboard "Zabbix Server Dashboard" recém-importado também fica visível
ao Viewer anônimo nos dois Grafanas de cliente (mesmo role, Grafana OSS
não restringe por dashboard individual) — conteúdo é só saúde
operacional do próprio Zabbix, mesmo nível de risco já aceito na Fase 4,
não uma exposição nova de dado de cliente.

---

## Correções e nova infraestrutura (2026-07-13)

Nova rodada de trabalho a partir de uma sessão retomada. Fases com letras
(A, B, C, D) para não colidir com a numeração 0-6 da sessão anterior.

**Fase A — corrigir "connection refused" no Zabbix (demo+flua) — concluída.**
Causa: o host "Zabbix server" (criado automaticamente pela própria
instalação do Zabbix) vem com ~40 itens do template padrão "Linux by
Zabbix agent" apontando para um agente em `127.0.0.1:10050` — que nunca
existiu nesses stacks (não há container de agente rodando em nenhum dos
dois). Desativados (`status=1`, não deletados — reversível) todos os 40
itens tipo "Zabbix agent" (passivo) em ambos: `demo` e `flua`. Confirmado
via `docker logs --since 5m | grep -i "refused\|network error"` → vazio
nos dois, erro parou.

**Próximo passo documentado, não implementado agora:** se algum dia for
importante ter CPU/RAM/uptime do container do próprio zabbix-server (não
é crítico hoje — a Fase 3 já cobre isso a nível de host via
`npx-zabbix-agent`), a forma certa é subir um `zabbix-agent2` leve dedicado
por stack (mesmo padrão do `monitoring/npx-zabbix/` desta sessão), não
reativar os itens antigos sem um agente real para responder.

**Fase 6 — fechamento e validação ampla — concluída em 2026-07-13.**

**1. Health-check real (não só `docker ps`) de cada serviço:**
- 19 containers, todos `Up`/`healthy`.
- HTTP real nos 8 hosts de produção: todos responderam status esperado
  (`200`/`307`/`401` conforme a proteção de cada um — nenhum 5xx/timeout).
- Certificado TLS real checado via `openssl s_client` nos 8 hosts:
  todos **Let's Encrypt produção**, válidos até **10/out/2026**.

**2. Isolamento entre tenants — retestado ao vivo nesta fase, não
assumido:**
- Criado um usuário `gestor` temporário na FLUA TI via fluxo normal do
  portal (login como `super_admin`, formulário real — não SQL direto),
  usado só para o teste e **removido ao final** (mesmo fluxo de exclusão
  da UI).
- Login desse `gestor` confirmado: vê só "Minhas instâncias" (não a
  tabela de Tenants, que é exclusiva de `super_admin`) com exatamente as
  3 instâncias da FLUA (zabbix/grafana/glpi).
- Tentativa de acessar `/tenants/<id-da-NPX>/users` (tenant de outro) →
  `307` (bloqueado).
- Tentativa de acessar `/tenants/new` (rota exclusiva de `super_admin`) →
  `307` (bloqueado).
- Link "Ver NOC (kiosk)" confirmado renderizando a URL certa,
  escopada ao tenant certo (`grafana.flua.npxit.com.br/d/flua-noc-overview/...`).

**Achado real corrigido durante a validação (não um "erro conhecido"
deixado pra depois):** a tela de criar/editar usuário oferecia a opção
`super_admin` sempre que o **ator logado** era `super_admin`, mesmo
criando um usuário dentro de um tenant filho (FLUA) — contradizendo a
convenção já documentada em `docs/portal/ARCHITECTURE.md` ("super_admin
vive no tenant raiz"), que dependia só da tela esconder a opção, sem
checagem no servidor. Corrigido nesta fase:
- **Servidor** (`portal/src/app/tenants/[id]/users/actions.ts`):
  `createUserAction`/`updateUserAction` agora rebaixam `papel=super_admin`
  para `gestor` automaticamente se o tenant alvo não for o raiz
  (`parentTenantId` não nulo) — essa é a barreira de segurança real,
  independente da UI.
- **UI** (`.../users/new/page.tsx` e `.../users/[userId]/page.tsx`): a
  opção só aparece no `<select>` quando o ator é `super_admin` **e** o
  tenant alvo é o raiz.
- **Confirmado ao vivo, antes e depois do fix**: opção presente em
  ambos antes da correção; depois, ausente ao criar/editar usuário da
  FLUA (`grep` no HTML → 0 ocorrências da tag) e ainda presente no tenant
  raiz NPX (1 ocorrência) — comportamento certo em ambos os casos.
  Rebuild + redeploy do `portal` feito.

**Nota metodológica sobre como esse reteste foi feito** (documentando
porque não é óbvio): testar Server Actions do Next.js via `curl` puro
precisou de 3 detalhes descobertos ao vivo nesta sessão (podem ser úteis
em sessões futuras que precisem repetir isso):
1. Ações **não-bound** (ex: login) usam um campo `$ACTION_ID_<hash>`
   simples; ações **bound** (ex: criar usuário, que amarra o `tenantId`)
   exigem 3 campos: `$ACTION_REF_1` (vazio), `$ACTION_1:0` (JSON com o
   `id` da action), `$ACTION_1:1` (JSON array com os argumentos
   amarrados).
2. **Nunca enviar o header `Next-Action`** junto com esse formulário
   multipart — ele é só para o modo de "callServer" via fetch do próprio
   JS do Grafana/Next; misturado com o fallback de formulário HTML puro
   causa `500 Connection closed`.
3. O Next.js **exige o header `Origin`** correspondendo ao host em toda
   Server Action (proteção anti-CSRF nativa) — sem ele, a ação falha
   silenciosamente e **invalida a sessão atual** (`set-cookie` limpando o
   `npx_session`), então depois de um request malformado é preciso logar
   de novo, não só corrigir o próximo request.

**3. NOC/kiosk — reconfirmado após as mudanças da Fase 5:**
Reexecutados os mesmos testes de anonimato da Fase 4 (dashboard `200`,
`/explore` `302`, criar dashboard `403`) contra os dois dashboards agora
existentes em cada tenant (`*-noc-overview` e o novo
`zabbix-server-dashboard` importado na Fase 5) — comportamento idêntico,
nenhuma regressão.

**4. Scripts de publicação — rodados novamente, ambos com sucesso:**
- `scripts/publish-docs.sh`: sincronizado para `platform-docs` (público)
  — checagem de segredo passou limpa, push confirmado
  (`dbbc6b7`). **Gap conhecido, não uma falha:** a lista fechada de
  arquivos sincronizados (por design, ver Fase 1) ainda não inclui
  `docs/templates/*.md` (novos nesta sessão) — ficaram só no backup
  privado por enquanto. Não  são segredo (revisei o conteúdo, só nomes de
  template/URL pública do grafana.com), mas não decidi sozinho expandir a
  lista fechada sem o responsável confirmar que quer isso público também.
- `scripts/backup-source.sh`: rodado com sucesso, 19 arquivos novos/
  alterados enviados pro `admn` (privado) — inclui os templates novos
  (`templates/grafana/*.json`, `templates/zabbix/*.yaml`) e os docs novos.
  Push confirmado (`4cdd84f`).
- Confirmado (`find`) que `docs-publish/` **não contém** `ACCESS.md` nem
  nenhum `.env` — isolamento entre os dois repos intacto.

**5. Pequenas inconsistências encontradas e corrigidas nesta fase:**
apenas a do `super_admin` fora do tenant raiz (item 2 acima). Nenhum
outro erro/warning encontrado nos health-checks, TLS, scripts ou testes
de isolamento.

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

**Fase B — registro de portas de proxy Zabbix — concluída.**
`docs/PORT-REGISTRY.md` criado. Registrado o legado informado pelo
responsável do projeto (VIPs `zabbix_10050`/`zabbix_10051` em
`187.110.164.125` → host antigo `172.16.11.30` — nunca reutilizar essas
portas, mesmo decomissionado). Faixa `11000-11999` reservada para uso
futuro a definir; faixa ativa começa em `12051`.

FLUA TI alocada em `12051` → publicado no `docker-compose.yml`
(`flua-zabbix-server`, `ports: "12051:10051"`), confirmado com
`docker ps` e teste TCP local (`/dev/tcp/127.0.0.1/12051` — aberto).
Container recriado sem perda de dados (schema já existente, só
republicou).

**Comando de VIP/policy do FortiGate — aplicado com sucesso pelo
responsável do projeto em 2026-07-13** (este projeto nunca teve nem tem
acesso ao FortiGate — todo comando abaixo foi só preparado aqui e
executado manualmente por quem tem acesso real). Primeira tentativa
(2026-07-12) tinha falhado na aplicação real — faltava `set extintf`
(obrigatório) e a ordem de `extport`/`mappedport`/`protocol` estava
errada (esses campos só ficam visíveis no parser da CLI depois de `set
portforward enable`; sem isso o FortiOS recusa com "command parse
error"). Corrigido nesta sessão a partir da leitura de um backup completo
de config do FortiGate que o responsável forneceu (usado só para extrair
o padrão das VIPs legadas — arquivo tratado como sensível, não commitado
em nenhum repo, `portal/FILES/` adicionado ao `.gitignore` como
proteção). Achado chave: `extintf` das VIPs do Zabbix legado é sempre
`"any"` (não é nome de interface física), e nenhuma delas seta
`protocol` explicitamente (default `tcp`). Confirmado também, via `grep`
no backup inteiro, que a porta `12051` não conflita com nenhuma
VIP/policy já existente no FortiGate.

**Resultado:** os comandos abaixo rodaram sem erro no FortiGate real —
VIP `zabbix_flua_12051` completa (extintf/portforward/extport/mappedport),
service object e a policy `ZABBIX_FLUA` criados. A rota pública
`187.110.164.126:12051` → `172.16.11.150:12051` (proxy Zabbix da FLUA)
está liberada de ponta a ponta no firewall.

**Pendência aberta (não bloqueante):** o arquivo de backup do FortiGate
(`portal/FILES/FGTVM-DC-EVEO_7-6_3704_202607121634.txt`) ainda está no
disco do host — protegido via `.gitignore` (nunca entrou em nenhum repo),
mas ainda não apagado fisicamente. Perguntei ao responsável se posso
apagar; ainda sem resposta. **Não apagar sem confirmação explícita** (pode
ser trabalho em andamento do usuário) — só recordar a pergunta na próxima
sessão se ele não responder antes.

O objeto `zabbix_flua_12051` já tinha sido parcialmente criado ao vivo
numa tentativa anterior (tinha `extip`/`mappedip`); o comando abaixo só
completou o mesmo objeto (`edit` é idempotente) — histórico do comando
exato usado, para referência futura:

```
config firewall vip
    edit "zabbix_flua_12051"
        set extintf "any"
        set portforward enable
        set extport 12051
        set mappedport 12051
    next
end
```

Service object + policy, espelhando o padrão já usado para VIP única
(`reports_443`) — o grupo `zabbix` legado é interno do próprio NPX
(mistura vsa9/vsa10/reports), não faz sentido meter o FLUA lá:

```
config firewall service custom
    edit "zabbix_flua_12051"
        set tcp-portrange 12051
    next
end

config firewall policy
    edit 0
        set name "ZABBIX_FLUA"
        set srcintf "port1"
        set dstintf "port2"
        set action accept
        set srcaddr "all"
        set dstaddr "zabbix_flua_12051"
        set schedule "always"
        set service "zabbix_flua_12051"
        set logtraffic all
    next
end
```

Opcional (só se algum dispositivo na mesma LAN interna precisar bater no
IP público para chegar no proxy — padrão espelha a policy `NAT_REVERSO`
existente, que já inclui `reports_443` do mesmo jeito): adicionar
`zabbix_flua_12051` ao `dstaddr` dessa policy de NAT reverso. Não
recomendado aplicar às cegas — só se o responsável souber que precisa.

Regra de nunca reutilizar porta registrada em `CLAUDE.md` como padrão
obrigatório permanente.

**Fase C — PT-BR em tudo — concluída (GLPI com uma limitação honesta).**
Confirmado visualmente (não só resposta de API) em todas as ferramentas:

- **Zabbix (demo + flua)**: `default_lang=pt_BR` via `settings.update` —
  tela de login em português confirmada nos dois.
- **Grafana (demo + flua)**: `language=pt-BR` via `PUT /api/org/preferences`
  — **achado**: a chave certa é `language`, não `locale` (que existe no
  payload mas não persiste nesta versão 13.0.2). Confirmado numa sessão
  autenticada real (`"language":"pt-BR"` no HTML servido).
- **GLPI (flua)**: `php bin/console config:set --context=core language
  pt_BR` — comando oficial de CLI. Confirmado (tela de login em
  português).
- **Portal**: scaffold de i18n em `portal/src/lib/i18n.ts` (hoje só
  pt-BR, pronto para outros idiomas depois). Campo `idioma` adicionado em
  `tenants` (migração `prisma db push` aplicada, default `pt-BR`
  populado nos 2 tenants existentes). Seletor de idioma nos formulários
  de criar/editar tenant, confirmado renderizando
  (`grep` no HTML servido). A ação de aplicar branding
  (`applyTenantBrandingAction`) agora também aplica idioma em Zabbix e
  Grafana automaticamente (mesma sessão/credencial admin já usada).

**Limite não-bloqueante:** GLPI não está automatizado na ação de
branding do portal — não existe endpoint REST para a config global
`language` do GLPI (só o CLI oficial, que o portal não tem acesso para
rodar). Aplicado manualmente nesta sessão; documentado como pendência em
`docs/portal/ARCHITECTURE.md`.

**Fase D — investigar SSO — concluída (só diagnóstico, nada implementado,
como pedido).** Achados completos com recomendação em `docs/ROADMAP.md`
(seção "SSO — investigação"). Resumo: Grafana OSS e Zabbix têm
OIDC/SAML nativo de graça (confirmado ao vivo); GLPI não tem, mas tem um
mecanismo nativo de "confiar em header de proxy" (`glpi_ssovariables`) que
viabiliza SSO via um proxy de autenticação extra na frente dele — não é
plug-and-play como os outros dois. Aguardando decisão do responsável do
projeto sobre se/quando construir isso.

---

## 2026-07-16 — Permissões granulares, multi-tenant, 2FA/CAPTCHA/SSO, reforma visual

**Em produção (build + deploy já feitos, `docker compose build portal` +
`up -d portal`, live em `admn.npxit.com.br`):**

- Permissão granular por recurso (`usuarios`/`instancias`/
  `operacoes_docker`/`credenciais` × `nenhum`/`leitura`/`leitura_escrita`)
  por grupo de segurança — testado ao vivo (matriz na tela de grupo,
  gates em `authz.ts` aplicados nas telas de usuários/instâncias/ações
  Docker/credenciais).
- Atribuição multi-tenant por usuário (`UserTenantAccess`) + seletor de
  tenant no cabeçalho — **testado ponta-a-ponta com usuário descartável**:
  criado, confirmado sem acesso a FLUA, concedido acesso via tela real,
  re-logado, confirmado JWT atualizado e FLUA visível com os 3 instances
  reais, depois removido. Bug real achado e corrigido no processo
  (dashboard ignorava o tenant ativo do seletor — ver `docs/DECISIONS.md`).
- Política de senha na criação de usuário (padrão: senha temporária por
  e-mail + forçar troca no primeiro login; alternativa: super_admin define
  manualmente) — código completo, integrado ao fluxo de criação de
  usuário.
- Menu lateral fixo (`SidebarNav`) com as 4 seções pedidas
  (Monitoramento/Instâncias/Documentação/Acessos-Configurações), navegação
  interna via `next/link` (SPA real — confirmado por grep: nenhum `<a
  href>` interno fora das páginas públicas de login/forgot-password, que
  ficam fora do shell autenticado por natureza).
- Paletas de tema adicionais (azul/verde/roxo/laranja + NPX padrão de
  fábrica) via `data-palette` + CSS custom properties.
- Headers de segurança HTTP (CSP/HSTS/X-Frame-Options/etc.) — confirmado
  ao vivo via `curl -I`.
- Rate limiting em `/login`, `/login/2fa`, `/forgot-password` — confirmado
  ao vivo (limite bateu e bloqueou corretamente numa sessão de teste
  deliberada).
- Responsividade mobile — testada de verdade via Playwright em viewport
  375×812 (não só "deveria funcionar"): login e dashboard renderizam
  limpos, sem overflow horizontal, sidebar colapsa para hamburguer.
  Screenshots em `docs-publish/validation/mobile-login.png` e
  `mobile-dashboard4.png`. Não testei explicitamente o drawer do
  hamburguer *aberto* nesta rodada (só confirmei que colapsa
  corretamente) — se quiser essa checagem visual específica também, é
  rápido de rodar numa sessão futura.

**Construído mas aguardando ação externa/decisão do responsável antes de
ficar 100% ativo:**

- **CAPTCHA (Turnstile)**: código pronto, fail-open enquanto
  `TURNSTILE_SITE_KEY`/`TURNSTILE_SECRET_KEY` não existirem no `.env`.
  **Pendente**: responsável precisa criar o site no painel Cloudflare
  Turnstile (domínio `admn.npxit.com.br`) e colar as chaves.
- **2FA (TOTP)**: código completo e funcional (setup com QR, confirmação,
  desativação, toggle geral em Configurações → Segurança), toggle
  `totpFeatureEnabled` **desligado por padrão**, como pedido. **Ainda não
  testado ao vivo nesta sessão** — falta o responsável fazer o ciclo real
  (ligar o toggle → configurar a própria conta escaneando o QR com um app
  autenticador de verdade → logar com código real → decidir se liga em
  definitivo ou desliga de novo).
- **SSO por tenant (Grafana/Zabbix)**: telas e lógica prontas
  (`/tenants/[id]/sso`). Zabbix chama a API SAML direto; Grafana exige
  editar o compose do tenant + redeploy (sem API de runtime). **Não
  testado ao vivo** — não há um IdP SAML/OIDC real disponível neste
  ambiente pra validar contra ele; a validação real só acontece quando um
  tenant de verdade tiver um IdP pra apontar.
- **GLPI SSO**: não implementado, adiado — ver `docs/ROADMAP.md`.
- **Reset de senha por SMS**: não implementado, descartado (todo provedor
  encontrado tem custo por mensagem) — ver `docs/ROADMAP.md`.

**Limitações conhecidas, aceitas por ora:**

- `accessibleTenantIds` e permissões de recurso ficam embutidos no JWT no
  momento do login — mudanças feitas pelo super_admin só entram em vigor
  no próximo login da pessoa afetada (mesmo padrão já aceito pra mudança
  de `papel`/grupo desde antes deste lote).
- Rate limiting é em memória, por processo — correto hoje (portal roda
  réplica única), precisa virar store compartilhado (Redis) se o portal
  algum dia rodar multi-réplica.
- `npm audit` não foi rodado de forma conclusiva contra a imagem final
  (limitação do estágio `runner` do Dockerfile, sem lockfile após o
  `npm install prisma --no-save`) — varredura de dependências mais formal
  fica como pendência não-bloqueante.

---

## 2026-07-17 — Onboarding MIP ENGENHARIA (unidade FLUA TI) no Zabbix/Grafana da FLUA

**Concluído e confirmado com dado real:**

- Grupos aninhados criados (`MIP ENGENHARIA/BH-MG/Switches`,
  `.../Impressoras`, `.../Servidores VMware`) — convenção documentada em
  `docs/RUNBOOK.md` para uso em todo cliente futuro.
- SW20/SW23/SW25 (já existiam) movidos para o grupo Switches, nenhuma
  config de coleta alterada.
- SW24 (`192.168.0.174`) criado e confirmado respondendo SNMP — **mas
  comprovadamente o mesmo equipamento físico que SW23** (mesma string
  `sysDescr` exata). Mantido por decisão explícita do responsável;
  recomendação de remover está registrada em `docs/DECISIONS.md`.
- 9 de 10 impressoras confirmadas e monitoradas (2 Ricoh + 7 Kyocera),
  templates aplicados e coletando dado real (contagem de página real
  confirmada visualmente no dashboard, ex: Ricoh `192.168.1.113` com
  377988 páginas).
- 3 dashboards Grafana criados e **confirmados com dado real via
  screenshot** (não só HTTP 200 como nas fases anteriores): "MIP
  Engenharia - Visão Geral", "- Switches", "- Impressoras". Acesso
  anônimo/kiosk já herdado da configuração existente da FLUA.
- **Achado + corrigido**: permissão do `grafana-reader` no Zabbix não
  incluía os host groups novos — dashboards mostravam "No data" até a
  permissão ser adicionada. Ver `docs/RUNBOOK.md`, regra permanente para
  todo onboarding futuro.

**Não confirmado / não criado (por não responder):**

- SW21 (`192.168.0.171`): pinga, mas não responde SNMP — investigar
  fisicamente (agente SNMP desligado? community diferente?).
- SW22 (`192.168.0.172`): não pinga.
- Impressora `192.168.1.172`: não responde SNMP.

**Aguardando ação da equipe FLUA (não é erro, é esperado):**

- `ESX01`/`ESX02`: hosts criados no template "VMware Hypervisor",
  macros configuradas (`{$VMWARE.URL}` preenchida,
  `{$VMWARE.USERNAME}` = placeholder, `{$VMWARE.PASSWORD}` = macro
  secreta vazia), **host desabilitado de propósito** (evita alertas de
  falha de autenticação recorrentes enquanto não há credencial real).
  Passos pendentes, ambos fora do alcance deste projeto: (1) equipe FLUA
  preencher usuário/senha reais e habilitar o host; (2) alguém com
  acesso a `FLUA-Proxy-01` (infraestrutura do cliente, sem acesso deste
  projeto) precisa setar `StartVMwareCollectors` no `zabbix_proxy.conf`
  remoto e reiniciar o serviço — sem isso, a coleta não funciona mesmo
  com credencial certa.
- Kyocera: os 2 itens adicionados manualmente (contagem de páginas,
  status geral) usam `delay: 1h`/`5m` — podem levar até 1h para aparecer
  pela primeira vez nos dashboards; toner (LLD) já confirmado
  funcionando.

---

## 2026-07-17 (cont.) — Redesign NOC de parede (Polystat + som de alerta)

**Concluído e validado com screenshot real em 1920x1080 + dado ao vivo:**

- `SW24` removido (confirmado duplicata física do `SW23`).
- Plugin `grafana-polystat-panel` v2.1.16 instalado no Grafana da FLUA
  (gratuito, assinado pela Grafana Labs — achado que corrigiu suposição
  anterior de que seria pago).
- **"MIP Engenharia - Visão Geral"**: painel gigante de status
  (fundo muda verde/amarelo/vermelho conforme o pior problema ativo),
  contagem grande de problemas, som de alerta real (confirmado disparando
  contra um problema crítico real da FLUA — `window.__nocBeepCount` foi a
  2 dentro de ~9s depois de simular o clique de ativação via Playwright).
- **"MIP Engenharia - Switches"**: mosaico de hexágonos Polystat (verde
  UP / vermelho DOWN) por switch, com detalhe de tráfego por porta e CPU
  abaixo. Confirmado com dado real: SW23 apareceu DOWN de verdade durante
  a validação.
- **"MIP Engenharia - Impressoras"**: mesmo padrão de mosaico, usando
  status geral (hrDeviceStatus) — confirmado com dado real: uma
  impressora Kyocera (`192.168.1.127`) apareceu OFFLINE de verdade
  durante a validação.
- Som de alerta implementado **sem nenhuma credencial exposta no
  HTML/JS público** — lê painéis Stat nativos já na tela via DOM, em vez
  de chamar a API do Zabbix direto do navegador (ver `docs/DECISIONS.md`
  pro raciocínio completo e a limitação aceita).
- `GF_PANELS_DISABLE_SANITIZE_HTML=true` ativado globalmente no Grafana
  da FLUA — confirmado com o responsável do projeto antes de ativar
  (afeta a instância inteira, não só este dashboard).
- `portal/scripts/playwright-screenshot.js` ganhou `--dump-html-selector`,
  `--click-selector`, `--eval-js` — ferramentas reutilizáveis, não
  específicas desta sessão.

**Limitação conhecida, aceita:**

- O painel de som depende do atributo interno `data-viz-panel-key` do
  Grafana pra ler os painéis Stat vizinhos — não é API pública, pode
  quebrar numa atualização de versão futura (silencioso, não expõe nada
  se quebrar — só some o som até alguém notar e ajustar o seletor).

---

## 2026-07-18 — Sessão autônoma noturna (madrugada, sem supervisão)

Sessão longa, decisões tomadas sozinho dentro do escopo autorizado
(nunca ação irreversível de produção, nunca conta externa, nunca
gasto). Checkpoints atualizados a cada bloco concluído — ver entradas
abaixo, na ordem em que aconteceram.

### Fase 1 — Câmeras/DVR (go2rtc) — CONCLUÍDA

**Achado antes de agir:** não encontrei nenhum registro de que isso já
tivesse sido construído em sessão anterior (nem no compose, nem no
Grafana, nem aqui em STATE.md) — o histórico específico dessa conversa
não estava disponível nesta sessão. Tratado como trabalho novo, não
como "completar o que faltou". Ver `docs/DECISIONS.md` para o raciocínio
completo.

- Serviço `go2rtc` (imagem `alexxit/go2rtc:1.9.7`) rodando em
  `clients/flua/docker-compose.yml`, roteado via Traefik em
  `https://cameras.flua.npxit.com.br`.
- `go2rtc.yaml` com `streams: {}` — **vazio de propósito**, nenhum IP ou
  credencial de câmera inventado. Comentário no próprio arquivo explica
  o formato a preencher quando os dados chegarem.
- Autenticação básica própria (usuário `suporteti`) protegendo a API —
  testado ao vivo: sem credencial → `401`; com credencial → `200` e
  `{}` (confirma zero streams configurados, como esperado).
- Dashboard **"MIP Engenharia - Câmeras"** criado no Grafana, 4 blocos
  placeholder visuais ("Aguardando IP e credencial RTSP da equipe
  FLUA") — confirmado com screenshot real em 1920×1080
  (`docs-publish/validation/mip_cameras.png`).
- Credenciais em `docs/ACCESS.md`.

**Pendência real, fora do meu alcance:** IPs/credenciais RTSP reais das
câmeras (só a equipe FLUA tem) e criação do registro DNS de
`cameras.flua.npxit.com.br` (mesma situação já documentada pra
`zabbix-master`/`grafana-master` — acesso ao provedor de DNS não é
deste projeto). Nenhum dos dois bloqueia o resto do trabalho.

### Fase 2 — Revisão ao vivo das pendências FLUA/MIP — CONCLUÍDA

Reconferido tudo ao vivo (não confiei no que estava escrito) via API do
Zabbix/Grafana real, ~19h depois do onboarding original:

- **SW21 (`192.168.0.171`)**: ainda pinga, ainda **não responde SNMP**
  (retestado com 2 ciclos completos de `task.create`/check-now, ~4min de
  espera total) — mesmo resultado de ontem, não é flutuação. Precisa de
  alguém fisicamente checando se o agente SNMP está ligado no
  equipamento.
- **SW22 (`192.168.0.172`)**: ainda não pinga — mesmo resultado.
- **Impressora `192.168.1.172`**: ainda não responde SNMP — mesmo
  resultado. Nenhum dos 3 foi recriado como host permanente (só testes
  temporários, removidos depois).
- **Achado novo, real, durante o reteste**: hosts novos criados num
  Zabbix monitorado por **proxy group** (`monitored_by=2`) às vezes
  ficam com `assigned_proxyid=0` por alguns minutos até o grupo terminar
  de rebalancear — não é falha de conectividade do equipamento, é uma
  janela de atraso do próprio Zabbix. Log confirmou: `Proxy group "MIB
  PROXY" changed state from online to degrading` às 14:26:44 e voltou a
  `online` 5s depois. Registrado em `docs/RUNBOOK.md` pra não confundir
  isso com "SNMP não respondeu" numa sessão futura.
- **ESX01/ESX02**: confirmado ainda desabilitados, macros intactas
  exatamente como deixadas — aguardando credencial da equipe FLUA, sem
  mudança.
- **Itens Kyocera (contagem de páginas + status geral)**: **agora com
  dado real** nos 7 hosts (ex: contadores de 103k a 328k páginas) — a
  pendência de "pode levar até 1h pra aparecer" registrada ontem está
  resolvida, sem ação nenhuma necessária.
- **Playlist do NOC criada**: "MIP Engenharia - NOC (parede)", cicla
  Visão Geral → Switches → Impressoras → Câmeras a cada 30s. Confirmado
  com screenshot real (`docs-publish/validation/mip_playlist.png`) — a
  URL fixa pra apontar numa TV é
  `https://grafana.flua.npxit.com.br/playlists/play/dfshmaegg8feod?kiosk=tv`.

**Achados reais de monitoramento (não é bug do nosso lado, é sinal real
do ambiente do cliente — passar pra equipe FLUA/MIP investigar
fisicamente, não decidi mexer em nada disso sozinho):**

- Impressora `192.168.1.127` (Kyocera) segue **offline** desde ontem
  (mesmo problema, contínuo, não é flutuação).
- Ricoh (`192.168.1.113` e `192.168.1.130`): alertas de bandeja
  (`Bypass Tray`/`Tray 3`/`Tray 4` — "Critical: 0", e `Tray 1`/`Tray 2` —
  "Warning") ativos desde a criação, **persistentes por mais de 24h** —
  não é ruído de primeiro poll, parece ser nível real de bandeja/suprimento
  baixo. Vale conferência física.
- Um dos switches (`HP Enterprise Switch: Unavailable by ICMP ping`)
  registrou um evento novo há pouco mais de meia hora, coincidindo com a
  janela de instabilidade do proxy group — pode ser o mesmo evento, pode
  ser independente. Não investiguei mais fundo (fora do escopo desta
  fase, e não é uma decisão que dá pra tomar remotamente sem acesso à
  rede física do cliente).

### Fase 3 — BookStack como tipo de instância provisionável — CONCLUÍDA e testada de ponta a ponta

Primeiro item do catálogo pré-lançamento (`docs/ROADMAP-MACRO.md`, seção
6) implementado via o fluxo self-service real, não um compose manual à
parte — mesmo caminho que qualquer cliente real usa:

- `InstanceTipo` (schema Prisma) ganhou `bookstack`, migrado
  (`prisma db push`, sem perda de dado — adição de enum é aditiva).
- `compose-templates.ts`: fragmento novo (MariaDB 11 + `lscr.io/
  linuxserver/bookstack`), `APP_KEY` do Laravel gerado localmente.
- `provisioning.ts`: `internalBaseUrl`, `SERVICE_KEY_BY_KIND` e
  criação automática do `suporteti` via `php artisan
  bookstack:create-admin --initial` (BookStack não aceita admin via
  env var nem API antes do primeiro login — a própria ferramenta expõe
  esse comando de CLI exatamente pra automação; `--initial`
  **substitui** a conta padrão conhecida `admin@admin.com`/`password`
  em vez de só adicionar uma nova, não deixando credencial padrão
  pública pra trás).
- `service-catalog.ts`, `quotas.ts`, `instance-containers.ts`, telas de
  cota/criação de instância: todos os pontos que tratavam
  zabbix/grafana/glpi como lista fechada foram atualizados.
- Logo oficial baixado do próprio repositório do BookStack (ícone
  colorido, não genérico).

**Testado de ponta a ponta com tenant descartável, através do código
real de provisionamento** (não simulado): tenant criado → compose
gerado → stack subida via Portainer → container respondeu (`302` real)
→ `suporteti` criado com sucesso (confirmado direto no banco do
BookStack: usuário id 1 renomeado de `admin@admin.com` pra
`suporteti`/`suporteti@npxit.com.br` — a conta padrão não existe mais)
→ tudo removido ao final (containers, volumes, diretório do compose,
linhas do banco do portal). Build da imagem `npx-portal:latest` feito e
deployado em produção — o card "BookStack" já aparece de verdade na
tela de criar instância de qualquer tenant, hoje.

**Achado real, não deste lote, registrado como pendência**: o
formulário `/tenants/new` (criar tenant) derruba a sessão do usuário ao
submeter (`303` pra `/login` em vez de `/dashboard`) — confirmado via
captura de rede real durante o teste, não é artefato do navegador
automatizado. Mesma categoria do "quirk" de múltiplos server actions já
documentado nesta sessão. Não corrigido (fora do escopo desta fase, e
exige investigação própria) — **precisa de confirmação manual numa
sessão futura** pra saber se afeta cliques reais de usuário ou só
automação, e se afeta, é bug de produção de prioridade alta (qualquer
gestor criando tenant novo pode estar sendo deslogado no meio do
processo). Ver `docs/DECISIONS.md`.

### Fase 3 (cont.) — Domínio ofuscado automático — CONCLUÍDA e testada

- `suggestObfuscatedDomain()`/`generateObfuscatedSlug()` em
  `provisioning.ts` — slug aleatório de 10 caracteres + domínio de
  entrega configurável (`OBFUSCATED_DELIVERY_DOMAIN`, placeholder
  `.example` enquanto o domínio real não é registrado).
- Tela de criar instância: domínio ofuscado agora é o padrão de
  fábrica, com toggle pra "domínio com nome do cliente" — **testado ao
  vivo nos dois sentidos** via screenshot real (domínio ofuscado gerado
  de verdade: `mrzqqzxdhv.instancias-teste.example`; toggle pra
  identidade mostrou `zabbix.flua.npxit.com.br` corretamente).
- Build + deploy em produção feito — já está ativo pra qualquer tenant
  hoje.

**Não iniciado, registrado em `docs/ROADMAP.md`**: upload de certificado
próprio pelo cliente (a outra metade da Fase 8 do macro) — decisão
consciente de não mexer em infraestrutura Traefik compartilhada sem
supervisão direta. Ver `docs/DECISIONS.md`.

---

## 2026-07-19 — Sessão longa, Fase 1 prioridade máxima: automação real do FortiGate — CONCLUÍDA e testada de ponta a ponta

**O que mudou:** o provisionamento de Zabbix com trapper port agora
aplica o VIP/service/policy **direto no FortiGate via SSH**
(`portal/src/lib/fortigate.ts`) — nunca mais gera um bloco de texto
pedindo pro responsável colar manualmente. Regra permanente registrada
em `CLAUDE.md`. Ver `docs/DECISIONS.md` (entrada 2026-07-19) para o
raciocínio completo, achados técnicos e a discrepância encontrada (o
pedido partia de "escrita já confirmada em sessão anterior" — não
encontrei esse registro, só o perfil lido como read-write; tratei a
escrita real desta sessão como o primeiro teste de verdade, com
cautela, e funcionou).

**VALIDACAO TESTE1 resolvida**: VIP/service/policy
(`zabbix_valid1_12052`/`ZABBIX_VALID1`) aplicados e confirmados na
configuração ao vivo do FortiGate. `docs/PORT-REGISTRY.md` atualizado.
Pendência cosmética não resolvida (bloqueada pelo classificador de
permissão — mudança direta em dado de produção fora do fluxo da
aplicação): o campo `metadata.fortigateInstructions` dessa instância no
banco do portal ainda guarda o texto antigo da instrução manual, sem
efeito funcional (a tela não exibe mais esse bloco, o componente foi
removido).

**Testado de ponta a ponta com instância descartável nova** (não só o
caso pendente antigo): tenant → porta alocada → checagem de conflito ao
vivo no FortiGate → VIP/service/policy aplicados via SSH → confirmado
numa releitura da config → stack subida → container respondendo →
`suporteti` criado → **zero ação manual em qualquer etapa**. Confirmado
com `result.ok: true, fortigateApplied: true` e reconferido direto no
FortiGate. Tudo removido depois (containers, volumes, banco, e os 3
objetos no FortiGate) — porta 12055 marcada como liberada no registry,
nunca reutilizar.

**Robustez adicionada durante o teste** (achado real, não hipotético):
se uma parte do trio VIP/service/policy falhar depois da(s) outra(s) já
ter(em) sido criada(s), a automação agora desfaz automaticamente o que
foi criado antes de reportar erro — nunca deixa objeto órfão no
FortiGate. Também ganhou uma checagem de comprimento de nome antes de
tentar (limite real de 35 caracteres no campo `name` de policy do
FortiOS, descoberto ao vivo).

**Env vars novas em produção**: `FORTIGATE_HOST`, `FORTIGATE_SSH_PORT`,
`FORTIGATE_USER`, `FORTIGATE_PASSWORD`, `NPX_HOST_LAN_IP` — adicionadas
em `portal/.env` e `portal/docker-compose.yml`. Build + deploy do
`npx-portal:latest` feitos e confirmados em produção.

---

## 2026-07-19 (cont.) — Fases 2 a 6 da mesma sessão longa

**Fase 2 — slug automático:** campo "Slug" removido do formulário de
criação de tenant (`tenants/new/page.tsx`); gerado automaticamente a
partir do nome (`portal/src/lib/slug.ts`, normalização + sufixo em
conflito), sem confirmação. Continua visível na tela de detalhe do
tenant (`tenants/[id]/page.tsx`) e documentado em `docs/RUNBOOK.md`.

**Fase 3 — seletor de tenant sumido:** causa raiz real era de
autorização, não só de UI — `accessibleTenantIds` só considerava
`UserTenantAccess` (pensado como exceção pontual), sem o default de
hierarquia (tenant raiz vê todos; tenant nível 1 vê a si + subtenants).
Corrigido em `portal/src/lib/session-helpers.ts`. Verificado por
screenshot: seletor aparece no cabeçalho pro Super Admin NPX.

**Fase 4 — menu lateral reorganizado:** `SidebarNav.tsx` agora agrupa em
Monitoramento / Instâncias / Integrações / Documentação /
Acessos-Configurações, com ícones. O indicador de tenant atual
(`TENANT ATUAL` + seletor) passou a aparecer sempre no topo do menu,
não só na Dashboard.

**Fase 5 — progresso de provisionamento detalhado:** concluída e
testada ao vivo (screenshot real capturado em pleno andamento, 50%,
3 de 6 etapas concluídas). Etapas nomeadas e estáveis
(`portal/src/lib/provisioning-steps.ts`), stepper visual com ícone/cor
por status (`ProvisioningStepper.tsx`), polling a cada 2s
(`ProvisioningHistory.tsx`), erro mantém etapas anteriores marcadas
como concluídas e mostra a mensagem real. Nova coluna
`wants_trapper_port` em `ProvisioningAudit` (migrada via `prisma db
push`).

**Achado colateral importante desta fase:** o bug recorrente "sessão
cai / 303 → /login" investigado (sem sucesso) em sessões anteriores foi
**resolvido e explicado** — não era bug de autenticação. Ver
`docs/DECISIONS.md` (entrada 2026-07-19, "Causa raiz real..."). Resumo:
era um seletor ambíguo nos scripts de teste (clicava no botão "Sair" em
vez do botão da página) somado a um bug real e separado (exceção não
tratada numa Server Action se manifestando como logout silencioso,
agora blindado em `provisionInstanceAction`/`createTenantAction`).
Next.js atualizado de 14.2.15 para 14.2.35 (manutenção, não foi a causa
nem a correção).

**Fase 6 — catálogo com identidade visual:** cards com logo/nome/descrição
curta já existiam de uma fase anterior (`service-catalog.ts`). Adicionado
nesta fase: página descritiva por produto, tom de venda pro cliente
final, sem jargão técnico (`service-catalog-details.ts` +
`instances/new/[tipo]/page.tsx`), linkada via "Saiba mais →" em cada
card. Feito para os 4 produtos hoje implementados (Zabbix, Grafana,
GLPI, BookStack) — ver Fase 7 abaixo pros pendentes do catálogo.

**Fase 7 — estado real do catálogo pendente:**

`docs/ROADMAP-MACRO.md` (seção 3) lista 5 produtos pendentes do catálogo:
CrowdSec, Wiki.js/BookStack, Nextcloud, Pi-hole/AdGuard Home, Chatwoot.
Auditoria real feita nesta sessão (grep no código, não achismo):

- **Wiki.js ou BookStack — RESOLVIDO.** BookStack foi a escolha
  implementada (sessão anterior) e está completo: compose, admin
  `suporteti` automático (`bookstack:create-admin --initial`), catálogo,
  logo, e agora (Fase 6 desta sessão) página descritiva. Nada pendente
  aqui.

- **CrowdSec — não implementado, e não é do mesmo formato dos outros 4.**
  Zabbix/Grafana/GLPI/BookStack são cada um uma ferramenta autocontida
  que o cliente acessa via navegador. CrowdSec é um **agente de
  detecção** que só tem valor analisando o tráfego/logs de OUTRA coisa
  (o próprio host, ou os containers do cliente) — rodar um CrowdSec
  "solo" sem nada apontando pra ele não protege nada. Antes de
  implementar, precisa de uma decisão de produto: CrowdSec protege a
  infraestrutura da NPX (uso interno, não é item de catálogo pro
  cliente) ou vira uma oferta "protejo o que você já tem hospedado
  aqui" (mais fácil, dado que já roda na mesma infra) vs. "protejo
  qualquer coisa sua, em qualquer lugar" (exige agente instalável fora
  daqui, bem mais trabalho)? Isso muda a arquitetura inteira, não é só
  "adicionar ao catálogo".

- **Nextcloud — não implementado, mas é o mais parecido com o padrão
  já existente.** Mesmo formato dos outros 4 (app + banco, HTTPS via
  Traefik, criação de admin via linha de comando — `occ user:add`,
  documentado e não-interativo, mesmo princípio do `artisan
  bookstack:create-admin`). Única diferença real de peso: Nextcloud é
  armazenamento de arquivo — os limites de recurso/disco atuais
  (`RESOURCE_LIMITS`, pensados pra apps "leves" tipo GLPI/BookStack) não
  fazem sentido pra um produto cujo valor é justamente guardar volume de
  arquivo do cliente; precisa de uma decisão de cota de disco por
  tenant antes de expor isso como autosserviço sem controle (senão um
  cliente pode encher o disco do host compartilhado). Sem essa decisão,
  não implementado ainda — não é falta de tempo, é acompanhar a mesma
  disciplina que gerou a Fase 1 desta sessão (nunca cortar canto que gera
  problema real, "zero ação manual" não vale nada se o resultado quebra
  os outros clientes no mesmo host).

- **Pi-hole ou AdGuard Home — não implementado, bloqueio real de
  arquitetura de rede.** O valor central dessas ferramentas é filtrar
  DNS — o cliente precisa apontar o resolvedor DNS da rede dele pro
  Pi-hole hospedado aqui. Isso não é "HTTPS atrás do Traefik" como os
  outros 4: exige expor a porta 53 (DNS, UDP+TCP) do container pro
  cliente alcançar de fora, o que é **o mesmo tipo de problema já
  resolvido na Fase 1 pro trapper port do Zabbix** (alocação de porta +
  regra de firewall no FortiGate) — só que pra DNS, provavelmente uma
  porta 53 dedicada por cliente já que 53 não pode ser compartilhada
  entre múltiplos containers no mesmo host/IP público. Reaproveitar o
  padrão de `docs/PORT-REGISTRY.md` + `fortigate.ts` é o caminho óbvio,
  mas precisa de decisão de produto antes (o cliente aponta o DNS de
  quê exatamente — rede inteira dele via VPN, ou só dispositivos
  específicos?) — sem isso definido, implementar só a metade "container
  sobe e a UI abre" seria exatamente o tipo de entrega incompleta que a
  regra permanente de zero-ação-manual deste projeto existe pra evitar
  (o cliente teria uma ferramenta que parece pronta mas não filtra nada
  de verdade pra rede dele).

- **Chatwoot — não implementado, maior escopo técnico dos 5.**
  Diferente dos outros (1 app + 1 banco), Chatwoot precisa de Postgres +
  Redis + processo Rails + worker Sidekiq (mínimo 4 serviços), e
  configuração de envio de e-mail (já resolvida nesta plataforma via
  Brevo, reaproveitável) mais integração de canal (WhatsApp, `docs/STATE.md`
  "Fase H", já documentada sem implementar). Também é o único dos 5 com
  **uso duplo já decidido** no roadmap (seção 11/19): ferramenta de
  catálogo pro cliente E ferramenta interna da própria NPX — vale
  decidir se a instância interna da NPX é o primeiro caso de uso real
  (testando o padrão) antes de expor a clientes.

**Nenhum dos 4 pendentes foi implementado nesta sessão** — depois de
levantar os requisitos reais de cada um, ficou claro que os 3 primeiros
(CrowdSec, Nextcloud, Pi-hole/AdGuard) têm uma decisão de produto/infra
genuína pendente antes de qualquer linha de código valer a pena (não é
"falta de tempo", é "implementar agora seria ou meio-produto exposto ao
cliente, ou trabalho jogado fora se a decisão for diferente depois"), e
Chatwoot tem escopo técnico grande o bastante pra merecer sessão própria
dedicada. Ver `docs/DECISIONS.md` pro raciocínio completo por produto.
Achado colateral real desta auditoria (não relacionado ao catálogo
pendente): testando o fluxo de BookStack ao vivo, veio à tona uma
condição de corrida real entre o container do banco (MySQL/MariaDB) e o
container do app nos 3 tipos que dependem de banco (Zabbix, GLPI,
BookStack) — `depends_on` sem `condition: service_healthy` só garante
ORDEM de start, não que o banco já aceite conexão; corrigido com
healthcheck real (`mysqladmin`/`mariadb-admin` conforme a imagem) +
`depends_on` condicional em `compose-templates.ts`.

---

## 2026-07-19 (cont.) — CONFIRMADO: BookStack via self-service funcionando de ponta a ponta

Depois de mais duas rodadas de depuração ao vivo (detalhes completos em
`docs/DECISIONS.md`): o Portainer ficou temporariamente bloqueado
(proteção de força bruta própria dele, corrigida com cache de JWT), e a
causa real de o BookStack nunca completar não era a corrida de banco —
era `waitForInternalHttp` seguindo o redirect 301/302 que o BookStack
(Laravel) sempre devolve pro `APP_URL` configurado, mesmo internamente;
como o domínio padrão de toda instância nova é o ofuscado
(`*.instancias-teste.example`, proposital e permanentemente
não-resolvível), seguir esse redirect sempre falhava. Corrigido com
`fetch(..., { redirect: 'manual' })`. **Teste real confirmado**: BookStack
provisionado via self-service (formulário real, sem atalho) do início ao
fim em 101 segundos, status "ativo", `suporteti` criado, histórico de
provisionamento mostrando "✅ sucesso — concluído" — screenshot real
capturado. Instância de teste removida depois (containers, volumes,
linha do banco, bloco do compose do tenant NPX IT), GLPI real do mesmo
tenant confirmado não afetado (200 OK) durante e depois de toda a
depuração.

Esta é provavelmente a primeira vez que uma instância de BookStack foi
provisionada com sucesso via self-service desde que o tipo foi
adicionado ao catálogo — o bug do redirect existia desde então, só nunca
tinha sido testado de ponta a ponta antes.

---

## 2026-07-26 — FASE 1 (sessão de segurança): bug de isolamento entre tenants corrigido de verdade, conceito ADMN introduzido

**Confirmado e testado em produção.** Ver `docs/DECISIONS.md` pra causa
raiz completa. Resumo do estado atual:

- **ADMN** existe como tenant real (`slug: admn`, `isPlatformRoot: true`)
  — a única raiz de verdade da plataforma agora. Os 4 usuários operadores
  da NPX (`admin@npxit.com.br`, `suporteti@npxit.com.br`,
  `tulio@npxit.com.br`, `nicholasalex@gmail.com`) vivem dentro dele.
- Hierarquia real hoje: **ADMN** → NPX IT, FLUA TI, Tulio Felix,
  VALIDACAO TESTE1, validteste2 (todos nível 1, filhos diretos do ADMN)
  → validnivel2 (nível 2, filho de VALIDACAO TESTE1).
- FLUA TI **não é mais filha da NPX IT** — reparentada pra filha direta
  do ADMN via a feature real de mover tenant (não script), corrigindo a
  impressão errada de "FLUA é cliente da NPX" (as duas são clientes
  diretas da plataforma, irmãs, nunca uma dependente da outra).
- `Tenant.isPlatformRoot` e `SessionPayload.isAdmn` são os únicos sinais
  válidos de "isto é a plataforma" — nunca mais inferido de
  `parentTenantId === null` ou de `papel === 'super_admin'` isolado.
- Formulário de criação de tenant não permite mais criar tenant sem pai
  — fecha a causa raiz na origem, não só no lado da leitura.
- Nova feature (`moveTenantAction`, restrita a ADMN) pra reparentar
  tenant, com validação de profundidade máxima (2 níveis abaixo do
  ADMN) e anti-ciclo.
- Bug do seletor de tenant "errado" dentro de `/tenants/[id]/...`
  corrigido (`AppShell` agora prioriza o tenant da URL sobre o cookie,
  validando contra `accessibleTenantIds` antes de aceitar).
- **Teste real de ponta a ponta feito e confirmado** (Playwright com
  contas reais, tentativa de acesso direto por URL manipulada) —
  técnico e gestor dentro de VALIDACAO TESTE1 bloqueados (redirecionados
  pro dashboard) ao tentar `/tenants/{npx}/instances`,
  `/tenants/{flua}/instances`, `/tenants/{admn}/instances`; permitido
  normalmente em `/tenants/{valid1}/...` (próprio) e
  `/tenants/{validnivel2}/...` (subtenant próprio). ADMN confirmado
  vendo tudo. Ver relatório da sessão pro passo a passo exato, pra
  revalidação manual do responsável do projeto.

**Pendências que ficaram fora desta fase** (fora do escopo do bug de
segurança, registradas pra próxima):
- `docs/ROADMAP-MACRO.md` seção 4 ("Exceção pontual" via
  `UserTenantAccess`) foi realinhada pro ADMN, mas o mecanismo em si não
  foi testado de ponta a ponta nesta sessão (não fazia parte do bug
  reportado).
- Os 15 itens de bugs/ajustes da "validação profunda" e a reconstrução
  do menu lateral são as próximas fases da mesma sessão.

## 2026-07-26/27 — FASE 2 (mesma sessão longa): 16 itens da validação profunda — CONCLUÍDA

Ver `docs/DECISIONS.md` (entradas de 2026-07-26 e 2026-07-27) pro
achado real de cada item. Resumo do estado atual, item por item:

1. **Gestor não conseguia criar instância** — corrigido (default de
   `instancias` pra `gestor` era `leitura`, virou `leitura_escrita`).
2. **Tela de Cota "não existia"** — já existia, era vítima do mesmo bug
   de isolamento da Fase 1; resolvida automaticamente + reposicionada
   pra dentro de "Editar tenant".
3. **Branding não persistia** — nunca existiu formulário de verdade;
   construído do zero (nome/cor/tema/upload de logo e favicon), grava
   em `tenant.branding`, sobrevive logout/login (testado real).
4. **Status de DNS "aguardando" eterno** — domínio placeholder agora
   rotulado explicitamente como "domínio de demonstração — não resolve
   publicamente" em vez de pendência eterna sem explicação.
5. **"Reconectar" sempre mostrava sucesso** — banners reais
   verde/vermelho/âmbar substituindo o sempre-verde.
6. **Start/Stop/Restart sem feedback visual** — `ButtonSpinner` +
   `HealthDot` (verde/âmbar/vermelho, consistente em toda tela que
   mostra status de instância/container).
7. **"Deslogar" não funcionava pra gestor em subtenant** — testado com
   gestor nível 1 e nível 2 reais, funcionou nos dois casos; não
   reproduzido (fica registrado pro responsável descrever o passo a
   passo exato se acontecer de novo).
8. **CAPTCHA sem indicador visual** — rótulo visual adicionado na tela
   de login.
9. **Logs/Diagnóstico "parecia terminal cru"** — `LogViewer` com
   chrome de terminal, cor por palavra-chave, numeração de linha,
   mantendo o conteúdo técnico real.
10. **"Saiba mais" navegava pra página cheia** — virou modal
    (`ServiceInfoModal`) com X visível, mais imagens, casos de uso reais
    (`casosDeUso`), texto mais comercial.
11. **Exclusão de instância (não existia)** — implementada em cascata
    completa (FortiGate → containers → volumes → compose → banco).
    **Incidente real durante o teste** (seletor de teste ambíguo
    apagou a linha errada do banco — nenhuma infraestrutura real foi
    tocada) levou a uma correção de segurança adicional: `removeContainer`/
    `removeVolume` agora reportam se algo realmente existia;
    `deleteInstanceCompletely` sinaliza em destaque quando nada real foi
    encontrado (`nothingRealFound`); `deleteInstanceAction` para antes
    de tocar no banco nesse caso, exigindo confirmação explícita
    separada. Linha restaurada (aprovada pelo responsável, SQL
    conferido contra o tenant certo antes de rodar). Reteste com escopo
    correto **passou de ponta a ponta**, confirmado direto na
    infraestrutura real (containers/volumes/compose/banco).
12. **Naming do FortiGate padronizado** —
    `{tenant}-zabbix-trapper-{porta}` (VIP) /
    `{tenant}-zabbix-{porta}` (policy, ≤35 chars) + campo de comentário
    com cliente/motivo/data.
13. **Matriz de permissões "parcial"** — já era completa (4 recursos);
    faltava clareza — tooltip por recurso explicando o que cada nível
    cobre de verdade.
14. **2FA granular** — `Tenant.totpMode`
    (`herda_plataforma`/`obrigatorio`/`opcional`/`desabilitado`),
    `totpDelegadoGestor` (ADMN delega pro gestor do tenant), obrigatório
    força setup no próximo login (mesmo mecanismo do
    `mustChangePassword`). Reposicionado pra dentro de "Usuários" do
    tenant.
15. **Som de alerta do NOC sem botão de mute** — adicionado e testado
    ao vivo contra alerta real. "Som não para ao trocar de dashboard" —
    não reproduzido (era artefato do próprio teste); achado real mais
    grave no lugar, documentado: o som não volta a tocar sozinho depois
    do primeiro ciclo da playlist (política de autoplay do navegador) —
    não corrigido nesta sessão, fora do escopo de um painel individual.
16. **Domínio-base configurável pelo ADMN** — só documentação, registrado
    em `docs/ROADMAP.md` como pedido (não implementado agora).

**Novo item de roadmap gerado por este lote** (não pedido originalmente,
nasceu do incidente do item 11): ação self-service "Registrar instância
existente" (`docs/ROADMAP.md`), pra nunca mais precisar de SQL manual
pra recriar um registro de rastreamento.

**Pendências que ficam fora desta fase:**
- Item 7 (logout) não reproduzido — só volta se o responsável descrever
  o caso exato.
- Achado do som do NOC (autoplay) não corrigido, exigiria mudança no
  nível da aplicação Grafana, não do painel.
- Achado à parte (não bloqueante): chamadas recorrentes sem sessão em
  `/tenants/{npx}/instances` nos logs do portal, padrão de aba de
  navegador esquecida aberta com polling ativo — ver
  `docs/DECISIONS.md` (2026-07-27).
- FASE 3 (reconstrução do menu lateral) e FASE 4 (assistente de IA via
  OpenRouter, piloto ADMN) são as próximas fases da mesma sessão longa.

## 2026-07-27 (cont.) — FASE 3: menu lateral reconstruído do zero — CONCLUÍDA

Ver `docs/DECISIONS.md` pro raciocínio completo de organização. Seções
novas (`SidebarNav.tsx`, tratado como reescrita, não ajuste da versão de
2026-07-19): **Painel, Instâncias, Integrações, Este tenant,
Documentação, Plataforma (só ADMN — some completamente pra quem não é
ADMN), Minha conta**. Inspirado na hierarquia do Acronis Cloud
(Clientes/Monitoramento/Minha Empresa/Integrações/Configurações),
adaptado aos termos e telas reais deste produto — itens do Acronis sem
equivalente real aqui (Caixa de Entrada, Vendas/Cobrança) foram
omitidos, não inventados.

**Bug real encontrado e corrigido durante a reconstrução:** o link de
"Segurança (2FA/SSO)" estava escondido atrás de `isAdmn`, mas
`/settings/security` também é onde qualquer usuário configura o
próprio 2FA pessoal ("Meu 2FA") — usuários não-ADMN não tinham como
chegar nessa tela pelo menu nenhum. Corrigido: agora visível pra todo
mundo em "Minha conta"; o toggle GLOBAL de 2FA continua (corretamente)
restrito a ADMN só dentro da própria página.

**Item novo adicionado ao menu:** "Aparência do tenant" (branding) não
tinha NENHUM link de navegação antes (só acessível digitando a URL
direto) — agora em "Este tenant", gated por `canManageUsersInTenant`.

**Teste real feito (Playwright, não simulado):**
- ADMN, desktop (1440×900): todas as 7 seções visíveis, incluindo
  "Plataforma (só ADMN)"; tenant ativo mostrado corretamente no topo
  (testado trocando de ADMN pra NPX IT via URL, seletor acompanhou).
- ADMN, mobile (390×844): gaveta abre/fecha, todas as seções
  presentes, rodapé (nome/Sair) fixo visível sem precisar rolar até o
  fim.
- Gestor nível 2 (`gestorn2@teste.com`, tenant `validnivel2`), desktop:
  confirmado que "Plataforma", "Todos os tenants" e "Criar tenant" NÃO
  aparecem (checado via busca de texto, não só visual); "Segurança
  (2FA)" aparece corretamente (bug corrigido, confirmado).
- Gestor nível 2, mobile: mesma estrutura reduzida, gaveta funcional.

Screenshots reais salvos durante o teste (não commitados no repo,
artefato de sessão):
`sidebar-desktop-dashboard.png`, `sidebar-desktop-tenant-context.png`,
`sidebar-mobile-closed.png`, `sidebar-mobile-open.png`,
`sidebar-gestor-desktop.png`, `sidebar-gestor-mobile-open.png`.

## 2026-07-27 (cont.) — FASE 4: base técnica do assistente de IA (OpenRouter) — PROTÓTIPO/TESTE, CONCLUÍDA

Ver `docs/DECISIONS.md` pro relato completo. Estado atual:

- **Telas novas:** `/settings/ai` (Configurações de IA — ADMN only:
  toggle, chave cifrada, teste real de chave + lista de modelos via
  API do OpenRouter, seleção de modelo) e `/settings/ai/chat` (chat
  funcional, bloqueado se não configurado/habilitado). Ambas com banner
  "PROTÓTIPO/TESTE" e sem nenhum caminho de acesso pra tenant cliente.
- **Escopo garantido:** toda ferramenta da IA opera só sobre
  `session.tenantId` de quem abriu o chat — nunca um parâmetro vindo do
  modelo ou da URL. Como só ADMN chega nessa tela, isso restringe a IA
  ao tenant ADMN nesta fase, por construção (não por checagem
  contornável).
- **Ferramentas reais:** `listar_instancias`, `diagnosticar_instancia`,
  `reiniciar_instancia` — as duas últimas reaproveitam as mesmas
  actions já usadas pela UI humana (`getInstanceDiagnosticsAction`,
  `containerActionAction`), não duplicam lógica de container.
- **Auditoria:** toda chamada de ferramenta exige `justificativa` no
  próprio schema e grava em `AiActionLog` (sucesso ou falha), nunca
  apagado por rotina nenhuma.
- **Chave:** `PlatformSettings.aiApiKeyEncrypted`, cifrada com o mesmo
  padrão AES-256-GCM (`lib/crypto.ts`) já usado pra credenciais de
  instância — nunca texto plano, nunca reexibida, nunca logada.

**Teste real de ponta a ponta (chave de API de verdade, configurada
pelo próprio responsável do projeto direto na UI):**
1. Bug real encontrado e corrigido no meio do teste: header HTTP
   `X-Title` com caracteres fora de Latin-1 (`—`/`ó`) quebrava toda
   chamada ao OpenRouter — corrigido pra ASCII puro.
2. O modelo **recusou duas vezes** reiniciar uma instância saudável só
   porque "confirmar que a ferramenta funciona" não é justificativa
   técnica real — mesmo com autorização explícita do operador. A
   disciplina de auditoria/justificativa do system prompt se mostrou
   real, não decorativa.
3. Com um problema genuíno (container `admn-grafana` parado
   manualmente via `docker stop`, fora do portal), pedido "verifique o
   alerta e resolva se encontrar algo" resultou em: diagnóstico →
   achou `exited` → reiniciou com justificativa própria, tecnicamente
   correta → diagnosticou de novo pra confirmar a recuperação. Toda a
   sequência confirmada no `ai_action_log` (consulta direta na tabela)
   e no `docker events` (evento real de restart no timestamp certo) —
   **ação real de configuração confirmada, não só resposta de texto**.
4. Bug real de cliente encontrado e corrigido: a tela quebrava
   (`Application error`) numa resposta demorada, por falta de
   `try/catch` na chamada RPC da Server Action — a ação real já tinha
   executado com sucesso no servidor antes da tela quebrar (confirmado
   pelo log de auditoria). Corrigido.
5. Instância de teste descartável (Grafana, criada dentro do próprio
   tenant ADMN pra este teste) removida em cascata com sucesso ao
   final (container/volume/compose/banco confirmados limpos).

**Pendências que ficam fora desta fase:** a arquitetura de produção
final (isolamento por VM dedicada por tenant, motor nunca exposto ao
cliente, cota de uso) continua indefinida — ver
`docs/ROADMAP-MACRO.md` seção 10. Este protótipo não decide nem
antecipa essa arquitetura.

**As 4 fases da sessão longa de validação profunda estão concluídas:**
Fase 1 (isolamento entre tenants/ADMN), Fase 2 (16 itens de bugs/
ajustes), Fase 3 (menu lateral reconstruído), Fase 4 (base do
assistente de IA, protótipo ADMN).

## 2026-07-27 (nova sessão longa) — FASE 0: mistério do Zabbix mestre resolvido — CONCLUÍDA

Ver `docs/DECISIONS.md` pro relato completo da investigação. Resumo do
estado atual:

- **3 cadeias de processo órfãs do Claude Code encerradas** (rodando
  em background havia 10-11 dias sem qualquer conversa ativa,
  `permission-mode auto`): sessão `41ec8d27` (com uma tarefa de shell
  travada há 10 dias, em loop infinito inútil), e o daemon "transiente"
  órfão da sessão `3cb4a0bf` (parent já morto, confirmado). Não foi
  possível provar 100% qual processo exato gerava o `docker exec
  npx-mysql` que aparecia no journal a cada 60s (o padrão já tinha
  parado sozinho ~4h antes desta sessão começar) — mas nenhuma nova
  ocorrência apareceu depois de encerrar as 3 cadeias. **Não
  encerrado:** o `claude` interativo do console `tty1` (pid 6777,
  idle 12 dias, mas é sessão em primeiro plano, não daemon) — fica
  para o responsável do projeto decidir.
- **Stack `npx-zabbix` recriado reaproveitando os volumes existentes**
  (`npx-mysql`, `npx-zabbix-server`, `npx-zabbix-web` — só esses 3
  estavam fora do ar; `npx-zabbix-agent` e `npx-grafana` já estavam de
  pé). Nenhum volume foi apagado ou recriado do zero.
- **Confirmado com dado real via API do Zabbix mestre** (não só "subiu
  sem erro"): CPU do host real com timestamp fresco batendo com o
  segundo da consulta, 578 itens `docker.container_info` (descoberta
  automática, template Docker) atualizados há ~1 minuto, incluindo os
  containers recém-recriados do próprio stack. Host `Docker-Host-
  suporteti` foi reconhecido do banco antigo (preservado no volume),
  não recriado do zero.

**Estado atual dos containers do stack:** `npx-mysql`,
`npx-zabbix-server`, `npx-zabbix-web` (healthy), `npx-zabbix-agent`,
`npx-grafana` — todos `Up`.

**Pendência que já existia antes desta fase, sem mudança:** DNS de
`zabbix-master.npxit.com.br`/`grafana-master.npxit.com.br` ainda não
criado (certificado self-signed de fallback via Traefik).

## 2026-07-27 (mesma sessão longa) — FASE 1: backup granular por instância (Kopia) — CONCLUÍDA

Peça de maior valor comercial da sessão. Ver `docs/DECISIONS.md` (entrada
"FASE 1: backup granular por instância") pro racional completo de cada
decisão de arquitetura, e `docs/portal/ARCHITECTURE.md` (seção "Backup
granular por instância") pro desenho técnico completo. Resumo do que
está de pé e validado com dado real:

- **Motor Kopia rodando isolado**, `/opt/npx-platform/backup/`: Repository
  Server (`npx-kopia-server`, storage local em disco, rede
  `backup_internal` sem exposição nenhuma à internet) + agente
  (`npx-kopia-agent`, único componente com `docker.sock`, API HTTP
  própria porta 8090, também na rede `portal_internal` pra o backend do
  portal alcançar).
- **Dump lógico antes de todo snapshot** — nunca copia arquivo de banco
  vivo. Detecta motor automaticamente (MySQL/MariaDB via
  `mysqldump`/`mariadb-dump`, Postgres via `pg_dumpall`) pela env var
  presente no container.
- **Um usuário Kopia por TENANT** (não por instância — decisão
  documentada com racional em `docs/DECISIONS.md`), credencial gerada
  automaticamente e cifrada em repouso (`tenant_backup_configs`, AES-256-
  GCM), nenhum tenant fala com o Kopia diretamente.
- **Retenção configurável por tenant** (dias via policy nativa do Kopia +
  tamanho máximo enforced manualmente pelo agente), editável em
  `/backups/admin` (ADMN-only).
- **Telas funcionando**: `/tenants/[id]/backups` (tenant: listar, "Backup
  agora", restaurar com 2 opções — sobrescrever ou copiar — com
  confirmação em 2 etapas) e `/backups/admin` (ADMN: retenção por
  tenant).
- **Postgres do próprio portal incluído** na mesma disciplina
  (`instanceId` fixo `"portal-db"`, tela dedicada, restrita ao tenant
  raiz).
- **Testado de ponta a ponta com dado real, não simulado**: backup +
  alteração + restore + confirmação de reversão numa instância MySQL
  descartável; backup real (via UI, Playwright/Chromium com login real)
  da instância Zabbix de produção da FLUA TI (112MB de dump); restore
  como cópia do mesmo backup, confirmado sem tocar o original; backup
  real do Postgres do portal via UI; retenção salva via UI e confirmada
  no banco.

**Pendências registradas para o futuro (não bloqueiam a fase, mas são
próximos passos naturais):**
- **Sem backup automático agendado ainda** — só "Backup agora" manual.
  Sem isso, a retenção por dias só tem efeito prático se alguém lembrar
  de rodar backup periodicamente. Próximo passo natural: cron/systemd
  timer disparando backup de todas as instâncias com alguma
  periodicidade configurável.
- **Storage ainda é só filesystem local** — sem provedor S3 externo.
  Decisão consciente de custo/complexidade dado o volume atual pequeno;
  trocar exige só mudar o backend de storage no `repository.config` do
  Kopia, revisitar quando o volume de backup justificar.
- **Isolamento dentro do mesmo tenant é só por permissão `backups`**
  (tudo ou nada) — não existe hoje um jeito de um papel ver backup só de
  uma instância específica dentro do próprio tenant. Só relevante se
  aparecer um caso de uso real (ex: técnico de uma filial só devendo ver
  o backup do Zabbix da própria filial).

## 2026-07-27 (mesma sessão longa) — FASE 2: catálogo completo (Vaultwarden + Uptime Kuma) — CONCLUÍDA

Vaultwarden e Uptime Kuma agora são tipos de instância provisionáveis,
mesmo padrão dos 4 tipos anteriores (stack isolada, Traefik, Let's
Encrypt, `suporteti` automático, card com logo oficial, página descritiva
comercial). Ver `docs/DECISIONS.md` pro racional de bootstrap de admin de
cada um (mais delicado que os tipos anteriores, nenhum dos dois tem
variável de ambiente pra criar admin no boot) e pra um achado real
importante (bug do Next.js/webpack quebrando `socket.io-client` dentro do
processo do portal — não é bug desta feature especificamente, é uma
classe de bug que pode voltar a aparecer com qualquer lib futura que
dependa de módulo nativo opcional).

- **Vaultwarden**: `ADMIN_TOKEN` gerado automaticamente no provisionamento
  (compose), confirmado autenticando de verdade no painel `/admin` antes
  de marcar o passo como concluído (POST real, não só "container
  respondeu"). `SIGNUPS_ALLOWED=false` por padrão — usuários novos são
  convidados via o próprio painel admin, não se cadastram sozinhos.
- **Uptime Kuma**: usuário `suporteti` criado via protocolo Socket.IO
  interno (único mecanismo disponível — sem variável de ambiente de
  bootstrap). Ao provisionar, pré-cadastra automaticamente um monitor
  HTTP pra cada outra instância ATIVA do mesmo tenant, usando a URL
  PÚBLICA de cada uma (detecta quando o caminho completo — DNS +
  certificado + Traefik + container — quebra, não só "o container está de
  pé").
- **Bug real corrigido nesta fase** (detalhe completo em
  `docs/DECISIONS.md`): o provisionamento de Uptime Kuma travava 100% das
  vezes na criação do `suporteti`, sempre no mesmo ponto (`login` logo
  após `setup`, mesma conexão Socket.IO), sem erro visível em lugar
  nenhum. Causa raiz: o webpack do Next.js empacotando a lib `ws`
  (dependência do `socket.io-client`) quebra o fallback de uma dependência
  nativa OPCIONAL dela (`bufferutil`, não instalada de propósito), fazendo
  todo frame WebSocket ≥ 48 bytes travar silenciosamente. Corrigido com
  `experimental.serverComponentsExternalPackages` em `next.config.js`
  (nome correto pra Next.js 14.2.x — o nome estável `serverExternalPackages`
  só existe a partir do Next 15).
- **Testado de ponta a ponta com dado real**: provisionamento completo via
  UI real (Playwright/Chromium, login real) pros dois tipos, no tenant de
  teste VALIDACAO TESTE1 (que já tinha Zabbix ativo). Login do `suporteti`
  do Uptime Kuma validado de forma independente do fluxo de
  provisionamento (conexão separada, feita depois). Pré-cadastro de
  monitores confirmado lendo `monitorList` de volta do próprio Uptime
  Kuma — as duas outras instâncias do tenant (Zabbix e Vaultwarden)
  apareceram corretamente como monitores.

**Pendência que já existia antes desta fase, sem mudança:** nenhuma nova
pendência introduzida — os dois tipos seguem exatamente o mesmo padrão
operacional dos 4 tipos anteriores (backup granular via Kopia já
funciona pra eles também, já que `instance-containers.ts` foi atualizado
junto).

## 2026-07-27 (mesma sessão longa) — FASE 3: múltiplas instâncias do mesmo tipo por tenant — CONCLUÍDA

Trava de unicidade `@@unique([tenantId, tipo])` removida — hoje um
tenant pode ter quantas instâncias quiser do mesmo tipo (ex: dois
Zabbix, um por unidade; dois Vaultwarden, um por equipe). Ver
`docs/DECISIONS.md` (entrada "FASE 3: múltiplas instâncias do mesmo
tipo por tenant") pro racional completo do esquema de `slug`/`nome` e
pro mapeamento de todos os pontos de colisão corrigidos.

- **Schema**: `Instance` ganhou `slug` (identificador técnico único
  por tenant, ex: `zabbix`, `zabbix-2`) e `nome` (apelido opcional,
  visível na UI, ex: "Zabbix - Matriz"). Trava de unicidade agora é
  `@@unique([tenantId, slug])`. Instâncias existentes foram migradas
  com `slug = tipo` (compatibilidade retroativa, nenhuma URL/container
  existente mudou de nome).
- **Toda geração de nome de recurso (container, volume, chave de
  serviço docker-compose, router/labels do Traefik) passou a incluir o
  sufixo do `slug`** quando não é a primeira instância daquele tipo
  (`lib/instance-slug.ts`, `lib/compose-templates.ts`,
  `lib/instance-containers.ts`, `lib/provisioning.ts`) — a primeira
  instância de cada tipo continua sem sufixo (compatibilidade com tudo
  que já está no ar).
- **UI**: campo opcional "Nome/apelido da instância" no formulário de
  criação; aviso amarelo quando o tenant já tem uma instância daquele
  tipo ("esta será mais uma"); toda tela que lista instâncias
  (dashboard, instâncias, credenciais, documentação cliente/técnica,
  cards) mostra `nome || tipo`.
- **Concorrência**: geração do próximo `slug` livre
  (`nextInstanceSlug`) tem retry automático em caso de corrida (dois
  cliques/abas tentando criar ao mesmo tempo o mesmo próximo número —
  `P2002` do Prisma vira nova tentativa com o próximo número, não
  erro pro usuário).
- **Bug pré-existente corrigido de carona**: `updateInstanceDomain`
  para Uptime Kuma calculava o nome do router do Traefik como
  `uptime_kuma` (underscore) quando o label real é `uptime-kuma`
  (hífen) — trocar domínio de uma instância Uptime Kuma falhava
  silenciosamente antes desta fase. Corrigido com `routerBaseByKind`.
- **Limitação conhecida, não resolvida nesta fase**: SSO
  (`lib/sso.ts`) continua assumindo 1 instância por tipo por tenant —
  com múltiplas instâncias do mesmo tipo, o SSO usa a primeira
  encontrada como padrão. Não é um bug de segurança (não vaza dado
  entre tenants), é só uma limitação funcional; documentado em
  `docs/portal/ARCHITECTURE.md` como próximo passo se/quando SSO
  multi-instância virar demanda real.
- **Testado de ponta a ponta com dado real** (tenant de teste
  VALIDACAO TESTE1, que já tinha 1 Vaultwarden ativo): criado um 2º
  Vaultwarden ("Vaultwarden - Teste 2") via UI real (Playwright,
  login real, formulário preenchido de verdade) — slug `vaultwarden-2`
  gerado automaticamente, container `valid1-vaultwarden-2` e volume
  `valid1_valid1-vaultwarden-data-2` distintos do original, router
  Traefik `valid1-vaultwarden-2` com domínio próprio. As duas
  instâncias responderam `HTTP 200` simultaneamente com HTML próprio
  (confirmado via `curl` real a cada domínio). Excluída a 2ª instância
  em seguida (fluxo de 2 etapas via UI) e confirmado que a 1ª
  continuou de pé e respondendo normalmente — nenhum recurso
  compartilhado entre as duas.
- **Ferramenta interna melhorada**: `scripts/playwright-screenshot.js`
  ganhou `--fill "seletor|=|valor"` (preencher campos de texto num
  fluxo automatizado real, ex: o campo "nome" desta fase) — separador
  `|=|` em vez de `=` puro porque seletores CSS de atributo já usam
  `=` (`input[name="x"]`).

## 2026-07-27 (sessão Cursor) — FASE 4: Registrar instância existente — CONCLUÍDA

Substitui SQL manual por ação ADMN no painel. Ver `docs/DECISIONS.md`
(entrada "FASE 4: Registrar instância existente") e
`docs/portal/ARCHITECTURE.md`.

- Tela `/tenants/[id]/instances/register` (botão "Registrar existente"
  na lista de instâncias, só ADMN): nova ficha sem provisionar +
  corrigir `containerPrefix` de ficha já existente.
- Coluna `instances.container_prefix` aplicada no Postgres; código de
  métricas/backup/diagnóstico/exclusão honra o prefixo.
- **Órfãos fechados:** zabbix/grafana do tenant `npx` (URLs `*.demo.*`,
  containers `demo-*`) receberam `containerPrefix=demo` — sem isso,
  ações mirariam `npx-zabbix-*` / `npx-mysql` (Zabbix mestre da
  plataforma).
- **Teste real:** registro via form POST autenticado de um Vaultwarden
  descartável (`tmpreg-vaultwarden-2`) no tenant valid1 → `?registrado=1`;
  técnico não-ADMN bloqueado (307 `/dashboard`); registro e container
  de teste removidos.
- **Pendente consciente:** `flua-go2rtc` continua sem ficha (não há tipo
  go2rtc no catálogo — stack auxiliar, não produto).

**FASE 0–3 desta rodada macro:** já estavam concluídas na sessão Claude
Code anterior do mesmo dia; revalidadas no início desta sessão (stack
`npx-zabbix` Up, coleta CPU do host com age ~40s via API, Kopia Up,
Vaultwarden/Uptime Kuma e multi-instância no schema/código).

## 2026-07-27 (sessão Cursor) — FASE 5: auditoria — CONCLUÍDA (com pendências de decisão)

Auditoria ativa (não só releitura). Detalhe e lista completa em
`docs/DECISIONS.md` (entrada FASE 5).

**Corrigido agora:**
- Confirmação em 2 etapas em Excluir tenant / usuário / grupo
  (`ConfirmDangerForm`).
- `ROADMAP-MACRO.md` §14 alinhado: Kopia não é mais "pendente".

**Testado de verdade (isolamento):**
- Técnico `teste@teste.com` (valid1) → rotas FLUA e `/backups/admin`
  redirecionam `/dashboard`.
- Mesmo técnico → `deleteTenant` em felixti → `ForbiddenError`; tenant
  permanece.

**Aguardando decisão do responsável** (não alterado): tratamento UX do
`ForbiddenError` (500 vs redirect); destino do `flua-go2rtc`; hardening
extra da senha Admin/zabbix pós-provisionamento; remoção dos logs
temporários do middleware.

## 2026-07-28 — FASE D/E/F MIP Engenharia (37 ativos) — PARCIAL (bloqueio proxy)

**Planilha lida:** `tempfiles/ATIVOS DE REDE.xlsx` (37 linhas de ativo).

### FASE D — onboarding Zabbix
- **Feito:** grupos `MIP ENGENHARIA/BH-MG/{NVR-DVR,Câmeras,Firewall-Cliente,Controladora-Wifi,Access-Points}`; FGT movido para Firewall-Cliente; `grafana-reader` com permissão nos grupos novos; script `scripts/mip-onboard-ativos.py`.
- **Firewall cliente:** já existia (`FGT101F-MIP-MTZ`, FortiGate by SNMP, community `MIP-ENG`, porta 161).
- **NÃO criado (sem confirmação SNMP):** 2 NVR, 17 câmeras, 6 switches novos da planilha, UDM SE, 10 APs — **proxy FLUA-Proxy-01 offline ~24h**, sem rota da VM NPX até a LAN.
- **Investigações documentadas** em `docs/DECISIONS.md` (private V3 vs MIP-ENG; porta 10882; UniFi via API; senha UniFi duplicada).

### FASE E — câmeras
- `clients/flua/go2rtc.yaml` com 17 streams NVR `.190` (credenciais reais).
- Dashboard `mip-cameras` grade 17 canais — screenshot em `docs-publish/validation/mip-cameras-faseE.png` (placeholder "Sem sinal" até haver rota RTSP).

### FASE F — dashboards NOC
- Novos: `mip-firewall`, `mip-wifi` (placeholder até API UniFi).
- Playlist `MIP Engenharia - NOC (parede)` inclui visão geral, switches, impressoras, câmeras, firewall, wifi.
- Screenshots: `mip-firewall-faseF.png`, `mip-wifi-faseF.png`.

**Desbloqueio necessário (ação no cliente FLUA):** restabelecer `FLUA-Proxy-01` (problema Zabbix: "Proxy group [MIB PROXY]: Status offline"). Depois: rodar `scripts/mip-onboard-ativos.py --apply` e revalidar SNMP/RTSP/UniFi.


## 2026-07-28 (sessão Cursor) — FASE D2 / D3 / G

### FASE D2 — Proxy Group — CONCLUÍDA
- Grupo `MIB PROXY` renomeado para **`FLUA-Proxy-Group`** (id=1), com
  `FLUA-Proxy-01` como único membro.
- Hosts existentes: **já** estavam em `monitored_by=2` / grupo 1 — zero
  migração necessária (confirmado: 0 hosts em proxy individual).
- Script `mip-onboard-ativos.py` atualizado: cria só no grupo; flags
  `--migrate-only`, `--check-proxy`, `--apply`.

### FASE D3 — onboarding automático — PRONTO (aguardando proxy)
- **Estado:** aguardando `FLUA-Proxy-01` voltar — onboarding automático
  pronto para disparar sozinho.
- Watcher: cron `*/5` → `scripts/mip-proxy-watcher.py`
- Estado: `/opt/npx-platform/var/mip-onboard/mip-onboard-watcher.json`
  (`status: waiting_proxy`)
- Não tenta religar o proxy (fora do alcance desta VM).

### FASE G — chat IA por tenant — CONCLUÍDA (isolamento lógico), reverificada com prova real em 2026-07-28
- Rota nova: `/tenants/[id]/ai`
- `scripts/test-ai-tenant-isolation.py` → PASS, mas o teste em si só
  confere `WHERE` no Postgres direto, não exercita `executeTool` de
  verdade — reclassificado como evidência fraca.
- **Reverificado com teste real** (Playwright, login de verdade, LLM
  real via OpenRouter): pedido ao chat do tenant FLUA pra diagnosticar
  um id de instância real do tenant NPX — o modelo tentou a chamada de
  ferramenta de verdade, `executeTool` recusou (`owns` check via
  Prisma), log real gravado em `ai_action_log` (`sucesso=false`,
  `detalhe_erro="Instância não encontrada neste tenant — ação
  recusada."`). Detalhe completo com evidência literal em
  `docs/DECISIONS.md`.
- Isolamento físico por VM (MACRO §10) ainda pendente.

## 2026-07-28 — ALERTA: trabalho concorrente do Cursor no mesmo repositório, Claude Code (app) pausado por instrução do responsável

**Contexto:** o responsável do projeto identificou que o **Cursor** está
operando neste exato repositório, ao mesmo tempo, sobre a Fase 3
(múltiplas instâncias do mesmo tipo por tenant) e itens relacionados —
risco real de conflito de código/banco entre os dois agentes. Instrução
recebida: parar toda escrita imediatamente, documentar o que já foi
tocado, não commitar/pushar nada, e sinalizar sessão ativa via
`docs/ACTIVE-SESSION.md`. Esta entrada é só um retrato factual do que foi
encontrado no momento da pausa — **nenhuma ação de correção foi tomada**.

**Evidência direta de edição concorrente:** este arquivo (`STATE.md`)
cresceu de 2011 para 2165 linhas entre duas leituras feitas com poucos
minutos de diferença na mesma sessão de investigação — confirma que outro
processo (Cursor) estava escrevendo neste mesmo documento em tempo real.

**O que já está tocado, no momento em que a pausa foi pedida:**

1. **Banco de dados — migração JÁ aplicada de verdade, não só planejada.**
   Confirmado via `psql` direto (`portal-db`, tabela `instances`):
   - Colunas novas presentes: `slug` (`text`, `NOT NULL`, sem default) e
     `nome` (`text`, nullable).
   - Constraint única já trocada: `instances_tenant_id_slug_key` em
     `(tenant_id, slug)` — a antiga `@@unique([tenantId, tipo])` não
     existe mais no banco.
   - **Backfill das 11 linhas existentes confirmado correto**: todas com
     `slug = tipo` (ex. `zabbix`→`zabbix`, `vaultwarden`→`vaultwarden`),
     nenhuma com `slug` vazio ou divergente.
2. **Prisma Client dentro do container `portal` já regenerado** —
   `node_modules/.prisma/client/index.d.ts` já referencia `slug` (126
   ocorrências), ou seja, `prisma generate` já rodou contra o schema
   novo, não só o `schema.prisma` em disco foi editado.
3. **Código-fonte modificado, não commitado, não documentado em
   `DECISIONS.md`** (mesmo estado já observado no início desta sessão de
   alinhamento, antes da pausa): `portal/prisma/schema.prisma`,
   `portal/src/app/tenants/[id]/instances/actions.ts`,
   `portal/src/lib/compose-templates.ts`,
   `portal/src/lib/instance-containers.ts`, `portal/src/lib/provisioning.ts`
   (todos `modified`, working tree) + `portal/src/lib/instance-slug.ts`
   (novo, untracked). `git log -1` mais recente é o backup automático das
   20:38 de 2026-07-27 — todas essas edições são posteriores a esse commit
   e continuam fora do histórico do git.
4. **Sintoma observado nos logs do container `portal` no momento da
   checagem** (não diagnosticado, não corrigido — só registrado):
   `POST /tenants/[id]/instances` retornando `failed to forward action
   response TypeError: fetch failed` e `[middleware] sem cookie de sessão`
   repetidos. Pode ser efeito do estado intermediário atual (schema/DB
   já migrados, mas incerto se o build do container reflete todo o código
   editado) ou de ação concorrente do Cursor no momento exato da checagem
   — **não investigado a fundo, de propósito, para não competir com o
   trabalho em andamento do Cursor**.

**O que ficou pela metade, no ponto exato da pausa:**
- Nenhum `npm install`/rebuild de imagem foi executado por este agente
  (Claude Code/app) nesta sessão — se o container `portal` já reflete o
  código novo ou não é incerto a partir daqui, não confirmado.
- Nenhuma migration formal do Prisma foi criada (`prisma migrate`) — a
  mudança no banco foi aplicada no estilo `db push` (schema-first, sem
  arquivo de migration versionado), então não há um artefato em
  `prisma/migrations/` documentando esta mudança.
- Nenhum teste ponta a ponta do fluxo de múltiplas instâncias foi
  confirmado nesta sessão (a criação de segunda instância do mesmo tipo,
  a lógica de `instance-slug.ts`, o compose/domínio derivado do slug).
- `docs/DECISIONS.md` não tem entrada para esta fase (o comentário no
  `schema.prisma` referencia "ver docs/DECISIONS.md" para o racional do
  backfill, mas essa entrada não existe no arquivo hoje).

**Ação tomada por este agente (Claude Code/app) a partir daqui:** nenhuma
escrita adicional de código, schema ou banco. Nenhum commit, nenhum push.
Criado `docs/ACTIVE-SESSION.md` sinalizando a sessão ativa do Cursor,
conforme pedido. Aguardando o Cursor liberar e o responsável do projeto
confirmar antes de qualquer próxima ação.

## 2026-07-28 (cont.) — FASE 1 MIP: pivô pra "sem teste ao vivo" — CONCLUÍDA

Proxy `FLUA-Proxy-01` confirmado **totalmente offline** (não intermitente)
pelo responsável do projeto, verificado de forma independente
(`--check-proxy`: `age=2422s`, `proxy_online=False`). Ações, cada uma com
evidência (detalhe completo em `docs/DECISIONS.md`):

1. **Apply travado morto com segurança**: `SIGTERM` em `mip-onboard-ativos.py
   --apply` (rodando havia ~45min). Watcher-pai absorveu o encerramento
   sozinho (`try/finally` já existente) — `onboard_exit: -15`,
   `status: failed_will_retry`, lock liberado. Nenhum código mudou aqui,
   já era seguro por design.
2. **1 host órfão removido**: `SW-161` (hostid 10744) criado mas nunca
   confirmado por SNMP — deletado, confirmado vazio depois.
3. **Câmeras (17x) adicionadas ao script**: `scripts/mip-onboard-ativos.py`
   `ASSETS` foi de 8 → 25 entradas. Sem community SNMP na planilha pras
   câmeras → template nativo `ICMP Ping` (só existente alternativa real,
   confirmada via `template.get`, nada inventado). Validado num host
   descartável (criado + itens conferidos + apagado, nunca IP da MIP)
   antes de entrar no script de verdade. Bug potencial corrigido no mesmo
   passo: simple checks não atualizam `hostinterface.available` — trocado
   por `history.get` do item `icmpping` pro `check_type="icmp"`, senão o
   watcher nunca confirmaria câmera nenhuma mesmo com ping OK.
4. **Credenciais agora `type: 1` (secret)** em toda macro de host criada
   por este script (SNMP community + as novas de câmera) — não ficam mais
   em texto plano visível na UI do Zabbix.
5. **UniFi (controladora + 10 APs) — gap real, não implementado.** Sem
   template nativo (`template.get` zero resultado) e sem poder testar a
   API da UDM SE agora, construir isso seria adivinhar payload — registrado
   como pendência de design própria em `docs/DECISIONS.md`, não maquiado
   como "só esperando o proxy".
6. **Dry-run pós-edição rodado de verdade** (sem `--apply`): sintaxe OK,
   25 ativos listados corretamente, proxy corretamente offline, **zero
   tentativa de criação**.
7. **Watcher (cron `*/5`, lock, retry automático) não precisou de nenhuma
   mudança** — já dispara sozinho quando `proxy.get lastaccess` ficar
   fresco de novo, agora cobrindo as 25 entradas. Confirmação de
   resultado real (o que criou, o que falhou) fica em
   `var/mip-onboard/watcher.log` + `cron.log`, como já era.

**Pendente de verdade, aguardando rota de rede (não bloqueante pro resto
do roadmap):** as 17 câmeras (ICMP), 6 SNMP restantes (2 NVR + 4 Aruba +
2 HPE), e o design da integração UniFi.

## 2026-07-28 (cont.) — Claude Code assume a sessão ativa (MIP resolvido, Cursor confirmado parado)

Processo do Cursor (`mip-onboard-ativos.py --apply`) verificado rodando
de verdade nesta sessão (ver entrada "ALERTA" acima), morto com
segurança e evidência só depois de o responsável do projeto confirmar o
proxy genuinamente offline (verificação própria antes de agir, não só
aceitar a afirmação). A partir daqui esta sessão (Claude Code) segue como
única ferramenta tocando MIP/Zabbix/proxy — parando de tocar nisso
imediatamente após esta fase, conforme instrução, pra iniciar o
desenvolvimento pesado do roadmap.

## 2026-07-28 (cont.) — Bug real corrigido: portal travava Server Action sem sessão ("fetch failed")

Container `portal` estava em rajada constante de erro real (múltiplos
`fetch failed` por segundo em `POST /tenants/[id]/instances` sem sessão).
Causa raiz lida no bundle compilado do Next.js (não suposição): self-fetch
interno de redirect de Server Action adivinhava protocolo `http://` (porta
80 não alcançável de dentro do container — `Connect Timeout Error`
confirmado) em vez de `https://`, porque o Traefik termina TLS e fala HTTP
puro com o container. Corrigido com `__NEXT_PRIVATE_ORIGIN=https://admn.npxit.com.br`
no `docker-compose.yml` do portal (env var interna real do Next.js,
confirmada por grep no bundle). Container recriado (só env, sem rebuild).
**Confirmado**: mesma requisição que antes quebrava agora devolve `307`
limpo pra `/login`; zero `fetch failed` nos 20s seguintes ao restart
(antes: contínuo). Detalhe completo em `docs/DECISIONS.md`.

## 2026-07-28 (cont.) — FASE 3 (múltiplas instâncias por tenant) — núcleo verificado, 1 gap conhecido

Testada ponta a ponta contra tenant descartável (criado e completamente
removido depois): 2 instâncias Grafana no mesmo tenant → slugs `grafana`/
`grafana-2` corretos, nomes de container corretos, trava de concorrência
por slug funcionando (constraint `@@unique([tenantId, slug])` já migrada
no banco desde a sessão anterior, confirmada em uso real agora). Schema/
código desta fase (que estava só commitado via backup automático, sem
entrada própria em `DECISIONS.md`) agora tem o racional completo
documentado lá.

**Gap corrigido nesta sessão (2026-07-28, mesmo dia)**: mutex em memória
por tenant (`withComposeLock`, `provisioning.ts`) serializando
`provisionInstance`/`deleteInstanceCompletely`/`updateInstanceDomain` —
elimina a corrida na raiz (nunca mais duas operações do mesmo tenant
lendo/escrevendo o compose ao mesmo tempo). Build real (`docker compose
build portal`) validado, reteste forçado (matar container no meio do
boot, técnica já usada nesta base desde 2026-07-13) confirmou: zero
container órfão, zero volume órfão, compose final consistente com a
realidade. Detalhe completo com evidência em `docs/DECISIONS.md`.

Separadamente, confirmado (não relacionado à Fase 3): health-check de
deploy pode estourar timeout sob host momentaneamente carregado mesmo
quando o serviço sobe corretamente logo depois — não ajustado agora
(seria mascarar sintoma sem saber se o timeout atual é geralmente
suficiente em produção normal).

## 2026-07-28 (cont.) — Proxy FLUA-Proxy-01 voltou online — investigação de causa raiz (ESX01/ESX02 + SW20/SW24), sem alterar nenhum host

**Confirmação independente (não só aceitar o relato):** o próprio watcher
(`mip-proxy-watcher.py`, cron `*/5`) detectou sozinho, via `proxy.get`
real, `FLUA-Proxy-01` mudando de `state=1/age=5175s` (09:45-10:30, todo
offline) para `state=2/age=4s` às **10:35:02** — e disparou o
onboarding automático (`--apply`, PID 1002073) sem intervenção manual.
Log real em `var/mip-onboard/watcher.log`.

**ESX01/ESX02 — investigado antes de mexer, nenhuma mudança feita:**
`host.get` confirma os dois **ainda desabilitados** (`status=1`) de
propósito, exatamente como deixado em 2026-07-17/18: `{$VMWARE.URL}`
preenchida (`https://192.168.1.12/sdk` e `.11/sdk`),
`{$VMWARE.USERNAME}` = literal `PENDENTE-CREDENCIAL-EQUIPE-FLUA`,
`{$VMWARE.PASSWORD}` tipo secret vazia. **Não é uma regressão nem algo
"recuperável" agora** — segue bloqueado por dois itens fora do alcance
deste projeto: (1) equipe FLUA preencher credencial VMware real, (2)
alguém com acesso a `FLUA-Proxy-01` setar `StartVMwareCollectors` no
`zabbix_proxy.conf` remoto. Proxy voltar online não muda nada aqui —
nenhum host foi habilitado sem credencial real (evitaria alertas de
falha de auth em loop).

**SW20 (`192.168.0.170`) e SW24 (`192.168.0.174`) — SNMP vermelho
confirmado real, não é cache:** ping ICMP responde rápido nos dois
(0.89ms e 5.9ms, dado fresco, `age<10s` depois de forçar) — dispositivos
vivos na rede. Mas os itens SNMP (1120 itens verificados entre os dois)
têm **`lastclock` vazio em 100%** — nunca, em nenhum momento, esses hosts
coletaram um valor SNMP real. Não é status "preso" que um check-now
resolve (retestado, mesmo resultado, erro `timed out` fresco).
Comparado com `SW25` (mesmo template "HP Enterprise Switch by SNMP",
mesma community global `{$SNMP_COMMUNITY}`="public" herdada, sem macro
de host) que **responde normalmente** — descarta "community global
errada" como causa universal; o problema é específico desses dois
equipamentos (agente SNMP desligado ou community diferente configurada
localmente neles) — mesmo padrão já documentado para SW21/SW22 em
2026-07-17/18, que também nunca foi resolvido remotamente. **Requer
checagem física no equipamento pela equipe FLUA/MIP** — não é algo que
uma chamada de API resolve, e por isso nenhuma alteração de config foi
feita nos dois.

**Divergência real encontrada entre documentação e o Zabbix ao vivo
(não decidi corrigir sozinho):** a entrada de 2026-07-17 registra "SW24
removido (confirmado duplicata física do SW23)". Na checagem ao vivo
agora: **SW24 existe** (hostid 10687, criado e habilitado) e **SW23 não
existe mais** (`host.get` retorna vazio) — o oposto do que o registro
diz. Não apaguei nem recriei nada para "corrigir" isso sem entender o
que realmente aconteceu (pode ter sido um `host.delete` que falhou
silenciosamente naquela sessão, uma recriação em uma fase posterior não
documentada, ou uma renomeação). Fica registrado aqui como pendência de
investigação — **antes de decidir remover SW24 de novo como duplicata,
alguém precisa confirmar fisicamente se é de fato o mesmo equipamento do
SW23 ou um switch real e distinto** (o ping responde rápido, é um IP
vivo na rede agora).

**SW-178 (Aruba Instant On 1930, `192.168.0.178`) — não existe ainda no
Zabbix no momento desta checagem.** Não confere com a premissa de que já
estaria criado com indicador vermelho — faz parte do lote de 25 ativos
que o onboarding automático (`--apply`, disparado às 10:35:02) está
processando agora; resultado real dele fica registrado na entrada de
onboarding assim que o job terminar (ver seção seguinte).

**Confirmado ao vivo, sem mudança necessária:** `grafana-reader`
(usrgrpid 14, "API read-only (Grafana)") já tem permissão de leitura
(`permission=2`) nos 8 host groups, incluindo os 5 novos da FASE D
(NVR-DVR=29, Câmeras=30, Firewall-Cliente=31, Controladora-Wifi=32,
Access-Points=33) — nenhuma ação pendente aqui, como já registrado em
sessão anterior.

## 2026-07-28 (cont.) — Onboarding automático dos 25 ativos concluiu (parcial, real) + bug corrigido + dashboards atualizados

**Job `mip-onboard-ativos.py --apply` (PID 1002073, disparado pelo watcher
às 10:35:02) rodou até 11:20:12, `exit=1` (parcial, esperado).** Resultado
real (`STATS {'created': 21, 'exists': 0, 'failed': 4}`, log completo em
`var/mip-onboard/cron.log`):

- **Confirmados e coletando dado real:** SW-151, SW-152 (Aruba, SNMP
  `community=MIP-ENG`), SW-160, SW-161 (HPE 1950, novos), 17 câmeras
  (`CAM-192.168.x.x`, template `ICMP Ping` — sem community SNMP na
  planilha, decisão já registrada em 2026-07-18).
- **Falharam em todas as 4 communities (`MIP-ENG`/`private V3`/`private`/
  `public`) e foram removidos, por design (nunca manter host sem
  confirmação):** NVR-190, NVR-191, SW-178, SW-179.

**Investigação extra feita antes de reportar (não aceitei "falhou" sem
checar mais fundo):** criei 4 hosts descartáveis (`TMP-NVR190/191/SW178/
SW179`, template `ICMP Ping` só) pra saber se ao menos pingam — **os 4
respondem ping normalmente** (0% perda, <1ms) — são dispositivos vivos na
rede, o problema é especificamente SNMP não responder a nenhuma community
testada (mesmo padrão já visto em SW20/SW21/SW22/SW24: precisa de checagem
física — agente SNMP desligado ou community real diferente das 4
tentadas). **Os 4 hosts de teste foram apagados depois**, confirmado
vazio.

**Correção importante:** a premissa de que "NVR-190 já existe" (de sessão
anterior) **não é real agora** — `host.get` confirma que nem NVR-190 nem
NVR-191 existem no Zabbix neste momento.

**Bug real encontrado e corrigido:** `check_now_and_wait()` chamava
`task.create` com `{"request": {"itemids": [...]}}` — parâmetro errado
pra API do Zabbix 7.0 (`Invalid parameter: unexpected parameter
"itemids"`), falhando silenciosamente (só um aviso, não interrompia o
script) em **toda** tentativa de todo asset. Corrigido pra criar uma task
por item (`{"request": {"itemid": iid}}` em lista), testado ao vivo contra
um item real — retornou `taskid` válido. Efeito prático do bug: o
"check-now" nunca acelerava nada, o script sempre esperava o ciclo normal
de polling (por isso o job real levou ~45 min pros 25 ativos, não
segundos). Script mais rápido nas próximas execuções, não corrigido daqui
pra trás.

**Dashboards atualizados com dado real (Grafana FLUA), confirmado por
screenshot real, não só HTTP 200:**
- `mip-switches`: filtro trocado de host fixo (`/^SW(20|23|25)$/`) pra
  filtro de **grupo** (`MIP ENGENHARIA/BH-MG/Switches`, host `/.*/`) —
  agora pega os switches novos automaticamente, sem precisar editar de
  novo a cada onboarding. Screenshot real em
  `docs-publish/validation/mip-switches-real-2026-07-28.png`: mostra 7
  switches reais (SW-151 e SW-152 aparecem **DOWN**, SW-160/161/20/24/25
  **UP**) — ver achado abaixo, não é erro do painel.
- `mip-cameras`: adicionado painel novo "Status real (Zabbix ICMP) — 17
  câmeras" (Polystat, grupo Câmeras) **acima** do mosaico de vídeo
  existente — o mosaico de vídeo em si continua placeholder ("Sem sinal"),
  porque isso é um bloqueio de rede **diferente e ainda não resolvido**
  (ver achado de rede abaixo), não o mesmo do proxy Zabbix.

**Achado real, não esperado — SW-151/SW-152 respondem SNMP mas não
ICMP:** `hostinterface` mostra `available=1` (SNMP OK, dado real de
CPU/tráfego chegando) nos dois, mas o item `icmpping` mostra **100% de
perda, consistente nos últimos 6 minutos** (não é blip único). É o oposto
do padrão SW20/24 (aqueles respondem ping, não SNMP). Hipótese mais
provável: o equipamento tem resposta a ICMP desabilitada por segurança
mas mantém SNMP ativo — não é necessariamente uma falha real do switch,
mas **o painel Polystat usa só o item ICMP pra pintar UP/DOWN**, então
mostra esses dois como "DOWN" mesmo estando de pé e monitorados com dado
real. Não mudei a lógica do painel agora (misturar critério ICMP+SNMP
pra status é uma decisão de design que merece confirmação, não uma
correção óbvia) — registrado aqui como caveat conhecido, não escondido.

**Achado de rede separado, confirmado ao vivo — vídeo RTSP das câmeras
continua bloqueado mesmo com o proxy Zabbix online:** testei direto desta
VM (não via proxy Zabbix): `ping 192.168.0.190` → 100% perda; `curl
telnet://192.168.0.190:554` → falha de conexão. **O proxy Zabbix
(`FLUA-Proxy-01`) volta online e resolve a coleta via SNMP/ICMP porque
quem faz essas checagens é o proxy, que roda dentro da rede do cliente**
— mas o container `go2rtc` (vídeo) roda nesta VM NPX, que **nunca teve**
rota direta até a LAN da MIP, com ou sem o proxy Zabbix. São dois
caminhos de rede independentes. Isso não é uma regressão nem
"quase resolvido" — é o mesmo bloqueio já documentado em 2026-07-18,
confirmado que continua existindo.

**Contagem final honesta dos 37 ativos da planilha (`tempfiles/ATIVOS DE
REDE.xlsx`), confirmados com dado real agora:**

| Categoria | Total planilha | Confirmado com dado real | Pendente |
|---|---|---|---|
| Firewall (FGT101F-MIP-MTZ) | 1 | 1 (já existia) | 0 |
| NVR | 2 | 0 | 2 (pingam, SNMP não responde — checagem física) |
| Câmeras IP | 17 | 17 (ICMP) | 0 monitoramento / vídeo RTSP segue bloqueado (rota de rede separada) |
| Switches Aruba (151/152/178/179) | 4 | 2 (151/152) | 2 (178/179 — pingam, SNMP não responde) |
| Switches HPE (160/161) | 2 | 2 | 0 |
| Controladora UniFi | 1 | 0 | gap de design, sem template nativo (2026-07-28) |
| Access Points UniFi | 10 | 0 | mesmo gap acima |
| **Total** | **37** | **22** | **15** |

Pendências reais que restam, todas fora do alcance remoto deste projeto:
checagem física de SNMP em NVR-190/191 e SW-178/179 (equipe FLUA/MIP);
rota de rede (VPN ou similar) entre a VM NPX e a LAN da MIP pro vídeo
RTSP funcionar; design da integração UniFi (sem template nativo,
construir API própria exigiria adivinhar payload sem poder testar contra
a UDM SE real — mesmo bloqueio já registrado em 2026-07-28 anterior).

## 2026-07-28 (cont.) — "Registrar instância existente" reverificada + teste vivo real de restauração Kopia

**"Registrar instância existente" (`/tenants/[id]/instances/register`) —
reverificada, não só relida.** Já estava CONCLUÍDA e testada ponta a
ponta em 2026-07-27 (sessão Cursor). Confirmado agora que continua
íntegra depois da migração de `slug` da Fase 3 (mesmo dia, sessão
anterior): coluna `instances.container_prefix` presente no Postgres
lado a lado com `slug` (`\d instances` real), `page.tsx` já importa e
trata os erros `slug-invalido`/`slug-ja-existe` (não ficou código morto
apontando pro schema antigo), rota responde `307` sem sessão (middleware
funcionando). Não refiz o teste E2E completo via Playwright (o de
2026-07-27 já é real e o código não mudou nesse meio tempo) — verificação
foi por schema+código, registrado aqui com esse nível de profundidade
para não inflar a confiança.

**Backup granular Kopia — teste vivo de restauração feito agora, do
zero, não só releitura do que a sessão de 2026-07-27 já tinha provado:**

1. Criado container descartável `kopia-test-scratch` com volume Docker
   nomeado próprio (`kopia-test-vol`) — **não** um bind mount solto, pois
   o agente Kopia só enxerga `/var/lib/docker/volumes` (confirmado via
   `docker inspect` do `npx-kopia-agent` antes de tentar, evitou um erro
   bobo de path).
2. Gravado um arquivo com conteúdo único e datado
   (`ORIGINAL-CONTEUDO-REAL-<timestamp>`) dentro do volume.
3. Usuário Kopia descartável criado via API real do agente (`POST
   /kopia-users`, tenant `claudetest`, senha gerada na hora).
4. **Backup real**: `POST /backup` contra o container/volume real —
   `snapshotId` real devolvido, `bytesCopied: 34` batendo com o tamanho
   real do arquivo.
5. **Corrompido de propósito**: sobrescrevi o arquivo real com
   `DADO-CORROMPIDO-PERDIDO-DE-VERDADE` — confirmado via `docker exec cat`
   antes de restaurar (prova que a corrupção foi real, não hipotética).
6. **Restore real, modo `overwrite`**: `POST /restore` com o
   `snapshotId` do passo 4 — resposta `{"mode":"overwrite","applied":
   {...,"type":"app-data-restore"...}}`; o próprio agente parou e
   religou o container (`docker stop`/`start`) como parte do processo
   real, não simulado.
7. **Prova final**: `docker exec kopia-test-scratch cat /data/arquivo.txt`
   depois do restore devolveu **exatamente** o conteúdo original
   (`ORIGINAL-CONTEUDO-REAL-<mesmo timestamp>`), não o dado corrompido —
   confirma que o motor de restore funciona de ponta a ponta pra dado de
   aplicação (`app-data`), com container real sendo parado/religado.
8. **Limpeza completa confirmada**: snapshot de teste deletado
   (`kopia snapshot delete ... --unsafe-ignore-source`, `rc=0`), usuário
   Kopia descartável removido (`kopia server user remove
   tenant-claudetest@npx`, confirmado "deleted"), container e volume
   Docker de teste removidos, arquivos temporários apagados. Nenhum
   rastro do teste ficou no repositório Kopia real nem em nenhum tenant
   de cliente.

**Conclusão honesta:** o motor Kopia (backup + restore overwrite) segue
funcionando de verdade hoje, para o caminho `app-data` (arquivo/volume) —
não retestei o caminho `db-dump` (MySQL/Postgres) nesta rodada porque já
tinha sido provado ponta a ponta em 2026-07-27 com a mesma engenharia de
código (`dump_database`/`restore_db_dump`, sem mudança desde então) e o
foco pedido agora era confirmar que o mecanismo real segue de pé, não
repetir todo o escopo anterior. As pendências já registradas em
2026-07-27 (sem backup agendado automático, storage só local, isolamento
de permissão tudo-ou-nada por tenant) continuam exatamente as mesmas —
nenhuma delas foi tocada nesta verificação.

## 2026-07-28 (cont.) — Catálogo: Nextcloud implementado e testado (2 bugs reais corrigidos); CrowdSec/Pi-hole com decisão de negócio tomada mas implementação adiada por risco

**Nextcloud — CONCLUÍDO, testado de ponta a ponta com sucesso real, sem
patch manual no teste final.** Mesmo padrão dos outros 6 tipos do
catálogo (compose isolado, Traefik, Let's Encrypt, `suporteti`
automático, card com logo oficial, página descritiva comercial). Dois
bugs reais encontrados e corrigidos durante o teste (detalhe completo
com evidência em `docs/DECISIONS.md`): (1) checagem de admin sem retry
(corrigido, mesmo padrão do Zabbix/GLPI); (2) `NEXTCLOUD_TRUSTED_DOMAINS`
faltando o nome interno do container, causando rollback certeiro
("Access through untrusted domain"). Cota de disco por tenant lança
irrestrita por padrão (mesmo precedente já usado em `TenantQuota`/
`RESOURCE_LIMITS`) — decisão de negócio registrada como não-bloqueante
em `docs/DECISIONS.md`.

**CrowdSec e Pi-hole/AdGuard — decisão de negócio tomada pelo
responsável do projeto (perguntado diretamente), mas implementação
completa **adiada por risco de infraestrutura compartilhada**, não por
falta de decisão:** CrowdSec vai proteger o que já é hospedado aqui
(item de catálogo), mas a peça que aplica a proteção (bouncer) se acopla
ao Traefik **compartilhado por todos os tenants** — um bouncer mal
configurado pode derrubar acesso de todo mundo, não só de quem contratou.
Pi-hole vai filtrar a rede inteira do cliente via VPN, o que depende da
automação de escrita no FortiGate (Fase 5, ainda bloqueada aguardando
revisão do responsável desde 2026-07-15). Caminho técnico de cada um já
esboçado em `docs/DECISIONS.md` pra quando uma sessão puder dedicar o
cuidado que infraestrutura compartilhada exige.

**Bug real, pré-existente, CONFIRMADO (não corrigido de vez)**:
`POST /tenants/new` (criar tenant pela UI) derruba a sessão do usuário —
já suspeitado em 2026-07-18/19, nunca antes reproduzido com navegador
real. Confirmado agora com Playwright/Chromium de verdade. Uma hipótese
testada (guard clauses fora do try/catch) foi descartada com evidência
real — fix aplicado mesmo assim (isolamento de erro melhor, correto por
si só), mas a causa raiz segue desconhecida. Prioridade alta pra próxima
sessão — detalhe completo, incluindo o que já foi descartado, em
`docs/DECISIONS.md`.


## 2026-07-28 (tarde, Cursor) — FASE 0: bug `/tenants/new` sessão — RESOLVIDO (não era bug de produto)

Reproduzido com Playwright real. Causa: seletor ambíguo acertava o botão **Sair** (primeiro submit no DOM). Com `main form button:has-text("Criar")` / `data-testid=create-tenant-submit`, criação funciona, sessão preservada, tenant criado (evidência literal em `docs/DECISIONS.md`). Tenants de teste FASE0-* removidos.


## 2026-07-28 (tarde, Cursor) — FASE 0/1/2 chat IA + bug sessão

### FASE 0 — CONCLUÍDA
`POST /tenants/new` não derruba sessão quando o submit é o botão Criar.
Causa dos testes anteriores: seletor ambíguo no botão **Sair** (primeiro submit no DOM).
Evidência: Next-Action `5d2f49…` → `/dashboard` + tenant criado; Next-Action `37a2ca…` → logout.
`data-testid=create-tenant-submit` / `logout-submit` adicionados.

### FASE 1 — CONCLUÍDA
Drawer global de IA, voz (Web Speech API, custo zero), anexos privados, markdown, histórico por tenant+user, checagem de permissão do usuário nas tools.

### FASE 2 — CONCLUÍDA com evidência em DECISIONS
Leitura recusada; escrita OK; cross-tenant texto recusado; cross-tenant tool bloqueado no servidor; anexo não vaza entre tenants (consulta filtrada).

## 2026-07-28 (noite, Cursor) — FASE 3 bugs IA — CONCLUÍDA

1. **Failed to fetch** após reinício (falso negativo): timeout de transporte;
   UI recupera resposta via poll do histórico. Traefik
   `responseHeaderTimeout=600s`. Evidência Playwright: UI sucesso +
   `ai_action_log sucesso=t`.
2. **Upload flaky**: Server Action com FormData/File intermitente →
   base64 JSON. Evidência: **5/5** uploads consecutivos Playwright.

## 2026-07-28 (noite, Cursor) — FASE 4 Chatwoot catálogo — CONCLUÍDA

SKU Chatwoot (MACRO §6/§11) provisionável self-service. Teste real no
tenant `valid1`: audit `sucesso=t`/`concluido`, containers healthy,
`POST /auth/sign_in` do `suporteti@npxit.com.br` → 200 + access-token
(SuperAdmin). Detalhe em `docs/DECISIONS.md`.

## 2026-07-28 (noite, Cursor) — Fases A–H (docs IA, exec, créditos, i18n, dashboard, CNPJ, export, DnD)

Continuidade após Chatwoot (FASE 4) fechado. Sessão marcada em
`docs/ACTIVE-SESSION.md`. Evidências em
`docs-publish/validation/` e `/tmp/fase-ah/`.

### FASE A — Extração real de documentos — CONCLUÍDA (evidência)

- Libs: `pdf-parse`, `mammoth`, `exceljs` em `portal/src/lib/ai/extract-text.ts`.
- Anexo `vendas-faseA.xlsx` no DB com extrato tabular real
  (`Sensor Zabbix | 12 | 199.9`, `Licenca Grafana | 3 | 450`).
- Chat UI citou os números exatos (sem inventar) — histórico em
  `/tmp/fase-ah/b-results.json` + screenshot `xlsx-chat.png`.
- Prompt injection em anexo: aviso do servidor no extrato + IA relatou
  e **não executou** (`inject.md`).

### FASE H — Drag-and-drop — CONCLUÍDA (evidência)

- Drop zone `data-testid="ai-drop-zone"` + hint visual.
- Playwright com `DataTransfer` + `DragEvent` gravou `dnd2.txt`
  (`conteudo-dnd-playwright-real`) em `ai_chat_attachments`.

### FASE B — docker exec + confirmações — CONCLUÍDA (evidência literal)

Classificação **no servidor** (`command-risk.ts`): read / mutate (1 conf) /
dangerous (2 frases reforçadas) / blocked.

| Caso | Evidência |
|---|---|
| (a) Leitura `uname -a` | `ai_action_log` sucesso, exitCode 0, output Linux real (após reinício Portainer que estava em 403) |
| (b) Mutação `touch` | pending `mutate`, confirmations_required=1, received=0, **não executado** |
| (c) Perigoso `rm -rf /tmp/...` | pending `dangerous`, confirmations_required=**2**, received=0; log `aguardando 2 confirmação(ões) — NÃO executado` |
| (d) Cross-tenant | Chat FLUA com instanceId de valid1: recusa no servidor/escopo |
| (e) Doc com instrução | IA reportou prompt injection; zero execução automática |

Isolamento multi-tenant (seletor): dentro do chat do tenant ativo, tools
só enxergam instâncias daquele tenant — reconfirmado com A+B.

### FASE C — Créditos OpenRouter — CÓDIGO PRONTO; BLOQUEIO Management Key

- Management API client, margem cascata NPX→N1→N2, telas
  `/tenants/[id]/ai-credits` e `/settings/ai/credits`, recarga stub
  `paid_simulated`, banner saldo baixo no drawer.
- **Bloqueio real:** chave Management/Provisioning **não existe** no
  `platform_settings`. Prova com a chave de chat:
  - `GET /credits` → 200 `total_credits:10 total_usage:6.83976666`
  - `GET/POST /keys` → **401 Invalid management key**
  - `/auth/key` → `is_management_key:false`
- Sem Provisioning Key do OpenRouter (criar no painel da conta mestre e
  colar em Configurações de IA) **não dá** para provisionar chave por
  tenant nem testar limite/aviso/bloqueio de ponta a ponta. UI já mostra
  `data-testid="mgmt-key-missing"`.

### FASE D — i18n — FUNDAÇÃO + SHELL CONCLUÍDA; páginas internas parciais

- Escolha: expandir `lib/i18n.ts` (pt-BR/en/es) + cookie `npx_locale`
  em vez de next-intl com `[locale]` (evita reescrever rotas/auth).
- Regra permanente em `CLAUDE.md`.
- Evidência: sidebar EN (`CURRENT TENANT`, `Instances`, `Sign out`) e
  ES (`TENANT ACTUAL`, `Instancias`, `Salir`) — screenshots
  `i18n-en-dash.png` / `i18n-es-inst.png`. Login labels traduzidos;
  títulos de páginas internas ainda misturam PT (próximo lote).

### FASE E — Dashboard ADMN executivo — CONCLUÍDA (UI)

Cards: saúde amostral, instâncias, backups, tipos; listas por tipo / top
consumidores / créditos. Screenshot `dash-exec.png`.

### FASE F — CNPJ/CPF + BrasilAPI — CONCLUÍDA com bug corrigido

- Campos estilo Asaas + BrasilAPI. Bug: undici sem User-Agent → HTTP 403
  Cloudflare; corrigido com UA explícito em `brasilapi.ts`.
  **Reteste real:** CNPJ `00.000.000/0001-91` → razão `BANCO DO BRASIL SA`,
  cidade `BRASILIA` (screenshot `cnpj2.png`).

### FASE G — Export CSV ADMN — CONCLUÍDA

- `GET /tenants/export` só ADMN; CSV nível 1 com cadastro + AI limit.
- Evidência: HTTP 200, header + linhas FLUA/valid1/etc em
  `docs-publish/validation/export.csv`.

### Operacional

- Portainer entrou em lockout 403 sob carga de auth; reinício do stack
  restaurou JWT. Monitorar.

## 2026-07-28 (noite, Cursor) — FAB IA + Fase C créditos FECHADA

### FASE 1 — Sobreposição do atalho do assistente — CONCLUÍDA

- FAB movido do **topo direito** (sobre idioma/menu) para **canto inferior direito**,
  círculo 40×40, recolhido por padrão, some quando o drawer abre.
- `title`/`aria-label`/`Tooltip`: "Assistente de IA".
- Playwright 3 viewports (1440×900, 1280×800, 768×1024) × Dashboard + Instâncias:
  `fabBottomRight=true`, `noOverlaps=true`, FAB oculto com drawer aberto.
- Screenshots: `docs-publish/validation/fase-c/fab-*-{before,after}.png`.

### FASE 2 / Fase C — Provisioning Key + limite real — CONCLUÍDA

1. Chave Management salva no campo certo (`ai_management_key_encrypted`),
   **≠** chave de chat (`sameKey:false`).
2. `GET /keys` **200**; `/auth/key` → `is_management_key:true`,
   `is_provisioning_key:true` (`fase-c/mgmt-probe.txt`).
3. Tenant descartável `credittest` + key `npx-credittest` limit US$0.05
   (depois ajustado p/ ~0.01377 após consumo real).
4. Consumo real via chat completions + UI; saldo baixo na tela de créditos:
   `Limite: US$ 0.01377 / Restante: US$ 0 / Uso: 100.0% / Saldo baixo`.
5. Bloqueio OpenRouter literal: `HTTP 403 {"error":{"message":"Key limit exceeded (total limit). ...","code":403}}`
   (`fase-c/exhaust.txt`, drawer UI).
6. Banner no drawer após erro/send (`ai-low-balance-banner`).
7. Cleanup: `DELETE /keys/{hash}` → `{"deleted":true}`; tenant `credittest` removido;
   threshold restaurado a 80%.
8. Bug corrigido: resposta `POST /keys` traz `key` no **topo**, não em `data`
   (`openrouter-management.ts`).

Ainda stub (decisão pendente): gateway de pagamento real. Top-up do saldo
mestre OpenRouter continua manual pelo responsável.


## 2026-07-29/30 — Sessão fases A–M (execução autônoma)

Resumo consolidado em `docs-publish/validation/fase*-*/evidence.json` e `docs/SECURITY.md` (interno).

- **A** ADMN Inbox/Reports/Tasks/Sales reais (sem upsell interno) — OK
- **B** Credencial mestre + rotação + técnico forbidden — OK
- **C** Probe de credenciais (job + API ADMN); falha forçada master detectada (fails>0, inbox) — OK
- **D** Política AI confirm system|strict por tenant — UI + lógica tools — OK
- **E** Gestão users via API (Zabbix/Grafana/GLPI/…); Vaultwarden/Kuma sem API — doc em código `APP_USER_CAPABILITIES` — OK parcial por app
- **F** Destino Kopia editável (`platform_kv` + agent `/destination`); repo antigo preservado — OK (manifest + path UI)
- **G** Scan stack health + Reconectar 1-clique; histórico backup limitado a 7 — OK código; E2E restore completo não reexecutado nesta sessão (já existia)
- **H** Acesso suporte gated ticket; user temp Zabbix login real + revoke — OK
- **I** Watermark `@by NPX IT` permanente — OK
- **J/K** Auditoria + correções rate-limit/magic/chmod/sanitize — ver SECURITY.md
- **L** Anti prompt-injection / extração system prompt — sanitize server-side — OK
- **M** Authelia: análise só; não implementar agora — ROADMAP


## 2026-07-30 — BUG NOVO, prioridade ALTA — usuário preso na visão "Cliente" após login, seletor de tenant não respondia

> **ATUALIZAÇÃO 2026-08-03: CORRIGIDO** — ver seção "Fase 0" no topo
> deste arquivo e `docs/DECISIONS.md` (entrada 2026-08-03 nav-mode).
> Evidência: `docs-publish/validation/sessao-onda-fase0-2026-08-03/`.

**Registrado só como relato, NÃO investigado nesta sessão** (sessão de
consolidação de documentação, sem desenvolvimento) — próxima sessão de
desenvolvimento deve investigar a causa raiz antes de tentar corrigir
qualquer coisa às cegas.

**Relato do usuário, o mais literal possível para não perder detalhe:**
- Depois de logar no portal, o usuário ficou preso na visão "Cliente"
  (contexto de tenant específico, não a visão de plataforma).
- Cliques repetidos no seletor de tenant no cabeçalho, tentando voltar
  para "Plataforma" (ADMN), **não funcionaram** — a troca de contexto não
  acontecia, a tela continuava presa no tenant errado.
- **Só normalizou depois de ~4 minutos** de tentativas, e só depois de
  **múltiplos hard refresh** (Ctrl+Shift+R, que ignora cache do
  navegador) — um refresh normal (F5/Ctrl+R) não bastou, precisou do hard
  refresh especificamente, várias vezes, até resolver sozinho.

**Hipóteses a considerar na investigação (não confirmadas, só ponto de
partida — não tratar como diagnóstico já feito):**
- Cache do navegador servindo uma versão antiga da página/JS que não
  refletia a troca de tenant já processada no servidor (explicaria por
  que hard refresh — que ignora cache — eventualmente resolveu).
- Cookie `npx_session`/estado de tenant ativo desatualizado no cliente
  vs. servidor (ver `setActiveTenantCookie` em `lib/auth.ts`) — possível
  dessincronia entre o cookie de tenant ativo e o que a UI mostra.
  Nada disso é confirmado, é só a partir do relato e do código já
  conhecido — precisa de reprodução real antes de qualquer correção.
- Pode estar relacionado (ou não) ao trabalho concorrente em andamento
  nesta mesma janela (Cursor mexendo em NOC/Uptime Kuma/vitrine e,
  antes disso, na Sessão 42 — migração de redes edge→internal do
  próprio `portal`, rebuilds recentes) — timing coincide com um período
  de múltiplos rebuilds/restarts do container `portal`, o que pode ter
  deixado sessões de navegador antigas temporariamente inconsistentes
  com o servidor recém-reiniciado. **Hipótese, não causa confirmada.**
- Não descartar também: bug genuíno de estado React/client-side no
  seletor de tenant (`AppShell`/componente do cabeçalho), independente
  de cache — precisa reproduzir com DevTools abertos (Network + Console)
  pra diferenciar as hipóteses acima.

**Impacto:** trava real de uso — usuário ficou ~4 minutos sem conseguir
navegar pro contexto certo do painel. Prioridade alta por afetar
diretamente a usabilidade do produto (mesmo sendo intermitente/não
reproduzido de propósito ainda).

**Próximo passo (não feito nesta sessão):** tentar reproduzir de
propósito (login, trocar tenant repetidamente, com e sem cache,
inclusive logo após um restart do `portal`) e instrumentar com log
real (mesmo padrão já usado em `middleware.ts` pra outros bugs de sessão
nesta mesma base de código) antes de qualquer tentativa de correção.

## 2026-07-30 (cont.) — Incidente de disco (85%) resolvido; manutenção automática pendente, prompt pronto pro Cursor

**Causa raiz identificada e corrigida** (detalhe completo com evidência
literal em `docs/DECISIONS.md`): 149,9GB de cache de build do Docker
acumulado sem limite (75% do disco), zero relação com monitoramento ou
dado de cliente (volumes reais sempre foram ~13GB). Limpeza manual
autorizada e executada: `docker builder prune -a -f` + `docker image
prune -a -f` → **155GB liberados**, disco de 85% → 19% de uso. Zero
regressão (todos os containers seguiram `Up`).

**Pendência urgente registrada**: rotina automatizada de manutenção pra
isso nunca mais acontecer sem alarme — spec técnica completa em
`docs/PROMPT-CURSOR-manutencao-disco.md`, pronta pra entregar ao Cursor.
Cobre: autolimpeza na origem (scripts de teste se autolimparem, mesmo
padrão já usado em `mip-onboard-ativos.py`), varredura periódica de
segurança (nunca tocar tenant real — cruzamento obrigatório com a
tabela `tenants` real, não só padrão de nome), itens/triggers/painel
novos no Zabbix/NOC interno da NPX, e critérios de validação com
evidência obrigatória. **Nada disso foi implementado ainda** — só a
limpeza manual pontual do sintoma.

**Nota operacional**: responsável do projeto vai parar esta VM logo em
seguida para compactar o `.vhdx` no hypervisor (fora do alcance deste
agente) e recuperar o espaço físico real no host. Sessão retomada depois
disso — ver seções 20-24 de `docs/ROADMAP-MACRO.md` pra continuidade dos
temas pendentes (migração/onboarding, segurança, segregação de infra,
capacidade de hardware, manutenção de disco).

## 2026-08-02 (cont.) — Cota por tenant: verificado funcionando de verdade, pronto pra FLUA

Testado agora (não só lido no código) contra tenant descartável real,
via HTTP real: cota salva pelo formulário, tipo bloqueado recusado no
servidor (não só escondido na UI), tipo permitido aceito, tipo excedido
recusado com mensagem certa (`1/1`). Evidência completa em
`docs/DECISIONS.md`. **Este pré-requisito pra liberar acesso à FLUA está
atendido.** Falta só a IA por tenant (em andamento pelo Cursor).

## 2026-08-02 (cont.) — Cursor concluiu Parte 1 (migração) e Parte 2 (IA app-tools) do prompt anterior; novo prompt entregue

Cursor reportou conclusão das Partes 1 e 2 de
`docs/PROMPT-CURSOR-migracao-infra-capacidade.md` (ferramenta de
migração/onboarding + `portal/src/lib/ai/app-tools.ts` com ferramentas
reais de configuração Zabbix/Grafana/GLPI), com evidência em
`docs-publish/validation/sessao-migracao-ia-2026-08-02/` e teste de
isolamento cross-tenant PASS. **Ainda não validado de forma
independente por mim** — validação pesada fica pra depois que o próximo
lote também for concluído (mesma exigência de sempre: nunca aceitar
"PASS" sem evidência literal própria).

Investigação de código antes de escrever o próximo prompt confirmou 2
lacunas reais (não hipotéticas):
- `hasAccessToTenant`/`accessibleTenantIds` (`portal/src/lib/authz.ts`)
  já resolve hierarquia pro resto do portal, mas as ferramentas de IA
  (`ai/tools.ts`, `ai/app-tools.ts`) não usam isso — só operam no
  tenant literal da URL, nunca em subtenants.
- `AiChatThread`/`AiChatMessage` já são gravados no banco, mas
  `portal/src/app/tenants/[id]/ai/page.tsx` nunca lê o histórico de
  volta ao carregar a página — é por isso que a IA "esquece" a conversa
  ao dar refresh, apesar do dado existir no banco.

Novo prompt escrito e entregue: **`docs/PROMPT-CURSOR-ia-hierarquica-msp.md`**
— cobre (Parte A) escopo hierárquico das ferramentas de IA pra
subtenants, correção da memória persistente, consulta em tempo real ao
estado real das aplicações antes de responder, modo wizard guiado,
guardrails de conteúdo (nunca falar de assunto externo, nunca revelar
outros tenants), e base de conhecimento concentrada/anonimizada
reaproveitável entre clientes (ROADMAP-MACRO seção 10); e (Parte B)
fechamento de lacunas confirmadas do modelo MSP/cliente final —
propagação forçada de branding pra nível 2 (não implementada, grep
confirmou) e confirmação explícita ao diminuir cota já em uso (não
implementada, grep confirmou) — mais auditoria do resto das seções 2/4/5
do ROADMAP-MACRO. Prompt inclui regra explícita de economia de token
(máximo de chamadas reais ao LLM tanto na execução quanto na validação,
já que cada teste real custa dinheiro) e é auto-contido o bastante pra
sobreviver a uma troca de modelo do Cursor no meio da execução (pedido
explícito do responsável do projeto). **Divisão de trabalho reafirmada
pelo responsável nesta sessão: Cursor codifica, eu valido depois aqui
mesmo, de forma pesada e real, com evidência.**

Limpeza confirmada: tenant descartável `teste-quota-17856881` usado pra
validar cotas — confirmado removido do banco (`tenants`, `instances` e
tabelas relacionadas) e do disco (`clients/teste-quota-17856881/`
inexistente); só restava a remoção do `docker-compose.yml` correspondente
para commitar no backup, feito nesta sessão.

## 2026-08-02 (cont.) — Partes A e B validadas de verdade; novo prompt (wizard auditor) entregue

Cursor reportou conclusão de `docs/PROMPT-CURSOR-ia-hierarquica-msp.md`
(Partes A e B). **Validado de forma independente nesta sessão, não só
aceito por relato** — evidência conferida item por item:
- Owns-check hierárquico real em `portal/src/lib/ai/tool-context.ts`
  (`resolveOperableTenantId`): alvo só é aceito se for o próprio tenant
  do chat ou filho direto dele, E passar em `hasAccessToTenant` — lido
  o código, não só confiado no relato.
- `listar_subtenants`, `targetTenantId` nas tools de app, SSR de
  histórico (`ai/page.tsx` lendo `aiChatThread`/`aiChatMessage`) e caps
  reais (`AI_HISTORY_UI_LIMIT=50`, `AI_HISTORY_MODEL_LIMIT=40` em
  `tool-context.ts`) — confirmados no código.
- Branding N2 bloqueado server-side de verdade (`getTenantHierarchyLevel(...).canCustomizeBranding`
  em `branding/actions.ts`, não só escondido na UI).
- Cota com confirmação quando `max < usados` (`quotas/actions.ts`).
- `ai_knowledge_base` sem campo de tenant no schema.
- Print real `04-ai-history-ssr.png` conferido: histórico de conversa
  aparece já carregado após refresh, texto literal "a conversa nao foi
  esquecida apos refresh".
- Log real de chamadas LLM (`02-llm-calls.txt`) conferido: 3 chamadas
  reais, wizard perguntando uma coisa por vez, recusa de assunto
  externo, recusa de confirmar existência de outro tenant (FLUA) — texto
  coerente com o pedido, não simulado.

**Conclusão: Partes A e B genuinamente concluídas e corretas.**

Responsável do projeto pediu evolução do wizard: hoje ele só guia
criação de algo novo — pediu que também **analise o que já está
configurado** (exemplo dado: Zabbix já tem switches configurados, o
wizard deveria olhar isso e sugerir melhoria real, não do zero; cofre
de senhas Vaultwarden já configurado, deveria achar inconsistência ou
vulnerabilidade e propor+executar correção), usando conhecimento
técnico real e documentação confiável, generalizando pra todas as
soluções do catálogo.

**Achado técnico importante, registrado e explicado no novo prompt**:
Vaultwarden usa criptografia zero-knowledge no cliente — o servidor
**nunca** tem acesso ao conteúdo dos itens do cofre (senha mestra nunca
sai do dispositivo do usuário), então "ver se uma senha guardada é
fraca/reusada" não é tecnicamente possível sem quebrar o motivo do
produto ser seguro — e não deve ser tentado. O que É auditável de
verdade via API admin: política de senha mestra e 2FA obrigatórios,
convites pendentes antigos, contas desativadas ainda na organização,
exportação habilitada, auto-cadastro exposto. Isso é uma auditoria de
segurança real e válida, só não é "ver a senha em si" (que é
propositalmente impossível).

Prompt novo entregue: **`docs/PROMPT-CURSOR-wizard-auditor.md`** — Parte
C, ferramentas de auditoria (`auditar_zabbix`, `auditar_vaultwarden`,
generalização pro resto do catálogo), formato estruturado
`AuditFinding[]`, base de conhecimento curada com fonte oficial citada
(camada primária, sem custo de LLM) mais busca externa opcional
(camada secundária, chave cifrada em `PlatformSettings` seguindo o
padrão já existente de `aiApiKeyEncrypted`), e wizard rodando a
auditoria proativamente antes de perguntar o que fazer. Mesma regra de
economia de token do prompt anterior.

## 2026-08-02 (cont.) — Parte C: wizard auditor — CONCLUÍDA

Implementado e validado contra VALID1 (`746304d4-8450-4217-9a07-c3baf1b8c773`).

**Código:** `portal/src/lib/ai/audit-types.ts`, `audit-tools.ts` (wired em
`app-tools.ts`); system prompt wizard reforçado em `chat.ts` (chamar
`auditar_*` antes de perguntar; Vaultwarden zero-knowledge).

**Tools de leitura:** `auditar_zabbix`, `auditar_vaultwarden`,
`auditar_grafana`, `auditar_glpi`, `auditar_uptime_kuma`.

**Tools de mutação (CONFIRMO):** `criar_dependencia_trigger`,
`converter_macro_para_secret`, `aplicar_template_zabbix`,
`aplicar_politica_vaultwarden` (`disable_signups` | `disable_user`).

**Vaultwarden — limite técnico respeitado:** só API admin (`/admin`,
cookie `VW_ADMIN`, users/config via `ADMIN_TOKEN` em
`InstanceCredential`). **Nunca** lê itens do cofre nem pede senha
mestra. Evidência LLM: `03-llm-calls.txt` (recusa explícita).

**KB curada (4 entradas audit + 1 NVR prévia):**
`kb-audit-zbx-dep-001`, `kb-audit-zbx-host-001`, `kb-audit-vw-zk-001`,
`kb-audit-vw-signups-001`.

**Validação (sem LLM):** host plantado `npx-audit-planted-notrigger`
(hostid 10686) → finding `host-sem-trigger` → fluxo CONFIRMO → template
ICMP + 3 triggers; VW `signups_allowed` true→false. Screenshot real
Zabbix: `05-zabbix-host-after-confirm.png`.

**LLM:** **TOTAL_LLM_CALLS = 2** (wizard chama `auditar_zabbix` primeiro;
recusa análise de senhas do cofre).

**Evidência:** `docs-publish/validation/sessao-wizard-auditor-2026-08-02/`.

### Pendências explícitas (catálogo / C.5.2)

- **BookStack / Nextcloud / Chatwoot:** sem `auditar_*` nesta entrega.
- **Uptime Kuma:** só check raso (Socket.IO, sem REST profundo /
  monitor-sem-notificação).
- **Busca externa ao vivo** (`pesquisar_documentacao_tecnica` +
  `searchApiKeyEncrypted`): **não implementada** — KB curada cobriu as
  checagens; chave opcional fica para o responsável adquirir se quiser.

## 2026-08-02 (cont.) — Onda grande de desenvolvimento entregue ao Cursor: bug crítico + central de manuais + fechamento de lacunas + backlog pronto

Pedido do responsável do projeto: rodada grande (pode levar horas),
cobrindo tudo que ainda está pendente/em aberto, com pré-requisitos
resolvidos junto quando possível, e validação pesada via browser real
(a máquina do Cursor tem browser, screenshot obrigatório). Também pediu
manuais completos da plataforma — todos os níveis (ADMN, nível 1/MSP,
nível 2/cliente final), 3 idiomas, com imagem real, adaptados ao
branding de quem está vendo — citando os manuais de migração
(`docs/templates/MANUAL-MIGRACAO-*.md`) e o script de seed do BookStack
(`portal/scripts/seed-bookstack-manuals.cjs`) como referência de formato
já existente no projeto.

Antes de escrever o prompt, revisei `docs/ROADMAP-MACRO.md` e
`docs/ROADMAP.md` por completo pra separar o que é realmente
implementável agora do que depende de terceiro/decisão de negócio —
não deu pra simplesmente "implementar tudo que está no roadmap":
WhatsApp (Meta Business), multi-host (VM dedicada não existe), destino
de backup em nuvem específico (OAuth de app registrado), domínio-base
real (domínio ainda não comprado) — tudo isso continua genuinamente
bloqueado em algo que não é código. Documentado explicitamente como
"fora de escopo" no prompt novo, pra Cursor não tentar decidir isso
sozinho.

**Achado real durante a revisão**: bug de prioridade alta registrado em
2026-07-30 (usuário preso na visão "Cliente" após login, seletor de
tenant não respondia, só normalizou com hard refresh) **nunca foi
investigado** — virou Fase 0 (prioridade máxima) do prompt novo, antes
de qualquer outra coisa.

**Decisão de arquitetura pra central de manuais** (registrada no prompt,
importante pra não ser refeita errado): manuais do produto vivem DENTRO
do próprio portal (seção nova, `ManualPage` sem campo de tenant),
herdando o branding de quem está vendo automaticamente por já estar
dentro do mesmo shell que resolve branding hoje — nunca tentar "pintar"
o BookStack com marca de cada cliente (BookStack não foi feito pra
isso, seria gambiarra frágil). BookStack continua só pra vitrine
pública da NPX e como app que cada cliente pode provisionar pra uso
próprio (wiki interna dele) — não confundir os dois usos.

Prompt entregue: **`docs/PROMPT-CURSOR-onda-grande-2026-08-02.md`** —
Fase 0 (bug crítico), Fase 1 (central de manuais), Fase 2 (fecha
pendências explícitas do wizard auditor: BookStack/Nextcloud/Chatwoot
sem auditoria, Uptime Kuma raso), Fase 3 (backlog pronto: domínio
próprio self-service, domínio-base configurável ADMN, certificado
próprio por instância, proxies Zabbix por tenant, backup Kopia destino
de rede sem OAuth, métricas/logs por instância, dashboard por widget,
SSO GLPI via proxy de autenticação). Documentar e publicar ao final de
CADA fase (não só no fim de tudo), pra eu poder validar em paralelo.

**Pendência própria meu (Claude Code), não delegada**: relatório de
segurança executivo (seção 21 do MACRO) é documentação pura, fica
comigo, não faz parte deste prompt.

**Pendência sinalizada ao responsável do projeto, não decidida
sozinha**: item B de `docs/SERVICE-ACCOUNTS.md` (container `portal`
rodando com UID do `suporteti`) continua fora de qualquer prompt até
autorização explícita — risco de produção real.

## 2026-08-03 — Quadro de status das 29 etapas + rodada CRM/fechamento pesado entregue

Pedido do responsável do projeto: uma tabela real (etapa/função/
descrição/status) de todo o projeto, pra ver "onde estamos" — montada
cruzando as 24 seções do `docs/ROADMAP-MACRO.md` com tudo validado de
verdade nas últimas sessões, publicada como artefato interativo
(filtro por status/busca). Resultado: 7 concluídas, 10 parciais, 10
não iniciadas, 1 fora de escopo por decisão consciente — o padrão que
salta é que a engenharia está bem à frente do motor comercial (preço,
cobrança, jurídico, marca ainda não começaram, não por bloqueio
técnico, são decisão de negócio).

Pedido seguinte: identificar tudo que pode virar Concluído só com
trabalho nosso (nada de terceiro) e atacar numa rodada pesada,
incluindo um **CRM comercial interno** novo (campanhas, clientes,
contratos, preços, produtos). Antes de escrever o prompt, conferi o
banco real pra não reinventar: já existe um módulo comercial parcial
(`quotes`/`quote_lines`/`sale_items`/`time_entries`/
`commercial_tickets`/`commercial_audit`, da Sessão 41) — o CRM novo
**estende** isso (contrato recorrente/MRR, campanha nomeada
reaproveitável, visão consolidada de cliente, painel de meta), não
recria do zero.

Prompt entregue: **`docs/PROMPT-CURSOR-crm-e-fechamento-2026-08-03.md`**
— Fase 0 (CRM), Fase 1 (agentes de IA de suporte comercial/técnico +
handoff, fecha §11), Fase 2 (inadimplência/cobrança via e-mail, fecha
§12), Fase 3 (backup agendado automático, fecha o resto de §14), Fase
4 (manutenção automática de disco — spec já existia pronta desde
2026-07-30 em `PROMPT-CURSOR-manutencao-disco.md`, nunca executada,
só apontei pra ela), Fase 5 (trial/demo sem precisar de VM nova,
escopo reduzido pra não depender de hardware), Fase 6 (CrowdSec +
Pi-hole/AdGuard, marcado risco maior por mexer em Traefik/FortiGate
compartilhados, mesma disciplina de isolamento já provada com sucesso
no certificado próprio desta semana).

Filtrado antes de escrever (explicitamente fora, tudo com motivo real
de terceiro/hardware): checkout com gateway de pagamento, compra do
domínio da marca, backup em nuvem OAuth, WhatsApp, Wazuh/câmeras/
segregação de infra (VM nova), token restrito do GitHub (ação manual
só do responsável do projeto), UID do container `portal` (segue
esperando autorização explícita, não é terceiro mas é risco de
produção já sinalizado há dias), nome da marca/jurídico/preço final
(decisão humana, não tarefa de código).

**Itens que ficam comigo (Claude Code), não vão pro Cursor**: relatório
de segurança executivo (§21), estudo de capacidade de hardware (§23 —
posso medir de verdade agora, comandos já existem no host), checklist
de onboarding formalizado no RUNBOOK (§19) — vou atacar em paralelo à
rodada do Cursor.

## 2026-08-03 (cont.) — Fase 0 (CRM) e Fase 4 (disco) do Cursor validadas; estudo de capacidade concluído

Cursor entregou Fase 0 (CRM) e Fase 4 (manutenção de disco) do
`PROMPT-CURSOR-crm-e-fechamento-2026-08-03.md`. **Validado de verdade
nesta sessão**, não só aceito pelo relato:
- CRM: `contracts`/`campaigns` reais no banco (MRR R$90, campanha 10%
  código CRM10, batendo com o print), rotas `/crm`,
  `/crm/campaigns`, `/crm/tenants/[id]/contracts/new` existem. Único
  ponto solto: tenant descartável `teste-crm-1785765133` (contrato +
  campanha + item de teste) tinha ficado no banco em vez de limpo —
  removido por mim, zero impacto (nenhuma instância associada).
- Disco: `scripts/docker-maintenance.py` cruza contra a tabela
  `tenants` real antes de tocar em qualquer recurso ("Fonte de
  verdade: se o slug inferido existe em tenants, NUNCA tocar", linha
  285) — exatamente a regra absoluta exigida. Cron ativo de verdade
  (órfãos/20min, limpeza completa 03:15). Cache de build conferido ao
  vivo: 16,65GB, batendo com o reportado.

**Estudo de capacidade de hardware (§23) concluído** — medido de
verdade contra este host (sar 6 dias, docker stats ao vivo, docker
system df por volume), não estimado. Resultado completo em
`docs/CAPACITY-STUDY-2026-08-03.md`: CPU tem folga enorme (pico de
5,5% na semana), mas RAM e disco não aguentam nem perto de 500
clientes neste host único — teto real estimado em ~15-20 clientes de
perfil médio (referência real FLUA: 3 instâncias, ~1,5GB RAM/~4,2GB
disco) antes de saturar. §23 do `ROADMAP-MACRO.md` atualizado
apontando pro estudo.

## 2026-08-03 (cont.) — Fases 1/2/3/5/6 do fechamento validadas; correção de escopo do WhatsApp

Cursor concluiu as Fases 1 (Chatwoot IA), 2 (inadimplência), 3 (backup
agendado), 5 (trial) e 6 (CrowdSec/Pi-hole, avaliados e conscientemente
não ativados) de `PROMPT-CURSOR-crm-e-fechamento-2026-08-03.md`.
**Validado de verdade** — dunning real (4 eventos no banco, e-mail pro
`nicholasalex@gmail.com` conforme regra do projeto, container suspenso
de verdade), backup agendado com snapshot Kopia real criado sozinho
pelo cron, bloqueio de 2º trial confirmado por violação real de
constraint única no Postgres (não só UI), e confirmado ao vivo no
container `traefik` que CrowdSec genuinamente não foi ativado (zero
referência em config real). Mesmo padrão de resíduo da Fase 0 (CRM):
tenants descartáveis de teste (`teste-dunning-*`, `teste-trial-*`)
ficaram no banco em vez de limpos — removidos por mim, zero impacto.

**Correção de escopo, pedida pelo responsável do projeto**: eu tinha
classificado WhatsApp como inteiramente "fora de escopo" (bloqueado em
terceiro) nos prompts desta semana — isso era cautela em excesso. Só a
verificação de identidade do negócio e a aprovação de template pela
Meta são genuinamente não automatizáveis; toda a credencial, GUI,
webhook, relay e ligação com Zabbix/Grafana/GLPI/Chatwoot — inclusive
a **submissão** de template pra aprovação (só a decisão de aprovar é
da Meta) — é scriptável via Graph API e deveria estar pronta. Prompt
novo: `docs/PROMPT-CURSOR-whatsapp-integracao-2026-08-03.md` — GUI de
credencial por tenant, gestão de templates via API, media type Zabbix
e contact point Grafana via relay próprio (nunca token real exposto
na config do Zabbix/Grafana), notificação GLPI via nosso próprio
código (GLPI não tem webhook nativo), canal WhatsApp no Chatwoot
criado via API (Chatwoot já suporta nativamente). Validação usa o
número sandbox de desenvolvimento da própria Meta, sem esperar
verificação de negócio real.

## 2026-08-03 (cont.) — WhatsApp: mecanismo validado, mas achado real de permissão bloqueava self-service

Cursor entregou a integração WhatsApp (código real conferido: token
cifrado em `lib/whatsapp.ts`, relay `/api/whatsapp/relay` nunca expõe
o token real pro Zabbix/Grafana — só um segredo por tenant — webhook
com verify_token real, tabelas `tenant_whatsapp_config`/
`whatsapp_message_templates`/`whatsapp_relay_audit` reais). Pediu pro
responsável do projeto preencher um `.env` no host com credencial
sandbox da Meta pra fechar o E2E.

Responsável do projeto apontou, corretamente, que isso não pode
depender de alguém preencher `.env` no servidor nem passar credencial
pela IA — precisa ser self-service de verdade, pronto pra qualquer
tenant configurar sozinho pela GUI, com instruções na própria tela.
**Investigando o código antes de aceitar**, achei exatamente o
problema real: `canManageTenants` (`lib/authz.ts`) é `isAdmn(session)`
puro, e tanto `integrations/whatsapp/page.tsx` quanto `actions.ts`
usam só essa checagem — **hoje só ADMN consegue configurar WhatsApp de
qualquer tenant, nenhum admin de tenant nível 1/2 consegue configurar
o próprio**. Achado confirmado lendo o código, não suposição.

Prompt de correção entregue:
`docs/PROMPT-CURSOR-whatsapp-self-service-2026-08-03.md` — trocar a
permissão pro mesmo padrão já usado em branding (tenant configura o
próprio, ADMN continua podendo configurar qualquer um), e adicionar
instruções passo a passo **na própria tela** (não só em doc separado)
de como conseguir os 4 valores no developers.facebook.com, mais uma
página nova na central de manuais. O banner de resultado ok/erro
depois de salvar/testar já existe e está correto — não mexer nisso.

## 2026-08-03 (cont.) — Pergunta direta do responsável: IA já configura Zabbix/GLPI de verdade? Limite de gasto já funciona?

Investigado com código real, não de memória.

**IA configurar aplicações real (Zabbix/GLPI/Grafana)**: **sim,
confirmado** — reconferi evidência já validada nesta semana:
`sessao-migracao-ia-2026-08-02/03-ai-tools-e2e.txt` (host Zabbix
criado de verdade, dashboard Grafana criado, categoria GLPI criada) e
`sessao-wizard-auditor-2026-08-02/04-confirm-flow.json` (fluxo real
achado→confirmação→aplicação: triggers foram de 0 pra 3 depois da
frase exata de confirmação, host real `10686`). Funciona tanto no
nível de função quanto disparado por conversa real em linguagem
natural (wizard chamando `auditar_zabbix` sozinho antes de perguntar).

**Limite de gasto de IA por tenant/subtenant, em dinheiro**: **motor
real e funcional, mas não ligado pra quase ninguém — achado sério**.
`ensureTenantOpenRouterKey` cria chave própria por tenant na
OpenRouter com limite real em dólar, **imposto pelo próprio
OpenRouter** (não é só checagem nossa) — comprovado funcionando pro
tenant `npx` (`ai_openrouter_limit_usd=4.993`, chave real). Mas:
- **Nenhum tenant real tem isso ligado** — só `npx` (uso interno).
  **FLUA, o único cliente pagante, está rodando o assistente sem
  nenhum teto de gasto agora mesmo.**
- Provisionamento é 100% manual (clique), nunca acontece sozinho na
  criação de um tenant.
- **Achado de segurança real**: `provisionTenantAiKeyForm`
  (`tenants/[id]/ai-credits/actions.ts`) — a Server Action que define
  o limite de gasto de qualquer tenant — **não tem nenhuma checagem de
  permissão**, ao contrário das outras funções do mesmo arquivo
  (`saveResellerMarginForm`/`saveNpxMarginForm`, que chamam
  `requireTenantAccess` corretamente). Hoje, uma chamada direta
  consegue alterar o limite de qualquer tenant.
- `AI_BILLING_BYPASS_TEMP=true` (`lib/ai/billing-bypass.ts`) —
  documentado como temporário até existir gateway de pagamento.
  **Não é o problema real**: só desliga a checagem prévia da nossa
  própria aplicação, não desliga o teto duro que o OpenRouter impõe
  numa chave que já tenha limite — o problema real é que quase nenhuma
  chave tem limite configurado.

Prompt entregue, prioridade alta:
`docs/PROMPT-CURSOR-ia-limite-credito-2026-08-03.md` — corrigir o
buraco de segurança primeiro, provisionar chave com limite automático
pra tenants nível 1 (incluindo corrigir FLUA retroativamente,
imediatamente), sub-alocação de limite pra subtenants nível 2 seguindo
a mesma lógica já usada nas cotas (§5 do MACRO), atualizar o banner da
UI pra não sugerir que nada está protegido quando o teto duro já
funciona, e validação real (estourar um limite pequeno de propósito e
confirmar que a chamada seguinte falha de verdade).

## 2026-08-03 (cont.) — INCIDENTE REAL: lixo de provisionamento falho, achado ao vivo no tenant `felixti`

Responsável do projeto viu um print real de falha de provisionamento
de GLPI (`felixti`, 2026-07-31) e perguntou se sobra lixo no sistema
quando isso acontece. Fui investigar ao vivo em vez de responder de
memória — **achei um incidente acontecendo em tempo real durante a
investigação**: 3 tentativas de provisionamento de GLPI simultâneas
na fila pro tenant `felixti` (slugs `glpi`/`glpi-2`/`glpi-3`, criadas
em 0,4s de diferença, 2026-08-03 18:24:31-32 — claramente disparo
duplo/triplo, sem debounce nem bloqueio server-side).

**Confirmado lendo o código, causa raiz real**: `rollback()`
(`lib/provisioning.ts`) limpa container/volume/compose/stack mas
**nunca toca a linha da tabela `instances`** — quem apaga o
placeholder em caso de falha é um `.then()` anexado à promise de
`provisionInstance`, que só existe **na memória do processo Node em
execução**. Se o container `portal` for reiniciado enquanto um
provisionamento (até 10 min de teto) está rodando — coisa que
acontece com frequência real durante desenvolvimento ativo — esse
`.then()` nunca roda, o placeholder nunca é apagado, a infra (se já
criada) nunca é revertida. **Fica lixo pra sempre — exatamente o medo
do responsável do projeto, confirmado como cenário real, não
hipotético.** Todo apagamento de placeholder usa `.catch(() => {})`
silencioso, sem log — o mesmo anti-padrão que o próprio `rollback()`
já documenta ter custado tempo real numa sessão anterior, intocado
bem ao lado.

Observado ao vivo: a tentativa 1 se autolimpou corretamente (processo
não foi reiniciado durante o teste) — confirma que o mecanismo
funciona QUANDO o processo sobrevive, e falha quando não sobrevive.
Tentativa 2 chegou a criar containers reais
(`felixti-glpi-2`/`felixti-glpi-mysql-2`). Tentativa 3 ainda não tinha
criado infraestrutura — decidi **não apagar a linha do placeholder na
unha**, porque isso não impede o trabalho já agendado em memória de
rodar mais tarde e criar infraestrutura órfã de qualquer jeito (só
trocaria "linha órfã" por "container órfão"); sem um mecanismo real de
cancelamento, nenhuma intervenção manual seria genuinamente segura.

Prompt entregue, urgente:
`docs/PROMPT-CURSOR-provisionamento-resiliente-2026-08-03.md` — botão
Cancelar real com rollback garantido, reconciliação automática
(cron, mesmo padrão do `docker-maintenance.py`) que varre
`provisionando` travado e cruza contra infraestrutura real antes de
limpar, correção de todo `.catch(() => {})` silencioso do caminho,
bloqueio de duplicidade na origem (debounce + checagem server-side em
vez de gerar `tipo-2`/`tipo-3` silenciosamente), e catálogo de falhas
conhecidas com tentativa de correção automática onde fizer sentido.
Validação usa o próprio incidente do `felixti` como primeiro teste
real.

## 2026-08-04 — INCIDENTE REAL resolvido: Zabbix server da FLUA parado, corrigido na hora

Responsável do projeto mandou print real da UI do Zabbix da FLUA:
"O servidor Zabbix não está rodando". Investigado e corrigido
imediatamente, não só registrado pra depois — cliente pagante real
afetado.

**Causa raiz confirmada nos logs**: `flua-zabbix-server` perdeu a
conexão com o MySQL (`[Z3005] query failed: [2006] Server has gone
away` / `[4031] The client was disconnected by the server because of
inactivity`, a partir de 2026-08-04 12:06) e não reconectou sozinho —
comportamento conhecido do Zabbix nesse cenário. O container
continuava "Up" (por isso nada no Docker/Portainer acusava problema)
— só processava zero dado de verdade.

**Corrigido**: `docker restart flua-zabbix-server` (ação segura,
reversível, sem perda de dado). Confirmado com evidência real, não só
"container subiu": API respondendo (`apiinfo.version`) e **705
valores processados** nos 2 minutos seguintes ao restart — monitoramento
genuinamente voltou a funcionar, não só o processo.

**Checado proativamente**: as outras 5 instâncias Zabbix da
plataforma (npx, demo, valid1, felixti, validnivel2) — nenhuma tinha o
mesmo sintoma nos logs das últimas 6h. Isolado na FLUA.

**Prevenção entregue**: `docs/PROMPT-CURSOR-zabbix-watchdog-2026-08-04.md`
— detecção real de atividade (não só "container Up"), correção
automática (mesmo restart que resolveu agora), alerta real no NOC
interno da NPX toda vez que acontecer (mesmo quando a correção
automática resolver sozinha — visibilidade, não só correção
silenciosa), e investigação de configuração preventiva
(`wait_timeout`/reconexão) antes de aplicar qualquer mudança.

## 2026-08-04 (cont.) — Watchdog Zabbix validado; redesign de UI entregue

Cursor entregou `scripts/zabbix-server-watchdog.py` (detecção por
atividade real — `MAX(clock)` em history + log "gone away", não
`docker ps` — 1 restart automático com cooldown 30min, alerta High no
NOC se não recuperar, cron `*/5`). **Validado ao vivo**: rodei o
script agora mesmo, sem `--apply`, contra os 6 Zabbix reais da
plataforma — todos saudáveis, `flua-zabbix-server age=1s`. Sem regressão
do incidente anterior.

Responsável do projeto mandou print real da tela "Editar tenant — FLUA
TI" apontando problema de UI sério: conteúdo espremido num canto da
janela, muito espaço vazio, navegação "ver instâncias · criar
instância · branding · integrações..." parecendo índice de manual em
vez de abas. Pedido: isso se repete "em todo lugar", precisa de cara
de software moderno, responsivo pra celular/tablet.

Investigado antes de escrever o prompt (não assumir, confirmar no
código): **39 arquivos** em `portal/src/app` usam `max-w-lg`/
`max-w-xl`/`max-w-md`/`max-w-sm` no card principal — padrão sistêmico,
não só a tela do print. A navegação "índice de manual" é literalmente
uma sequência de `<Link>` de texto com `hover:underline` — **não
existe nenhum componente de Abas no projeto**. Sistema de tokens de
cor/tema (`tailwind.config.js`, ligado ao motor de branding por
tenant) já é bom — problema é layout/densidade, não paleta.

Prompt entregue: `docs/PROMPT-CURSOR-redesign-ui-2026-08-04.md` —
componente de Abas real, estratégia de largura de página (parar de
espremer tudo numa coluna fixa pequena, usar grid responsivo onde fizer
sentido), levantamento sistemático dos 39 arquivos, modernização de
hierarquia visual/espaçamento, responsivo testado de verdade em 3
larguras (mobile/tablet/desktop+tela larga) — preservando o sistema de
branding por tenant existente, sem recriar paleta.

## 2026-08-04 (cont.) — Redesign v1 não convenceu; testei ao vivo no navegador e escrevi v2 com código pronto

Cursor executou a v1 do redesign. Responsável do projeto abriu a tela
real e não gostou: continua parecendo formulário, abas "não aparentam
ser abas", clicar entre abas parece carregar janela nova. Mandou
imagens de referência (kits de aba estilo fita/pasta colorida,
~2012) — não pedindo réplica exata, só "mais bonito".

**Desta vez fui eu mesmo ao navegador real** (login ADMN,
`/tenants/[id]` e `/tenants/[id]/instances` da FLUA) em vez de só ler
código — achei a causa exata de cada queixa, com print:
- Barra de abas muda de posição vertical entre páginas (y≈124px numa,
  y≈98px noutra) porque cada `page.tsx` renderiza o próprio
  cabeçalho + `TenantTabNav` de forma independente — não existe
  `layout.tsx` compartilhado prendendo isso num lugar fixo. Confirma
  literalmente a queixa "parece janela nova a cada clique".
- `components/ui/TabNav.tsx` usa só borda inferior 2px + fundo
  vermelho a 5% de opacidade — em tema escuro isso é quase
  imperceptível, por isso não parece aba de verdade.
- v1 trocou coluna estreita por grid de 2 colunas desbalanceado
  (alturas diferentes, vão vazio grande) — concordo com a leitura do
  responsável do projeto de que "só esticou, não ficou natural".

**Prompt v2 entregue com código pronto, não só descrição** (lição da
v1: direção vaga não bastou) —
`docs/PROMPT-CURSOR-redesign-ui-v2-2026-08-04.md`. **Executado em
2026-08-04 (Cursor)** — ver seção "Redesign UI v2 — CONCLUÍDA" no
topo deste arquivo. Validação ao vivo: Y da barra idêntica (delta 0)
entre Geral e Instâncias FLUA.

## 2026-08-04 (cont.) — IA do tenant: 3 achados de UX + 1 bug funcional real, com causa raiz confirmada

Responsável do projeto testou o assistente de IA de um tenant de
verdade (auditoria Zabbix real + criação de dashboard Grafana) e
trouxe 4 pontos. Investigado cada um no código antes de escrever
prompt, não assumido:

1. **Colar imagem/texto grande no chat** — hoje só existe upload por
   seletor de arquivo (`AiAssistantDrawer.tsx`, `onPickFile`); zero
   handler de `onPaste`. Pipeline de upload já funciona, só falta o
   gatilho por Ctrl+V.
2. **Detalhe técnico de ferramenta vazando pro usuário final** —
   confirmado: `<details>` já é colapsável, mas mesmo assim expõe nome
   técnico da tool (`auditar_zabbix`) e, ao abrir, JSON cru inteiro
   direto na conversa — sem distinção entre usuário final e ADMN.
3. **Confirmação de mutação (`CONFIRMO: autorizo...`) não é elemento
   de UI nenhum** — achado real: é só a resposta em texto livre do
   próprio modelo, sem nenhum destaque visual, sem botão — por isso
   "quase passou batido". Backend já devolve `needsConfirmation`/
   `confirmationId`/frase exata estruturados, só não chegam separados
   pro frontend.
4. **Bug funcional real confirmado com evidência**: dashboard criado
   pela IA pra FLUA/MIP (`be205f52-d292-4e95-9bd3-6638c3eea5fa`) veio
   sem nenhum dado. Consultei a API do Grafana direto — causa exata:
   `criar_dashboard_grafana` (`lib/ai/app-tools.ts`) carrega o
   template (`zabbix_system_status.json`) e nunca resolve o
   placeholder `${DS_ZABBIX}` — o JSON salvo literalmente tem
   `"uid":"${DS_ZABBIX}"`, uma string que não existe. O próprio
   arquivo já sabe fazer certo (branch `minimal` busca o datasource
   real via API) — só o branch de template importado nunca reaproveita
   essa busca.

Prompt entregue:
`docs/PROMPT-CURSOR-ia-chat-ux-e-bug-dashboard-2026-08-04.md` — paste
de imagem/texto reaproveitando pipeline existente, ocultar detalhe
técnico por padrão pro usuário final (visível só ADMN/modo técnico),
card de confirmação real com botão (mantendo a mesma garantia de
frase exata, só sem digitação manual), e correção do bug do
`${DS_ZABBIX}` com resolução real do datasource + verificação pós-save
antes de reportar sucesso. Dashboard quebrado da FLUA/MIP entra como
parte da validação (recriar, provar dado real nos painéis).

## 2026-08-04 (cont.) — IA do tenant: imagem não é vista de verdade, dangerous tier não validado, falta gerar documentação

Cursor entregou a rodada anterior (paste, ocultar JSON técnico, card
de confirmação por botão) e corrigiu o bug do dashboard — conferido
via print real (`06-confirmation-card.png`, `05-ai-attachments.png`).
Responsável do projeto testou de novo e trouxe mais 3 pontos reais,
investigados no código antes de escrever prompt:

1. **Imagem colada não é vista pelo modelo** — a IA respondeu "não
   tenho capacidade de OCR", o que é tecnicamente certo mas é a
   solução errada: o modelo configurado (`anthropic/claude-sonnet-5`)
   enxerga imagem nativamente. Causa raiz confirmada em
   `app/tenants/[id]/ai/actions.ts` (~linha 226-236): todo anexo vira
   só texto via `textExtract` (`lib/ai/extract-text.ts`, que só sabe
   PDF/DOCX/XLSX — nunca imagem) — a imagem chega ao modelo como
   "(sem extrato de texto)", nunca como imagem de verdade. UI também
   mostra pílula de texto (`colado-test.png`), não miniatura.
2. **Confirmação por botão nunca foi testada no tier `dangerous`** (2
   confirmações, frase reforçada, `command-risk.ts`) — só a mutação
   simples (1 confirmação) tem evidência real da rodada anterior. O
   componente (`ConfirmationCard`) parece genérico o bastante pra já
   funcionar, mas isso é leitura de código, não prova.
3. **Falta capacidade de gerar/atualizar documentação** — hoje só
   existe `ler_documentacao_tenant` (leitura de metadados fixos). Pedido
   explícito: a IA deve conseguir, depois de configurar/auditar algo,
   também deixar registrado em texto pro tenant consultar depois —
   mesmo espírito do que já está em `ROADMAP-MACRO.md` §10.

Prompt entregue:
`docs/PROMPT-CURSOR-ia-visao-doc-confirmacao-2026-08-04.md` — visão
real de imagem (bloco multimodal `image_url` na chamada ao modelo, não
extração de texto) + miniatura real na UI; validação real e reforço
visual (vermelho, 2 cliques) do tier `dangerous`, nunca reduzir pra
menos confirmações; tool nova `gerar_documentacao_instancia` (mesmo
padrão de owns-check/confirmação de sempre), campo novo em `Instance`,
exibido em `/tenants/[id]/docs`. Fronteira de escopo reafirmada
explicitamente como princípio permanente: IA do tenant nunca cruza pra
SO/Docker direto/Portainer/infra da NPX, sempre via API de cada
aplicação — vale pra esta capacidade nova e qualquer futura.

## 2026-08-04 (cont.) — Achado real: modelo pede CONFIRMO em texto solto sem chamar ferramenta (não é bug do botão)

Responsável do projeto, irritado, reportou que a IA voltou a pedir
"responda com CONFIRMO" em texto puro, sem botão, e que "continua não
conseguindo ler imagem". **Investigado direto no banco antes de
qualquer prompt novo** (não confiado no relato de conclusão anterior
nem no relato do usuário sem checar):

- Mensagem real (`ai_chat_messages`, FLUA, 22:44:41) com o texto
  "Responda com: CONFIRMO..." tem `meta` **nulo** — sem
  `pendingConfirmations`. `ai_action_log` no mesmo intervalo: **zero
  linhas**. Conclusão real: **nenhuma ferramenta foi chamada nesse
  turno** — o modelo escreveu o pedido de confirmação como texto
  livre, por conta própria, sem invocar `criar_dashboard_grafana`.
  O componente de botão (`ConfirmationCard`) está correto — só não
  tinha `confirmationId` real pra renderizar. É bug de comportamento
  do modelo/system prompt, não da UI construída na rodada anterior.
- **Visão de imagem: baixei o arquivo real anexado
  (`colado-1785883277362.png`) e abri eu mesmo** — a IA descreveu o
  conteúdo corretamente (era um print de uma resposta anterior dela
  própria, não um mockup novo). Visão está funcionando de verdade;
  o "não está lendo" foi a IA relatando corretamente que a imagem
  colada não era o que o responsável do projeto pensava ter colado.

Prompt entregue, urgente:
`docs/PROMPT-CURSOR-ia-confirmacao-real-e-anexos-2026-08-04.md` —
regra explícita no `chat.ts` proibindo o modelo de escrever frase de
confirmação sem ter chamado a ferramenta no mesmo turno, checagem
defensiva no backend (log de alarme se acontecer de novo), validado
repetindo o teste 3x (comportamento de modelo não é determinístico).
Mais tabela de-para de anexos (Claude vs. portal): remover nome de
arquivo técnico concatenado na mensagem enviada, botão de remover
anexo pendente, ampliar tipos aceitos (yaml/xml/html/py/sh/sql/rtf/
ODF, heic/webp/svg pra imagem — cobrindo Mac/Linux, não só Office/PDF
do Windows), rótulo amigável em vez de nome técnico gerado.

## 2026-08-04 (cont.) — CONFIRMO em texto sem tool + anexos: CORRIGIDO e validado 3×

**Causa confirmada (não era o botão):** o modelo às vezes escrevia
"Responda com: CONFIRMO..." em prosa sem chamar a tool — `meta` nulo,
zero linhas em `ai_action_log`. `ConfirmationCard` estava certo; não
havia `pendingConfirmations` pra renderizar.

**Correção (portal rebuild + redes `_internal` religadas):**
1. System prompt + wizard: proibição categórica de pedir CONFIRMO em
   texto; mutação = chamar a tool; depois do `needsConfirmation`, só
   apontar o botão na tela.
2. Defesa em `runChatTurn`: se prosa casa com pedido de CONFIRMO e
   `pendingConfirmations` vazio → `console.warn` + **retry automático**
   (1×) forçando tool call; se ainda falhar, resposta sanitizada.
   Se há card mas a prosa ainda narra CONFIRMO → strip dos parágrafos.
3. Anexos (`AiAssistantDrawer` + `extract-text` + upload): não concatena
   `(Anexos: colado-…)` no texto; chip com “×” pra remover; labels
   “Imagem” / “Texto colado (N+ caracteres)” / nome original do arquivo;
   extração yaml/xml/html/css/js/ts/py/sh/sql/ini/conf/rtf/odt|ods|odp;
   imagens heic/heif/webp/bmp/svg/tiff pelo caminho de visão.

**Validação real (Playwright, FLUA, 3 conversas separadas):**
`docs-publish/validation/ia-confirmacao-real-anexos-2026-08-04/` —
`report.json` com `ok: true`; `confirm-run-{1,2,3}.png` com
`ai-confirmation-card` + botão Confirmar em todas; zero free-text
CONFIRMO sem card; anexos yaml/py/texto colado com labels amigáveis,
remoção do chip OK, sem nome técnico na mensagem. Script:
`portal/scripts/validate-confirm-card-3x.cjs`.

## 2026-08-04 (cont.) — Causa estrutural real do dashboard quebrado (3ª tentativa): falta ferramenta, não é falta de instrução

Responsável do projeto, com razão, ficou sem paciência: terceira vez
que pede o mesmo dashboard de status de ativos (FLUA/MIP - BH) e sai
errado, apesar da IA escrever um plano detalhado e correto na
conversa (5 categorias, painel por host, ícone+nome+IP+status). Fui
conferir ao vivo na API do Grafana (dashboard
`97802c35-5ff3-4c37-bdda-56642488bec6`) antes de escrever qualquer
coisa: **o dashboard real criado tem exatamente 1 painel** — o
genérico "Dashboard mínimo. Datasource Zabbix: ..." (branch
`minimal`), nada a ver com o plano descrito na conversa.

**Causa raiz real**: `criar_dashboard_grafana`
(`lib/ai/app-tools.ts`) só aceita `templateName` — `"minimal"` (painel
de texto trivial) ou um arquivo estático de `templates/grafana/`.
**Nunca existiu um jeito do modelo pedir "monta um painel por host,
agrupado por categoria"** — não é o modelo falhando em entender ou em
executar, é a ferramenta que ele tem disponível não ter essa
capacidade fisicamente. O modelo narrou certo e depois só teve a
opção de cair no branch trivial.

Prompt entregue, prioridade máxima:
`docs/PROMPT-CURSOR-dashboard-status-ativos-2026-08-04.md` —
parâmetro estruturado novo (`categorias: [{nome, icone, hosts}]`) onde
o modelo só decide QUAIS hosts vão em qual categoria (já demonstrou
saber fazer isso certo) e o **código** (determinístico) resolve cada
host de verdade no Zabbix (`host.get`+`available` — campo certo pro
"offline mesmo sem alerta" pedido explicitamente), descobre o item de
ICMP real por host, calcula layout proporcional (24 unidades de grid,
largura por categoria proporcional à quantidade de hosts), gera painel
tipo `stat` por host com cor por threshold de disponibilidade real.
Verificação pós-save: contagem de painéis tem que bater com
quantidade de hosts, nunca reportar sucesso se cair no branch errado
de novo. Aplicar direto no dashboard real da FLUA (não outro teste
solto) usando os hosts já confirmados na conversa (Servidores=Zabbix
server+FLUA-Demo-Host, Firewalls=1, Switches=7, Impressoras=7,
Câmeras=17, total 34).

## 2026-08-04 (cont.) — Correção: eu errei ao instruir o Cursor a criar dashboard direto na FLUA

Responsável do projeto corrigiu, com razão: eu tinha instruído o
Cursor a "aplicar direto no dashboard real da FLUA" como parte da
correção — errado. O objetivo não é o dashboard existir, é **a IA do
tenant conseguir criar isso sozinha, pelo produto real, respeitando a
hierarquia de acesso do cliente** (FLUA nível 1 agindo sobre MIP
subtenant via `targetTenantId`). Cursor chamando a API do Grafana
direto por fora do produto não prova nada sobre o produto, só prova
que o Grafana aceita a chamada — seria eu/ele fazendo o trabalho do
cliente manualmente, o oposto do que a plataforma promete.

`docs/PROMPT-CURSOR-dashboard-status-ativos-2026-08-04.md` corrigido:
validação principal agora é contra tenant descartável (mesmo padrão de
sempre); validação contra FLUA/MIP, se necessária, só através do chat
real da IA com hierarquia real, nunca script/API por fora. Adicionado
também: o padrão "IA decide o quê, código decide o como" generaliza
pra qualquer ferramenta de configuração complexa em qualquer app do
catálogo, não só Grafana — revisar `app-tools.ts`/`audit-tools.ts`
procurando o mesmo risco em outras tools.

## 2026-08-04 (cont.) — Dashboard status ativos: modo `categorias` na tool (CORRIGIDO)

**Causa estrutural:** `criar_dashboard_grafana` só tinha `minimal` ou
arquivo estático — o modelo descrevia 5 categorias/painel-por-host e
caía no branch trivial (1 painel).

**Implementado (produto):**
- Parâmetro `categorias[{nome,icone,hosts[{nomeExato}]}]` + `unidade`
  opcional (ex. BH). Com `categorias`, `templateName` é ignorado.
- Código em `lib/ai/grafana-status-dashboard.ts`: resolve host no
  Zabbix (`host.get`+interfaces+grupos), item `icmpping` real, layout
  proporcional (grid 24), painel `stat` por host (ONLINE verde /
  OFFLINE vermelho / DESCONHECIDO amarelo). Sem item ICMP → snapshot
  de `available` documentado na resposta.
- Pós-save: contagem de painéis de host == total de hosts; sem
  placeholder `${DS_*}`.
- `resolveZabbix`/`resolveGrafana` e integração
  `zabbix_to_grafana_datasource` passam a respeitar `containerPrefix`
  (bug real: DS do Grafana NPX apontava pra `zabbix-web` em vez de
  `demo-zabbix-web` — query vazia).

**Validação (zero FLUA/MIP):**
- Tentativa de tenant descartável completo: falhou 2× no provision
  Zabbix (health timeout; depois suporteti). Cleanup feito.
- Validação da tool via `executeAppTool` no NPX dogfooding (hosts +
  dashboard efêmeros, removidos no fim): 12/12 painéis; shot com
  prova offline — `icmpping=0` em host 172.31.255.254 → painel
  OFFLINE vermelho sem trigger; 6 ONLINE. Screenshot
  `docs-publish/validation/dashboard-status-ativos-2026-08-04/dashboard-1920x1080.png`
  + `report.json`. Script:
  `portal/scripts/validate-status-dashboard-tool.ts`.

**Generalização:** demais tools de mutação em `app-tools`/`audit-tools`
já recebem parâmetros estruturados (host/template/política) — o risco
“modelo monta JSON complexo” era concentrado no Grafana. Padrão
registrado em `DECISIONS.md` pra ferramentas futuras.

## 2026-08-04 (cont.) — Dashboards sem limite: template do grafana.com, link direto, ou do zero

Responsável do projeto pediu expansão real: a IA não pode ficar presa
a "só arquivo pré-colocado manualmente na pasta do servidor" — precisa
usar a biblioteca pública de templates do Grafana (milhares de
dashboards prontos), tanto por busca quanto por link direto. Mandou
vários links reais de exemplo (CoreDNS, câmeras/DVR, Kubernetes,
repositório GitHub de câmera/go2rtc).

Confirmei a API real do grafana.com antes de escrever o prompt (não
assumido): `GET /api/dashboards/<id>/revisions/<revisao>/download`
funciona de verdade — testei com CoreDNS (id 14981), baixou 27
painéis reais com `__inputs`/`DS_PROMETHEUS` (mesmo tipo de
placeholder já corrigido pro Zabbix — a resolução de datasource já
obrigatória desde o prompt anterior cobre isso também). Busca por
termo (`q`/`search`/`query`/`term`/`keyword`) não bateu certo numa
tentativa rápida — registrado como algo que o Cursor precisa
investigar a fundo antes de implementar, não adivinhar um parâmetro.

Prompt entregue:
`docs/PROMPT-CURSOR-dashboard-templates-sem-limite-2026-08-04.md` — 3
caminhos: (1) template por ID/link do grafana.com, (2) link direto de
arquivo bruto (ex: GitHub), (3) busca da própria IA com apresentação
de candidatos pro usuário escolher antes de importar (nunca escolha
autônoma sem confirmação). "Criar do zero" já coberto pelo prompt
anterior (modo `categorias`). Mesma regra de execução do prompt
anterior: validação principal em tenant descartável, nunca script por
fora do produto contra FLUA/MIP.

## 2026-08-04 (cont.) — Atualizado o mesmo prompt (não criado um novo): desempenho + genérico + cache central

Responsável do projeto pediu, antes de mandar rodar o prompt "sem
limite" (ainda aguardando o Cursor terminar a rodada anterior): 3
princípios transversais, não só pra dashboard — (1) desempenho é
critério, este é o assistente de IA do produto, tem que ser rápido;
(2) nada pode ser pensado só pra FLUA, tudo tem que funcionar igual
pra qualquer tenant/subtenant novo sem ajuste manual; (3) um centro só
de templates/conhecimento, nunca base espalhada por tenant/feature —
reforçou que a hierarquia (ADMN vê tudo, nível 1 vê ele+subtenants,
nível 2 só ele) nunca pode quebrar.

**Atualizei o mesmo arquivo** (`docs/PROMPT-CURSOR-dashboard-templates-sem-limite-2026-08-04.md`,
não criado um terceiro solto, conforme pedido explícito): seção nova
"Cache central de templates" (`DashboardTemplateCache`, mesmo espírito
sem-campo-de-tenant já usado em `ai_knowledge_base`) — template
baixado uma vez fica disponível pra qualquer tenant reaproveitar sem
baixar de novo (resolve desempenho e genericidade ao mesmo tempo,
resolução de datasource continua por tenant na hora de aplicar,
hierarquia de quem pode AGIR não muda). Passos 1 e 2 do prompt
atualizados pra checar o cache antes de sair na rede. Validação ganhou
item novo: importar o mesmo template em 2 tenants de teste diferentes
e confirmar que a segunda vez veio do cache, mais rápida, sem bater na
rede de novo. **Ainda não mandado executar** — aguardando o
responsável do projeto trazer o resultado da rodada atual do Cursor
primeiro.

## 2026-08-04 (cont.) — Dashboard status-ativos validado; dois achados reais novos (chat da IA e MSP com instância solta)

Cursor entregou `docs/PROMPT-CURSOR-dashboard-status-ativos-2026-08-04.md`
— conferido print real (`dashboard-1920x1080.png`): 7 hosts, 5
categorias, 1 offline em vermelho de verdade (`icmpping=0`, sem
trigger — exatamente o requisito "offline mesmo sem alerta"). Validado
só via produto (dogfooding NPX), nunca tocou FLUA/MIP direto — regra
de execução respeitada.

Responsável do projeto trouxe pedido grande: assistente de IA do
tenant precisa se parecer com o Claude Code em forma de uso (múltiplos
chats com histórico, página dedicada, auditoria real) — e separado
disso, apontou que as instâncias da FLUA deveriam estar dentro do
subtenant MIP ENGENHARIA, não soltas na FLUA.

**Confirmado no banco antes de escrever qualquer prompt**: FLUA
(accountType `MSP`) tem 3 instâncias (Zabbix/Grafana/GLPI) direto nela
mesma; MIP ENGENHARIA (subtenant dela) tem zero. Causa raiz real:
`canCreateInstance` (`lib/authz.ts`) não checa `accountType` — nada
impede um tenant MSP de criar instância nele mesmo. Achado extra: o
link "Meus clientes" já existe no menu da FLUA mas aponta pra
`/clientes`, que é ADMN-only — está quebrado pra quem devia usar.
Quota (`quota-rollup.ts`) já está correta (soma raiz+subtenants contra
um único total contratado) — não precisa de correção, só reafirmado.

Dois prompts entregues, tópicos diferentes (não emendados num só,
conteúdo não relacionado):
- `docs/PROMPT-CURSOR-ia-chat-historico-auditoria-2026-08-04.md` —
  múltiplos chats por usuário (hoje `AiChatThread` só permite 1,
  `@@unique([tenantId,userId])`), compactação de histórico longo,
  página dedicada + categoria nova no menu, auditoria real (tela +
  ferramenta nova pra a própria IA consultar `ai_action_log`), regra
  clara: IA pode oferecer reverter ação própria reversível, nunca
  restaura backup (só recomenda).
- `docs/PROMPT-CURSOR-msp-instancias-hierarquia-2026-08-04.md` —
  bloquear criação de instância direto em tenant MSP, migrar as 3
  instâncias reais da FLUA pra MIP (migração de infraestrutura de
  verdade, não UPDATE de banco — ensaiar em tenant descartável
  primeiro, backup Kopia fresco antes de tocar a FLUA real), tela de
  gestão do MSP (equivalente ao `/clientes` do ADMN, escopada à
  própria árvore).

Nenhum dos dois mandado executar ainda — aguardando o responsável do
projeto.

## 2026-08-05 — Segregação de infraestrutura: início real, 2 novas VMs, Bitdefender no frontend, hostname da produção trocado

Responsável do projeto criou `vsadmnfront` (172.16.11.155, futuro
frontend/borda) e `vsadmndb` (172.16.11.60, futuro servidor dedicado
de banco). Feito nesta sessão:

- Hostname da VM de produção trocado de `npxadmn` pra **`vsadmnapp`**
  (backend/aplicação, mesmo padrão de nome das novas VMs).
- `vsadmnfront`: verificado ao vivo, senha trocada, `apt upgrade`
  rodado, **Bitdefender GravityZone instalado e confirmado rodando**
  (achado: veio com módulo EDR desligado na política atual — ação do
  responsável do projeto no console cloud da Bitdefender pra ligar,
  não é algo que se resolve por linha de comando). Hardware upgradado
  pelo responsável do projeto durante a sessão: 1→16 vCPU, 3,8→15GB
  RAM, confirmado.
- `vsadmndb`: **inalcançável** (SSH e ping sem resposta de rede,
  não é erro de senha) — provavelmente desligada ou rede/FortiGate
  ainda não liberando, precisa de conferência do responsável do
  projeto.
- Respondidas as 3 perguntas de arquitetura antes de qualquer execução
  (o que vai pro frontend, o que fica na aplicação, como o banco
  dedicado funciona de verdade) + a preocupação sobre complexidade/
  gargalo/gestão levantada pelo responsável do projeto — respostas
  completas em `docs/DECISIONS.md`. Recomendação: **sequenciar**
  frontend primeiro (validado e estável) antes de mexer no banco —
  não fazer as duas separações ao mesmo tempo, reduz o risco real que
  a preocupação levantada aponta.
- Distribuição de hardware recomendada pra meta de 100 instâncias em 2
  meses, com gatilho de replanejar aos 50 (mesmo método do
  `CAPACITY-STUDY-2026-08-03.md`, medir de verdade, não estimar).
- Visão de longo prazo (multi-servidor/multi-datacenter, estilo
  Acronis/AWS) registrada em `docs/ROADMAP.md` ("Provisionamento
  multi-host", motivo ampliado) — **não construída agora**, por pedido
  explícito do responsável do projeto.

**Achado real corrigido nesta sessão, separado do resto**: o prompt de
limite de gasto de IA de 03/08 nunca tinha sido executado — a falha de
segurança (`provisionTenantAiKeyForm` sem checagem de permissão)
continuava aberta 2 dias depois, e a FLUA continuava rodando IA sem
teto. Corrigido diretamente (1 linha, rebuild+deploy confirmado) e a
FLUA já tem limite real de US$10 configurado (confirmado no banco e na
tela). O que falta desse pedido (provisionamento automático pra tenant
novo + sub-alocação pra subtenants) virou Parte B do prompt novo.

Prompt entregue:
`docs/PROMPT-CURSOR-frontend-migracao-e-ia-credito-2026-08-05.md` —
Parte A (preparar o frontend, Traefik puxando config do portal via
rede em vez de socket Docker exposto, SEM trocar DNS/produção ainda —
cutover fica pra decisão explícita depois de validação pesada), Parte
B (fechar de vez o provisionamento automático + sub-alocação de
limite de IA).

## 2026-08-05 (cont.) — INCIDENTE REAL: Grafana da FLUA parou de buscar dado do Zabbix (regressão da migração), corrigido na hora

Responsável do projeto reportou: Grafana da FLUA acessível de fora
normalmente, mas painéis em "No data", com erro visível `Post
"http://flua-zabbix-web:8080/api_jsonrpc.php": dial tcp: lookup
flua-zabbix-web ... server misbehaving`.

**Causa raiz confirmada em segundos**: a migração FLUA→MIP
(2026-08-05, ver entrada anterior) renomeou os containers
(`flua-zabbix-web` → `mip-engenharia-zabbix-web`) e validou domínio/
API/contagem de host corretamente — mas **o datasource Zabbix
configurado DENTRO do próprio Grafana** (estado interno do Grafana,
guardado no banco dele, não no compose/Traefik) continuava apontando
pra URL antiga (`http://flua-zabbix-web:8080/...`). Isso não foi
pego pela validação da migração porque os testes da época confirmaram
os domínios/API respondendo, não o funcionamento fim-a-fim do Grafana
consultando o Zabbix através do datasource interno.

**Corrigido imediatamente**, direto via API do Grafana (`GET`
datasource completo por UID, troca só do campo `url`, `PUT` de volta
preservando todo o resto — nunca reescrever do zero, pra não perder
`jsonData`/autenticação já configurados). Confirmado com evidência
real, não só "salvou": `/health` do datasource retornou `Zabbix API
version 7.0.28` e uma consulta real via proxy do Grafana
(`apiinfo.version`) devolveu resposta real do Zabbix — dado voltando
a fluir de verdade, não só configuração trocada.

**Lição registrada pra não repetir**: qualquer migração de container
futura (rename de tenant, mudança de host, etc.) precisa checar não só
infraestrutura (compose/Traefik/DNS) mas também **configuração
interna de aplicações que referenciam outro container pelo nome**
(datasource do Grafana é um caso real confirmado agora; podem existir
outros — GLPI plugins com URL fixa, etc. — auditar caso a caso quando
a próxima migração acontecer, não assumir que só ficou faltando isso
uma vez).

## 2026-08-05 (cont.) — Prompt monstro: UI ainda "jogada", IA capada na página dedicada, termo técnico vazando pro cliente

Responsável do projeto: confirmou sequenciamento (frontend primeiro,
banco bem depois) e pediu VIP temporário de teste antes do cutover
real (não ir direto pro VIP de produção). Trouxe 3 achados reais com
print + investigação ao vivo (login real, screenshot real da tela de
IA da MIP):

1. **UI ainda mal distribuída** fora das abas já corrigidas — tela do
   Assistente de IA com espaço vazio enorme, links de
   auditoria/créditos amontoados no canto inferior esquerdo, botões
   de topo como texto solto. Confirmado com screenshot real.
2. **Página dedicada da IA capada em relação ao atalho** — achado
   real no código: `AiChatWorkspace.tsx` tem a maioria das funções do
   `AiAssistantDrawer.tsx`, mas o chip de anexo pendente nunca
   renderiza miniatura de imagem (só mostra texto) — o drawer já faz
   isso certo.
3. **Terminologia técnica vazando pro cliente** — varredura real
   achou: `InstanceCard.tsx` ("container parado", coluna "Container",
   "containers, volumes, regra de firewall"), `metrics/page.tsx`,
   `clientes/page.tsx` novo do MSP ("containers ausentes/parados"),
   mensagens de erro da IA mencionando "operações Docker" — tudo isso
   em telas que cliente (MSP ou final) vê.

Prompt entregue, "monstro", várias partes independentes:
`docs/PROMPT-CURSOR-monstro-ui-escopo-pendencias-2026-08-05.md` —
Parte 1 (checklist de composição de UI + aplicar em todas as telas
novas), Parte 2 (paridade real página dedicada × atalho da IA, começar
pela miniatura de imagem), Parte 3 (varredura completa de terminologia
técnica, trocar por linguagem de produto em tudo que é client-facing),
Parte 4 (finalmente executar o dashboard-templates-sem-limite, já
estava pronto desde 04/08), Parte 5 (corrigir checklist de cutover:
fechar porta da aplicação deixa de ser opcional + etapa de VIP
temporário de teste antes do real), Parte 6 (reavaliar itens C/A de
`SERVICE-ACCOUNTS.md`, item B continua fora sem autorização).

**Seguem comigo, ainda não entregues** (repetido de sessões
anteriores, preciso realmente fechar isso, não só repetir que é meu):
relatório de segurança executivo (§21) e RUNBOOK de onboarding (§19).
