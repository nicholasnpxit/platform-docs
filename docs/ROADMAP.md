# Roadmap — npx-platform

Itens ainda não implementados, para não perder contexto entre sessões.
Nada aqui está em progresso — é backlog. Última atualização: 2026-07-13.

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
- Auto-integração e status de conexão entre serviços (Zabbix↔Grafana,
  Zabbix↔GLPI) com botão de reconectar/auto-correção quando algo cair.
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

## Como usar este arquivo

Quando qualquer item acima for implementado (mesmo que parcialmente), mover
o registro daqui para `docs/STATE.md` (o que está pronto) e/ou
`docs/ARCHITECTURE.md` (como foi construído), e apagar a entrada
correspondente deste roadmap. Este arquivo deve conter só o que **ainda não
existe**.
