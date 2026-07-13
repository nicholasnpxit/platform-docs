# Roadmap — npx-platform

Itens ainda não implementados, para não perder contexto entre sessões.
Nada aqui está em progresso — é backlog. Última atualização: 2026-07-12.

> A fundação do portal (auth + modelo de tenants/usuários/instâncias +
> isolamento entre tenants) **já foi implementada** — ver
> `docs/STATE.md` e `docs/portal/ARCHITECTURE.md`. Os itens abaixo são o
> que ainda falta construir em cima dessa fundação.

## Portal de gestão multi-tenant

- Provisionamento self-service de instâncias (Zabbix/Grafana/GLPI) direto
  pelo painel — hoje isso é feito manualmente por sessão (como a stack
  `demo` e a `flua`, criadas à mão).
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

## Como usar este arquivo

Quando qualquer item acima for implementado (mesmo que parcialmente), mover
o registro daqui para `docs/STATE.md` (o que está pronto) e/ou
`docs/ARCHITECTURE.md` (como foi construído), e apagar a entrada
correspondente deste roadmap. Este arquivo deve conter só o que **ainda não
existe**.
