# Biblioteca de templates — Zabbix (v1, Fase 5)

Última atualização: 2026-07-13.

## O que já existe pronto, sem precisar importar nada

O próprio pacote oficial `zabbix/zabbix-server-mysql:alpine-7.0-latest` já
vem com **356 templates oficiais** carregados no banco (confirmado via
`template.get` sem nenhum `configuration.import` — eles já existem assim
que o Zabbix sobe pela primeira vez). Isso cobre a grande maioria dos
cenários de onboarding de cliente sem precisar de nenhum arquivo externo.

Matriz de recomendação por tipo de ativo (nomes exatos confirmados no
catálogo local desta versão — usar exatamente esses nomes em
`template.get`/`host.update`):

| Tipo de ativo do cliente | Template(s) oficiais recomendados |
|---|---|
| Servidor Linux (com Zabbix agent instalado) | `Linux by Zabbix agent` (ou `Linux by Zabbix agent active` se o agente só conseguir se conectar de fora pra dentro) |
| Host Docker | `Docker by Zabbix agent 2` (usado no próprio monitoramento da NPX, Fase 3) |
| Servidor Windows | `Microsoft Exchange Server 2016 by Zabbix agent` (só se for Exchange) — para Windows genérico usar `Linux by Zabbix agent`-equivalente do lado Windows: não existe um "Windows by Zabbix agent" genérico neste catálogo; usar os itens nativos do agente Windows via template customizado (ver `IIS by Zabbix agent` para servidores web Windows/IIS especificamente) |
| Switch/roteador via SNMP (genérico) | `Generic by SNMP` |
| Switch/roteador de marca conhecida | Templates dedicados por fabricante/modelo já catalogados — ex: `MikroTik CCR1009-...`, `Cisco IOS by SNMP`, `HP Enterprise Switch by SNMP`, `Huawei VRP by SNMP`, `Juniper by SNMP` (ver lista completa via `template.get`, são **~100+** variações só de rede) |
| No-break/UPS | `APC UPS by SNMP` (genérico) ou modelo específico (`APC Smart-UPS ...`) |
| Firewall FortiGate | `FortiGate by SNMP` ou `FortiGate by HTTP` (API REST do FortiOS) |
| Banco de dados MySQL | `MSSQL by Zabbix agent 2` é só para SQL Server — para MySQL usar item customizado ou o plugin nativo do Zabbix agent 2 para MySQL (não appareceu como template dedicado nesta busca; documentar como pendência se algum cliente precisar) |
| Serviço HTTP genérico | `Apache by HTTP`, `Nginx` (verificar nome exato se necessário), ou ICMP/porta simples via `ICMP Ping` |
| Nuvem (AWS/Azure/GCP) | Templates dedicados por serviço já catalogados (`AWS EC2 by HTTP`, `Azure Virtual Machine by HTTP`, `GCP Compute Engine Instance by HTTP`, etc.) |

**Como aplicar:** `template.get` com `filter: {host: ["<nome exato>"]}`
para pegar o `templateid`, depois `host.update` com `templates: [...]`
no host do cliente. Não precisa de `configuration.import` para nenhum
desses — já estão no banco.

## Template customizado NPX (demonstra o pipeline de `configuration.import`)

Criado nesta fase como prova de conceito **real e testada** (não só
teórica) do mecanismo que será usado para distribuir templates
autorais/customizados (que não vêm no catálogo oficial) entre os Zabbix
de todos os clientes:

1. Criado em `zabbix-master` (NPX): template **"NPX - Trapper Padrao"**
   (grupo `Templates/NPX`) — 1 item trapper (`npx.heartbeat`) + 1 trigger
   `nodata(...,10m)=1` ("NPX - Sem heartbeat recebido em 10 minutos").
2. Exportado via `configuration.export` (formato YAML) →
   `/opt/npx-platform/templates/zabbix/npx-trapper-padrao.yaml` (versionado
   no repo, é a fonte de verdade da biblioteca).
3. Importado via `configuration.import` nos Zabbix de **demo** e **FLUA**
   — comando idêntico, `rules` com `createMissing`/`updateExisting` em
   `template_groups`/`templates`/`items`/`triggers`.
4. Linkado ao host "Zabbix server" de `demo` (`host.update`).
   **Confirmado sem perda de dados**: o host tinha 121 itens próprios
   (não herdados de template nenhum, apesar do texto da Fase A sugerir o
   contrário — checado via `item.get` antes/depois, nenhum foi
   removido/alterado) — só o item novo (`templateid != 0`) foi
   adicionado.
5. **Testado ponta a ponta**: `zabbix_sender -k npx.heartbeat -o 1` →
   confirmado no `history.get` (valor `1` recebido, timestamp real).

**Como reusar para um cliente novo:** ler o YAML deste arquivo, chamar
`configuration.import` com o mesmo `rules` object, depois `host.update`
para linkar ao(s) host(s) desejado(s). Nenhum passo manual de UI
necessário.

## Fora de escopo do v1 (documentado, não implementado)

- Templates SNMP específicos por marca/modelo de equipamento de cliente
  real — o catálogo já cobre uma quantidade grande, mas escolher o
  específico exige saber o parque de hardware de cada cliente (não
  genérico o suficiente pra automatizar sem essa informação).
- Item MySQL/PostgreSQL dedicado nativo — não encontrado como template
  oficial pronto nesta busca; se algum cliente precisar, a rota é criar
  um customizado (mesmo padrão do "NPX - Trapper Padrao" acima) usando os
  plugins nativos do Zabbix agent 2 para bancos de dados.
