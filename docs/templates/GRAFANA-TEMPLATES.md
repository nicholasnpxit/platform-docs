# Biblioteca de templates — Grafana (v1, Fase 5)

Última atualização: 2026-07-13.

## Mudança de plano em relação ao esboço original (documentado por
transparência)

O plano original desta fase previa usar a API pública
`grafana.com/api/dashboards?tag=zabbix` para buscar dashboards prontos da
comunidade. **Testado ao vivo nesta sessão e o contrato da API mudou** —
hoje ela responde `409 Unexpected parameter: tag` para essa forma de
busca (a API pública do grafana.com evoluiu desde que esse plano foi
escrito). Em vez de investir tempo tentando descobrir a nova forma de
busca (dependência de rede externa, sem controle de versão nosso), usei
uma fonte **melhor e já disponível localmente**: o próprio plugin
`alexanderzobnin-zabbix-app` (que já instalamos em todos os Grafanas)
**vem com 3 dashboards prontos, oficiais, embutidos na imagem do
plugin** — sem depender de internet, garantidamente compatíveis com a
versão do plugin instalada.

## Dashboards da biblioteca (extraídos do plugin, versionados no repo)

Fonte: `/var/lib/grafana/plugins/alexanderzobnin-zabbix-app/datasource/dashboards/`
dentro de qualquer container Grafana com o plugin instalado. Copiados
para `/opt/npx-platform/templates/grafana/`:

| Arquivo | Título | Uso recomendado |
|---|---|---|
| `zabbix_server_dashboard.json` | Zabbix Server Dashboard | Saúde do próprio Zabbix server (fila de processamento, performance interna) — funciona em **qualquer** cliente novo, não depende de hosts/agentes reais já cadastrados |
| `zabbix_system_status.json` | Zabbix System Status | Visão geral de status do sistema monitorado |
| `template_linux_server.json` | Zabbix Template Linux Server | Dashboard pronto para hosts linkados ao template oficial `Linux by Zabbix agent` — só relevante quando o cliente tiver hosts Linux reais com agente instalado (nenhum tenant tem isso ainda nesta plataforma) |

Todos os 3 declaram `__inputs: [{name: "DS_ZABBIX", ...}]` — precisam de
um datasource Zabbix mapeado no import.

## Prova de conceito — testado ao vivo

Importado `zabbix_server_dashboard.json` via
`POST /api/dashboards/import` (com `inputs` mapeando `DS_ZABBIX` para o
datasource já existente de cada tenant) em:

- **`demo-grafana`** → `/d/fe0ylyg4cdd6oc/zabbix-server-dashboard` —
  confirmado `200`, 7 painéis carregados.
- **`flua-grafana`** → mesma URL/uid (`fe0ylyg4cdd6oc`, Grafana gera uid
  a partir do dashboard original, igual nos dois porque veio do mesmo
  arquivo fonte).

**Atenção de segurança (relevante por causa da Fase 4):** como os dois
Grafanas de cliente têm acesso anônimo Viewer habilitado, este novo
dashboard importado também fica visível anonimamente (mesmo escopo do
"NOC Overview" — é o mesmo Viewer role, não dá pra restringir por
dashboard individual no Grafana OSS). Conteúdo é só saúde operacional do
próprio Zabbix (fila, performance), não dado de cliente sensível — mesmo
nível de risco já aceito e documentado na Fase 4.

## Como importar em um cliente novo

```bash
curl -u admin:<senha> -X POST https://grafana.<slug>.npxit.com.br/api/dashboards/import \
  -H 'Content-Type: application/json' \
  -d '{
    "dashboard": <conteúdo do .json escolhido>,
    "overwrite": true,
    "inputs": [{"name":"DS_ZABBIX","type":"datasource","pluginId":"alexanderzobnin-zabbix-datasource","value":"<uid do datasource Zabbix do tenant>"}]
  }'
```

## GLPI — fora de escopo do v1 (registrado, não implementado)

Diferente de Grafana (dashboards são artefatos JSON portáveis,
importáveis via API em qualquer instância) e Zabbix (templates são
artefatos YAML/XML portáveis via `configuration.import`), o GLPI não tem
um conceito equivalente de "template importável" de forma nativa e
simples: a configuração de um GLPI (categorias de chamado, SLAs, campos
customizados, formulários) é feita via **Entity**, e replicar isso entre
clientes exigiria automatizar dezenas de chamadas de API específicas por
Entity (ou manipulação direta de banco, que já é um padrão que este
projeto só usa quando explicitamente autorizado — ver
`docs/DECISIONS.md`), não um único arquivo de export/import como os
outros dois. Fica registrado em `docs/ROADMAP.md` como item futuro, não
implementado nesta fase por decisão explícita de escopo do responsável do
projeto.
