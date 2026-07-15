# Roadmap — npx-platform

Itens ainda não implementados, para não perder contexto entre sessões.
Nada aqui está em progresso — é backlog. Última atualização: 2026-07-14.

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

- Gestão de proxies Zabbix por tenant (para clientes com múltiplas
  unidades/localidades), configurável direto pelo painel.
- Domínio próprio configurável por cliente, com Let's Encrypt automático
  (hoje isso já funciona no nível do Traefik para subdomínios de
  `npxit.com.br` — falta a camada de painel que permita ao cliente
  registrar um domínio totalmente próprio e disparar a emissão sozinho).
- Coleta e exportação de logs/erros por instância.
- Métricas de disco/CPU/memória por instância, escopadas ao próprio tenant
  (cada cliente só vê os números da sua própria stack — hoje a Fase 3 já
  coleta isso a nível de host inteiro via `monitoring/npx-zabbix`, mas
  ainda não está escopado/filtrado por tenant dentro do portal).

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

## Integração com WhatsApp (alertas e atendimento) — registrado em 2026-07-13, NÃO implementado

Só documentação de intenção/arquitetura — nenhum código escrito, nenhuma
conta criada no Meta Business. Provedor **já decidido** pelo responsável
do projeto: **WhatsApp Cloud API oficial (Meta)** — não um gateway
não-oficial (ex: Baileys/whatsapp-web.js), decisão que evita o risco de
banimento de número que gateways não-oficiais correm.

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

## Múltiplas instâncias do mesmo tipo por tenant — registrado em 2026-07-15, NÃO implementado

Motivado pelo exemplo dado na Fase 3 (sistema de cota): "Vaultwarden: 2"
pressupõe que um tenant possa ter mais de uma instância do mesmo tipo.
Hoje `Instance` tem `@@unique([tenantId, tipo])` — trava física em 1.
`TenantQuota.maxInstancias` já aceita qualquer inteiro (schema pronto),
mas a aplicação de N>1 não foi construída: exigiria revisar nomenclatura
de domínio (`suggestDomain`), nome de container (`compose-templates.ts`)
e possivelmente porta de trapper, todos hoje sem índice. Fica pra quando
um tipo com caso de uso real de N>1 (Vaultwarden ou outro) for de fato
implementado — ver `docs/DECISIONS.md` (entrada 2026-07-15) para o
raciocínio completo.

## Provisionamento multi-host — suporte a mais de um servidor Docker — registrado em 2026-07-14, NÃO implementado

Motivado pelos requisitos de recurso do Wazuh (ver `docs/DECISIONS.md` —
Wazuh pede, só pelo fabricante, 8GB RAM/4 CPU por instância single-node,
piso que já consome quase toda a folga hoje disponível no host principal
— ver achado real em `docs/STATE.md`). Quando a decisão de negócio for
tomada e uma VM dedicada pro Wazuh existir, o portal vai precisar deixar
de assumir um único host Docker/Portainer.

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

### Motor recomendado: Kopia (não Restic)

Kopia tem Repository Server com API REST própria, suporte nativo a
múltiplos usuários/credenciais e controle de acesso por usuário — mapeável
para "1 usuário Kopia = 1 instância ou 1 tenant" (avaliar a granularidade
certa na hora de implementar, não decidido ainda). Restic assume 1
repositório = 1 credencial só, o que exigiria construir toda a camada de
multi-tenant do zero por cima dele; Kopia já entrega parte disso pronto.

### Pontos a detalhar quando for implementar (nenhum decidido ainda)

- **Hook de pré-backup**: dump lógico do banco (`mysqldump` ou
  equivalente) antes de cada snapshot — nunca copiar o arquivo de banco
  vivo direto. Nenhuma ferramenta de backup genérica (Kopia incluído)
  entende estado de banco "ao vivo" de forma consistente.
- **Portal como único detentor de credencial de administração do
  Kopia** — o cliente nunca fala com o Kopia diretamente, só com a API
  do próprio portal (mesmo padrão já usado pro Portainer: nunca expor a
  ferramenta de infraestrutura crua a quem não deveria ter esse acesso).
- **Tabela nova de política por tenant** (dias de retenção e/ou cota de
  tamanho), configurável pelo tenant mestre por cliente/plano — mesmo
  padrão de "só o mestre configura" já usado pra `TenantQuota` (Fase 3,
  ver `docs/portal/ARCHITECTURE.md`).
- **Pesquisar depois** (não agora): 2-3 destinos de armazenamento
  compatíveis com Kopia (Backblaze B2, Wasabi, etc.) — comparar custo
  hoje e projetado pra ~20-30 clientes. Decisão de qual usar e o
  cadastro da conta são do responsável do projeto, não algo pra
  automatizar ou decidir sozinho.
- **Fluxo de restauração precisa cobrir dois casos**: sobrescrever a
  instância existente, ou restaurar como uma cópia nova pra inspecionar
  antes de decidir (não sobrescrever direto às cegas).
- **O Postgres do próprio portal entra na mesma disciplina** — não é só
  instância de cliente que precisa disso, o banco do portal (`portal-db`)
  também.
- **Telas necessárias**:
  - Tenant cliente: lista de backups da própria instância, botão
    "backup agora", botão "restaurar" com as duas opções do item acima.
  - Tenant mestre (NPX): visão de backups de todos os tenants +
    configuração de retenção por cliente.

## Como usar este arquivo

Quando qualquer item acima for implementado (mesmo que parcialmente), mover
o registro daqui para `docs/STATE.md` (o que está pronto) e/ou
`docs/ARCHITECTURE.md` (como foi construído), e apagar a entrada
correspondente deste roadmap. Este arquivo deve conter só o que **ainda não
existe**.
