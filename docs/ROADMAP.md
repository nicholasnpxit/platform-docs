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
  (cada cliente só vê os números da sua própria stack).
- Branding por tenant: cor, logo, favicon, fonte, nome — com fallback para
  o branding padrão **"Advanced Monitoring and Management - by NPX IT"**
  quando o tenant não definir o próprio.

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
