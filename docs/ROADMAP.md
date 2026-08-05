# Roadmap — npx-platform

> **Antes de ler este roadmap técnico**, leia `docs/ROADMAP-MACRO.md` — é a
> referência estratégica (metas comerciais, modelo de negócio, hierarquia
> de tenant, catálogo de produto) que orienta e dá contexto a todo item
> listado aqui embaixo. Um item técnico que não se encaixa em nenhuma
> seção do macro é sinal de que um dos dois documentos está desatualizado.

Itens ainda não implementados, para não perder contexto entre sessões.
Nada aqui está em progresso — é backlog. Última atualização: **2026-08-03
(prompt CRM+fechamento: fases 0–5 concluídas; Fase 6 avaliada sem ativar)**.

## Concluído em 2026-08-03 — CRM + fechamento (não é mais backlog)

| Tema | Status | Evidência |
|---|---|---|
| CRM ADMN `/crm` (contratos, campanhas, meta MRR, status comercial) | **Concluído** | `sessao-crm-fase0-2026-08-03/` |
| Extensão `sale_items` (editar/desativar) | **Concluído** | idem + `/sales/items/[id]` |
| Manutenção automática de disco (build cache/órfãos/Zabbix/NOC) | **Concluído** | `sessao-disco-fase4-2026-08-03/` |
| Chatwoot IA comercial/técnica + handoff | **Concluído** | `sessao-fechamento-fase1-2026-08-03/` |
| Inadimplência e-mail 5/15/25 + suspend 31 | **Concluído** | `sessao-fechamento-fase2-2026-08-03/` |
| Backup agendado (cron + UI) | **Concluído** | `sessao-fechamento-fase3-2026-08-03/` |
| Trial/demo 30d + autolimpeza (sem VM nova) | **Concluído** | `sessao-fechamento-fase5-2026-08-03/` |
| CrowdSec + Pi-hole no catálogo | **Avaliado, NÃO ativado** | `sessao-fechamento-fase6-2026-08-03/` |

## Concluído na sessão 42 (2026-07-30) — não é mais backlog

| Tema | Status | Evidência |
|---|---|---|
| Migração stacks restantes edge→`*_internal` | **Concluído** | `sessao-42-2026-07-30/01-*` |
| PDF 1-clique orçamento + relatório | **Concluído** | `02-quote.pdf` / `02-exec.pdf` |
| i18n telas comerciais | **Concluído** | `03-*` + enforce OK |
| Backups NPX “Carregando…” sob concorrência | **Concluído** | `04-backups-npx-*` |
| Branding login + e-mail reset white-label | **Concluído** | `05-*` |
| Auditoria E2E comercial/ops (`commercial_audit`) | **Concluído** | `06-commercial-audit-final.txt` |

## Concluído na sessão 41 (2026-07-30) — não é mais backlog

| Tema | Status | Evidência |
|---|---|---|
| Crédito IA OpenRouter (criar/consumir/bloquear no limite) | **Concluído** | `sessao-41-2026-07-30/01-ai-credit-limit-v2.txt` |
| Múltiplas instâncias do mesmo tipo / tenant | **Concluído** (valid1: `uptime_kuma` + `uptime_kuma-2`) | `02-two-instances.txt` |
| FINAL vs MSP na UI | **Concluído** (reconfirmado) | `03-account-*.png` |
| FLUA (MSP) / MIP (subtenant) visual | **Concluído** (reconfirmado) | `04b-picker-search-mip.png` |
| Docker Socket Proxy p/ kopia-agent | **Concluído** | `05-*` |
| Isolamento lateral cross-tenant | **Concluído** (+ migração edge→internal) | `06-*` |
| Seletor idioma = ícone globo | **Concluído** | `13-language-selector.png` |
| Backups “Carregando…” (causa: startTransition async) | **Concluído** (FLUA) | `14b-backups-flua.png` |
| Objetos comerciais (ticket GLPI, tempo, sale item, quote, desconto) | **MVP concluído** | ticket GLPI #535; tabelas SQL |
| Dashboard GLPI clicável | **MVP concluído** | `GlpiServiceDeskCards` |
| Relatório executivo + cron schedule | **MVP concluído** | `/tenants/[id]/reports/executive` |
| Analytics IA (`ai_action_log`) | **MVP concluído** | `/settings/ai/analytics` |
| Redis rate limit multi-réplica | **Concluído** | `29-rate-limit-multi.txt` |
| Destinos nuvem Kopia/rclone (doc + tipos + compose) | **Parcial** — OAuth mount pendente | `docs/BACKUP-CLOUD-DESTINATIONS.md` |
| Seletor tenant com busca + saúde | **Concluído** | `04b-picker-search-mip.png` |
| BookStack NPX produto + 8 manuais | **Concluído** | livro id 49 + Nextcloud manual |
| Chatwoot NPX volume real | **Concluído** | `35-chatwoot-rails.txt` |

## Revisão de segurança do Traefik como ponto único compartilhado — registrado 2026-07-29, NÃO implementado

Pedido explícito (FASE 5 sessão enforcement): investigar se continuar
dependendo do Traefik como ponto único HTTP(S) compartilhado para todas
as instâncias é a escolha certa a longo prazo, ou se vale avaliar
alternativa (ex: proxies por tenant, mesh, edge separado), **antes** de
continuar empilhando middlewares críticos (IP allowlist, SSO headers,
timeouts longos) nele. Inclui ameaça de configuração cruzada entre
tenants e blast radius de um bug no Traefik.

**Só registrar — não implementar nesta sessão.**

## Auditoria de segurança completa da plataforma e do código — registrado 2026-07-29, NÃO implementado

Revisão rigorosa futura: fragilidade real no portal, isolamento
multi-tenant, segredos, superfície de API, e proteção contra
cópia/engenharia reversa da ideia do produto (não só “checklist OWASP
genérico”). Escopo grande; depende de janela dedicada com o responsável.

**Só registrar — não implementar nesta sessão.**

## Integração OAuth real LinkedIn/Instagram (perfil) — discussão futura, registrado 2026-07-29, NÃO implementado

Hoje: validação de formato de URL + HTTP ≠ 404 (sem prova de propriedade).
Antes de investir em apps OAuth / verificação de identidade, validar se
há **valor de produto comprovado** (confiança, onboarding, whitelabel)
que justifique client_id/secret, manutenção e UX. Se não houver sinal
claro de venda, manter só validação de link.

## VPN / acesso à rede interna do cliente — CANCELADO 2026-07-29

Não vamos conectar na rede interna do cliente via VPN. Substituído por
lista de IPs/CIDRs WAN permitidos por tenant (implementado na FASE 4 da
mesma sessão).

## Customização de dashboard por widget — CONCLUÍDO em 2026-08-03 (onda §3.8)

MVP em `/tenants/<id>/board` (catálogo + reorder + persistência
user+tenant). Export PDF estilo Acronis ainda não.

## SSO do GLPI — CONCLUÍDO em 2026-08-03 (onda §3.9)

oauth2-proxy em `sso.<dominio-glpi>`; Host principal permanece com login
local. Ativação live exige IdP OIDC do tenant. Ver `docs/STATE.md` /
`lib/sso.ts` (`applyGlpiSso`).

**Quando priorizar:** decisão de arquitetura (qual proxy usar, se um
componente compartilhado com roteamento por tenant ou um por tenant) fica
pra quando isso for de fato priorizado — não decidido agora.

## Reset de senha por SMS — descartado em 2026-07-16, NÃO implementado

Pedido condicional: implementar só se existir um provedor de SMS
genuinamente gratuito. Pesquisado antes de escrever qualquer código —
conclusão: **não existe** provedor de SMS confiável com nível gratuito
real em escala de produção. Twilio, AWS SNS, Vonage, MessageBird — todos
cobram por mensagem enviada, sem exceção pra uso "de verdade" (só
créditos de trial limitados, não sustentáveis). As únicas opções
literalmente gratuitas encontradas são inadequadas por natureza: serviços
tipo "1 SMS grátis por dia por IP" (inviável pra uso real) ou gateways de
e-mail-pra-SMS de operadora (ex: número@txt.operadora.com) — frágeis
(depende de saber a operadora exata de cada número, opera-doras mudam/
descontinuam esses gateways sem aviso, entrega não confiável) e cada vez
mais bloqueados pelas próprias operadoras como vetor de spam. Não é
adequado pra um fluxo de segurança (redefinição de senha).

**Decisão:** não implementado. Se um provedor pago for aceitável no
futuro (mesmo que custo baixo, ex: SNS ~US$0,0075/SMS), essa é uma
decisão de negócio pro responsável do projeto tomar explicitamente —
registrar aqui quando/se isso mudar.

## Biblioteca de templates GLPI — fora de escopo do v1 (Fase 5, 2026-07-13)

Grafana e Zabbix já têm biblioteca v1 (ver `docs/templates/`). GLPI ficou
de fora por decisão explícita de escopo: não existe um artefato portável
único (JSON/YAML importável) equivalente a um dashboard Grafana ou
template Zabbix — a configuração de um GLPI (categorias, SLAs, campos
customizados) vive em objetos de Entity, replicáveis só via várias
chamadas de API específicas por Entity ou manipulação direta de banco.
Detalhes completos em `docs/templates/GRAFANA-TEMPLATES.md` (seção "GLPI
— fora de escopo").

> A fundação do portal (auth + modelo de tenants/usuários/instâncias +
> isolamento entre tenants) **já foi implementada** — ver
> `docs/STATE.md` e `docs/portal/ARCHITECTURE.md`. Os itens abaixo são o
> que ainda falta construir em cima dessa fundação.

> Provisionamento self-service de instâncias (Zabbix/Grafana/GLPI) direto
> pelo painel **já foi implementado** — ver `docs/STATE.md` e
> `docs/portal/ARCHITECTURE.md` (seção "Provisionamento self-service").

## Portal de gestão multi-tenant

> Domínio próprio, proxies Zabbix, métricas por tenant, export de logs —
> **concluídos em 2026-08-03** (onda §3.1/3.4/3.6/3.7). E2E Let's Encrypt
> com domínio descartável real ainda bloqueado (sem DNS controlável).

> Branding por tenant (cor/logo/favicon/tema) **já foi implementado** —
> ver `docs/STATE.md` (Fase 2) e `docs/portal/BRANDING.md` para a matriz
> real de capacidades e limites por ferramenta.

> Módulo de integração genérico entre apps (status de saúde +
> reconectar), por tenant, extensível a apps futuras — **já foi
> implementado** em 2026-07-15 — ver `docs/STATE.md` e
> `docs/portal/ARCHITECTURE.md` (seção "Módulo de integração genérico").

## SSO — investigação (2026-07-13, Fase D) — NÃO implementado, só diagnóstico

Investigado ao vivo (código-fonte + config + API real dos containers
rodando), sem implementar nada — decisão de arquitetura fica para o
responsável do projeto.

| Ferramenta | Suporte real | Evidência |
|---|---|---|
| **Grafana OSS 13.0.2** | ✅ OIDC/OAuth2 genérico **nativo, de graça** (`[auth.generic_oauth]` existe e funciona no OSS — SAML é que é Enterprise-only, mas não é o que precisamos) | Seção completa presente em `defaults.ini` dentro do container rodando |
| **Zabbix 7.0.28** | ✅ SAML 2.0 **nativo, de graça** — Zabbix não tem split Community/Enterprise (é 100% GPL, monetizam só suporte pago) | Biblioteca `onelogin/php-saml` presente no container; API `authentication.get` já expõe `saml_auth_enabled`, `saml_jit_status` (provisionamento automático de usuário no primeiro login) como campos nativos |
| **GLPI 11.0.8** | ❌ **sem** OIDC/SAML nativo no core (só Local/LDAP/CAS/Mail) — **mas** tem um mecanismo nativo de "confiar num header/variável de servidor" (`glpi_ssovariables`: `REMOTE_USER`, `HTTP_AUTH_USER`, etc.), com comentário no próprio código-fonte citando "e.g. OAuth SSO" como caso de uso pretendido | `Auth.php` linhas ~538-583, ~1434-1523; tabela `glpi_ssovariables` já vem com 6 valores pré-cadastrados, hoje desativada (`ssovariables_id=0`). Marketplace não pôde ser consultado (exige chave de registro GLPI Network que não temos) — não descarto plugin comunitário de OIDC existir, mas não confirmei nenhum ao vivo, então não vou citar um como se fosse fato. |

**Recomendação:** os três dão para conectar num Keycloak central, mas com
pesos diferentes de esforço:
- **Grafana e Zabbix**: configuração direta (client OIDC/SAML no Keycloak
  + preencher os campos nativos de cada um). Trabalho relativamente
  pequeno.
- **GLPI**: não dá para conectar direto no Keycloak via OIDC/SAML nativo.
  O caminho realista é colocar um **proxy de autenticação** na frente
  dele (ex: oauth2-proxy, Authelia, ou Traefik ForwardAuth com um serviço
  OIDC) que faz o handshake com o Keycloak e injeta um header tipo
  `REMOTE_USER` — o GLPI já sabe confiar nesse header nativamente. Isso é
  **infraestrutura nova** (mais um componente rodando), não só
  configuração dentro do GLPI.
- **Portal**: hoje usa JWT próprio (`docs/portal/ARCHITECTURE.md`) — para
  entrar no esquema também precisaria virar cliente OIDC do Keycloak
  (trabalho novo, não trivial, mexe na Fase 1 de auth já construída).

Se o objetivo é *single sign-on de verdade* (uma sessão só, entre painel +
as 3 ferramentas), Keycloak é o caminho certo, mas o GLPI exige um
componente de infraestrutura a mais (não é só apertar um botão de config).
Se o esforço do GLPI não valer a pena agora, a alternativa realista mais
simples é **"same sign-on"**: mesmo usuário/senha em todo canto (já é
quase isso hoje, já que o time administra todas as credenciais via
`docs/ACCESS.md`), sem sessão única — cada ferramenta continua com seu
próprio login separado.

## Decisões de arquitetura já fixadas (valem para todo o roadmap acima)

- **Isolamento de rede sempre via Docker, nunca VLAN física** — decisão
  tomada por portabilidade entre hosts. Qualquer feature de isolamento
  multi-tenant deve continuar usando redes Docker por stack (como já é
  feito em `demo` e `flua`), não infraestrutura de rede física.
- **Independência de faixa de IP pública** — a arquitetura já roteia por
  hostname via Traefik, não depende de IP fixo. Isso já vale hoje (troca de
  IP público não quebra nada, já que DNS + Host() rules é o que importa) e
  deve continuar sendo o princípio para qualquer evolução futura (ex: se um
  cliente migrar de IP, ou se o próprio NPX trocar de provedor).

## Integração com WhatsApp (alertas e atendimento) — AUTOMACAO IMPLEMENTADA 2026-08-03

**Codigo pronto** (GUI self-service por admin do tenant + relay + webhook + wire Zabbix/Grafana/Chatwoot +
hook GLPI) — evidência `sessao-whatsapp-2026-08-03/`. Provedor: **WhatsApp
Cloud API oficial (Meta)**. Falta so: (1) colar credenciais sandbox/producao
em `/opt/npx-platform/whatsapp/.env` ou na GUI; (2) Business Verification e
aprovacao de templates — acoes humanas na Meta, nao bloqueio de codigo.
Nao usar gateway nao-oficial (Baileys/whatsapp-web.js).

### Nível 1 — Alertas de saída (Zabbix e Grafana), esforço médio

- **Zabbix**: media type tipo Webhook (mesmo padrão já usado para a
  integração GLPI, ver `docs/STATE.md` — Fase 3 do cliente FLUA) chamando
  a WhatsApp Cloud API diretamente (`POST
  https://graph.facebook.com/v21.0/<phone_number_id>/messages`),
  autenticação via token de **System User** do Meta Business (token de
  longa duração, não o token de usuário comum que expira em horas).
- **Grafana**: contact point tipo `webhook` apontando pro mesmo endpoint
  da Cloud API — Grafana já tem suporte nativo a contact points webhook
  genéricos.
- **Restrição real da própria Meta, não nossa**: mensagens iniciadas pela
  empresa (não em resposta a uma mensagem do cliente) só podem sair
  **dentro de uma janela de 24h** desde a última mensagem do usuário, OU
  usando um **template de mensagem pré-aprovado pela Meta**. Para alertas
  de monitoramento isso significa **sempre precisar de um template
  aprovado** — não dá pra simplesmente mandar texto livre.
- **Aprovação de template não é instantânea** — prazo típico de horas a
  dias, e pode ser rejeitada por política de conteúdo. Dependência
  externa fora do controle deste projeto.
- **Cada tenant precisa da própria conta comercial verificada no Meta
  Business e do próprio número de telefone** — token + `phone_number_id`
  configuráveis por tenant no portal. O processo de verificação comercial
  é responsabilidade do próprio cliente final, não algo que a NPX ativa
  ou credencia sozinha.

### Nível 2 — Atendimento via WhatsApp no GLPI (conversa de chamado), esforço alto

- Canal de entrada via webhook da Cloud API, associando telefone a
  usuário/tenant, abrindo/atualizando chamado via API do GLPI,
  respondendo via Cloud API.
- **Decisão de UX em aberto, não decidir sozinho quando chegar a hora**:
  técnico responde pelo GLPI (replicado automático) vs. tela de conversa
  dedicada — trade-offs bem diferentes, fica para quando o responsável
  priorizar este nível.
- Mesma restrição de janela de 24h/template da Meta quando a NPX inicia a
  conversa.

### Pendência de decisão (não decidida agora)

Se o Nível 2 entra junto com o Nível 1 ou fica para uma fase separada —
em aberto para o responsável do projeto decidir quando a implementação
for de fato priorizada.

## Múltiplas instâncias do mesmo tipo por tenant — IMPLEMENTADO em 2026-07-27

Registrado em 2026-07-15, implementado e testado de ponta a ponta na
sessão longa de 2026-07-27 (Fase 3). `Instance` ganhou `slug` (técnico,
imutável, com sufixo a partir da 2ª instância do mesmo tipo) e `nome`
(apelido de exibição, opcional). Detalhe completo em
`docs/DECISIONS.md` ("FASE 3: múltiplas instâncias do mesmo tipo por
tenant") e `docs/portal/ARCHITECTURE.md`.

## Vaultwarden e Uptime Kuma como tipos de instância provisionáveis — IMPLEMENTADO em 2026-07-27

Registrado em 2026-07-15, implementado e testado de ponta a ponta na
sessão longa de 2026-07-27 (Fase 2 do catálogo). Ambos seguem o mesmo
padrão dos tipos anteriores (stack isolada, Traefik, Let's Encrypt,
`suporteti` automático). Uptime Kuma também ganhou pré-cadastro
automático de monitores das outras instâncias do tenant. Detalhe
completo em `docs/DECISIONS.md` e `docs/STATE.md` (entradas "FASE 2:
catálogo completo").

<!-- Notas históricas da investigação original de 2026-07-15, mantidas
pra contexto de por que não foi feito na hora: -->

**Achado ao investigar (Fase A11, 2026-07-15):** não existia registro
de que isso tinha sido pedido ou iniciado em nenhuma sessão anterior
àquela data — toda menção a "Vaultwarden"/"Uptime Kuma" era só exemplo
ilustrativo de app futura, nunca tarefa real.

**Detalhe técnico do Uptime Kuma que se confirmou verdadeiro na
implementação real:** SQLite embutido, sem variável de admin inicial
via env (o setup do admin/senha acontece na primeira visita à UI —
diferente do padrão dos outros três). A automação real usada foi via
protocolo Socket.IO interno (não manipulação direta do banco SQLite) —
ver `docs/DECISIONS.md` pro bug real encontrado nesse caminho (webpack
quebrando `socket.io-client`) e a correção aplicada.

## Provisionamento multi-host — suporte a mais de um servidor Docker — registrado em 2026-07-14, NÃO implementado

Motivado originalmente só pelos requisitos de recurso do Wazuh (ver
`docs/DECISIONS.md` — Wazuh pede, só pelo fabricante, 8GB RAM/4 CPU por
instância single-node). **Motivo ampliado em 2026-08-05**: com a
segregação de infraestrutura em andamento (`docs/ROADMAP-MACRO.md`
§22 — VMs `vsadmnfront`/`vsadmnapp`/`vsadmndb`), e a visão de longo
prazo do responsável do projeto de eventualmente operar múltiplos
servidores/datacenters/regiões (registrado, explicitamente **não**
pra construir agora — foco atual é bater 100 instâncias com a
topologia de 3 VMs), este item deixa de ser só "quando o Wazuh
precisar" e passa a ser relevante assim que existir mais de uma VM do
mesmo papel (mais de uma VM de aplicação, por exemplo). Continua **não
implementado** — só o registro do motivo mais amplo.

**Estado atual (o que muda):** `portal/src/lib/portainer.ts` hoje tem
`PORTAINER_URL` e `ENDPOINT_ID` fixos (`ENDPOINT_ID = 1`, comentado no
código como "único ambiente Docker deste host") — toda chamada de
provisionamento assume que existe só um Portainer/host de destino. O
modelo `Instance` (`portal/prisma/schema.prisma`) hoje não guarda em
qual host/ambiente uma instância roda — só a URL pública dela.

**O que precisa ser construído, quando a VM existir:**

1. **Múltiplos ambientes Portainer cadastrados** — hoje é uma constante
   fixa no código; precisa virar configuração (provavelmente uma tabela
   nova, ex. `DockerHost`: nome, URL do Portainer, endpoint id, talvez
   credenciais próprias se a VM dedicada tiver um Portainer separado em
   vez de um endpoint adicional no mesmo Portainer atual — as duas
   formas são tecnicamente possíveis com a API do Portainer, decisão de
   qual usar fica pra quando a VM existir e dermos load real nisso).
2. **Escolha automática de host por tipo de serviço** — regra simples
   tipo "Wazuh sempre vai pra VM dedicada, os demais (Zabbix/Grafana/GLPI)
   continuam na principal" é suficiente pro caso conhecido hoje; desenhar
   isso de um jeito que não exija reescrever tudo se um dia houver mais
   de duas VMs (ex: mapa `tipo → host` configurável, não `if` hardcoded).
3. **`Instance` precisa registrar em qual host cada instância vive** —
   campo novo (ex. `dockerHostId`, relação com a tabela de hosts do item
   1) — sem isso o painel não sabe pra qual Portainer perguntar
   status/métricas de uma instância específica depois de provisionada.
   Esse campo deveria existir mesmo enquanto só há um host (aponta pro
   único host cadastrado), pra migração não exigir backfill manual
   quando o segundo host aparecer.

**Não implementado:** nenhum código, nenhuma tabela nova, nenhuma
mudança em `portainer.ts` — só esta decisão de arquitetura documentada,
como pedido, para não perder o raciocínio até a VM existir.

## Backup e restauração granular por instância (complementar ao Acronis) — registrado em 2026-07-15, NÃO implementado

Só documentação de intenção/arquitetura, como pedido — nenhum código
escrito, nenhum provedor de armazenamento pesquisado/contratado ainda.
Complementar ao backup de infraestrutura já existente via Acronis
(nível de host/VM), não um substituto — este item é granular, por
instância, controlado pelo próprio cliente dentro do tenant dele.

### Contexto de produto

- O cliente vê e controla isso dentro do próprio tenant, sempre
  enquadrado como **"instâncias"** — nunca expor os termos
  "container"/"stack" pra ele em nenhuma tela.
- Plano padrão inclui backup por N dias de retenção e/ou um limite de
  tamanho; upgrade pago aumenta esses limites.
- Cliente pode disparar um backup manual a qualquer momento (ex: antes
  de uma tarefa arriscada na própria instância) e restaurar depois.
- Isolamento: cliente só vê e restaura o que é dele. A NPX (tenant
  mestre) vê e administra tudo, de todos os tenants.

### Backup granular (Kopia) — motor e telas já IMPLEMENTADOS

O planejamento original desta seção (motor Kopia, dump lógico pré-backup,
telas de tenant/mestre, restauração overwrite/cópia, Postgres do portal
incluído) foi **implementado e validado com dado real em 2026-07-27**
(`docs/STATE.md`, "FASE 1: backup granular por instância") e **retestado
com um restore vivo de ponta a ponta em 2026-07-28** (mesmo arquivo,
entrada "teste vivo real de restauração Kopia"). Detalhe de arquitetura
completo em `docs/portal/ARCHITECTURE.md`. O único ponto do planejamento
original que ficou de fato pendente é o destino de armazenamento
configurável — ver entrada nova abaixo.

## Destino de armazenamento do backup Kopia configurável (ADMN-only) — PARCIAL 2026-08-03

Rede/NAS/SFTP + objeto cloud na tela ADMN **feito** (onda §3.5). Pendente
só OAuth live OneDrive/GDrive (fora de escopo desta rodada). Ver
`docs/BACKUP-CLOUD-DESTINATIONS.md` e `docs/STATE.md`.

## Como usar este arquivo

Quando qualquer item acima for implementado (mesmo que parcialmente), mover
o registro daqui para `docs/STATE.md` (o que está pronto) e/ou
`docs/ARCHITECTURE.md` (como foi construído), e apagar a entrada
correspondente deste roadmap. Este arquivo deve conter só o que **ainda não
existe**.

## Certificado próprio do cliente por instância — CONCLUÍDO em 2026-08-03

Implementado (onda grande §3.3): `lib/custom-tls.ts`, `Instance.tlsMode`,
UI na InstanceCard, file provider Traefik. Evidência de isolamento LE vs
custom em `docs/STATE.md`. Pendência residual: política automática de
expiração/revogação (hoje só botão manual “Voltar para Let's Encrypt”).

---

## Domínio-base configurável pelo tenant ADMN — CONCLUÍDO em 2026-08-03

Mecanismo em `/settings/platform` + `PlatformSettings.deliveryDomainBase`
(onda grande §3.2). Domínio real de entrega **ainda não comprado** —
valor continua `instancias-teste.example` até o responsável decidir.
Detalhe e evidência em `docs/STATE.md`.

## Ação "Registrar instância existente" (self-service, ADMN) — CONCLUÍDA em 2026-07-27

**Motivação (incidente real):** quando o registro de rastreamento de uma
instância no banco do portal precisava ser recriado sem tocar em
infraestrutura (ex: linha apagada por engano, ou ferramenta legada
nunca cadastrada), a única forma era `INSERT` SQL manual — viola o
princípio de zero intervenção manual (`CLAUDE.md` / `ROADMAP-MACRO`
seção 1).

**Implementado:** tela `/tenants/[id]/instances/register` (ADMN-only) com
duas ações:
1. **Nova ficha** — grava `instances` sem provisionar; confirma via
   Portainer que o container principal esperado existe antes de gravar.
2. **Corrigir prefixo** — atualiza `containerPrefix` de ficha já
   existente (caso real: tenant `npx` com containers `demo-*`).

Campo `Instance.containerPrefix` + uso em métricas/backup/exclusão/
diagnóstico. Detalhe em `docs/DECISIONS.md` (FASE 4 desta sessão) e
`docs/portal/ARCHITECTURE.md`.

## Pendente — observabilidade IA (OpenRouter Broadcast) — 2026-07-28

Investigado, **não implementar agora**. OpenRouter Broadcast envia traces
OTLP para Grafana Cloud Tempo **ou** para um OpenTelemetry Collector /
Tempo self-hosted. Nosso Grafana self-hosted (`npx-grafana` /
grafana-master) **não** tem Tempo. Opções futuras:

1. Conta Grafana Cloud (OTLP gateway + traces:write) + Broadcast no painel OpenRouter — menor esforço operacional, custo cloud.
2. Subir Tempo (+ eventualmente OTEL Collector) no stack NPX e apontar Broadcast para ele — mais controle, mais operação.

Critério para retomar: quando o volume de uso de IA por tenant justificar
dashboard de custo/latência/erro além do que já temos em créditos
OpenRouter no portal.

## Pendente — DNS vitrine/NOC externo (Azure Microsoft 365)

Criar registros A → `187.110.164.126` para:
`uptime.npx.npxit.com.br`, `docs.npx.npxit.com.br`, `chat.npx.npxit.com.br`.
Sem isso a status page e o widget Chatwoot não são demonstráveis por URL
pública com certificado Let's Encrypt (mesmo bloqueio de
grafana-master/zabbix-master). Ideal futuro: API Azure DNS no portal
(zero passo manual).

## Pendente — WhatsApp no Chatwoot vitrine

Depende de aprovação da API oficial da Meta. Canal site já criado;
WhatsApp fica para sessão futura após aprovação.



## Backlog — Authelia / auth prévia em rotas ADMN sensíveis (FASE M, 2026-07-29)

**Não implementar agora.** Sessão A–M avaliou: já há sessão JWT + `isAdmn` + rate limit Postgres + Turnstile opcional.
Custo de IdP + Traefik forwardAuth + operação contínua não se justifica com 1 operador ADMN.
Reavaliar quando houver equipe >1 ou compliance explícito de cliente enterprise.
