# Portal de gestão multi-tenant — Arquitetura (Fase 1: fundação)

Última atualização: 2026-07-12. Este documento descreve o que existe hoje —
para o que ainda falta construir (provisionamento self-service, proxies
Zabbix, branding avançado, etc.), ver `docs/ROADMAP.md`.

## Stack técnica

- **Next.js 14 (App Router)** + TypeScript, rodando em modo `standalone`.
- **PostgreSQL 16** (container `portal-db`, rede `internal` própria do
  portal — não compartilha banco com nenhum cliente).
- **Prisma** como ORM/migrations.
- Autenticação: **bcryptjs** (hash de senha) + **jose** (JWT em cookie
  httpOnly).
- E-mail (esqueci minha senha): **nodemailer** contra SMTP do Office 365.

Onde mora: `/opt/npx-platform/portal/`. Sobe atrás do Traefik em
`https://admn.npxit.com.br`, mesmo padrão de certificado (Let's Encrypt
produção) dos demais serviços.

## Modelo de dados

Três tabelas (nomes de coluna em snake_case no banco, mapeados via
`@map`/`@@map` do Prisma para camelCase no código):

### `tenants`
| Coluna | Tipo | Observação |
|---|---|---|
| id | uuid | PK |
| nome | text | |
| slug | text | único |
| parent_tenant_id | uuid, nullable | `NULL` só no tenant raiz (NPX) |
| dominio_base | text, nullable | |
| branding | jsonb, nullable | `{cor, logo_url, favicon_url, nome_exibicao}` — quando `NULL`/campo ausente, a UI usa o fallback padrão NPX (branding avançado com essa lógica de fallback ainda não implementado nesta fase — só o campo existe) |
| idioma | text, default `pt-BR` | Adicionado na Fase C (PT-BR em tudo). Aplicado nas ferramentas do tenant via a mesma ação de branding (`applyTenantBrandingAction` → `branding.ts`) — reaproveita a sessão/credencial admin já usada para logo/cor/tema. Ver seção "i18n" abaixo. |
| status | enum | `ativo` \| `bloqueado` |
| criado_em | timestamp | |

Auto-relacionamento `parentTenantId → Tenant.id` (hierarquia de 1 nível na
prática: NPX na raiz, tenants de clientes como filhos — nada impede
tecnicamente uma hierarquia mais profunda, mas as regras de autorização
desta fase não foram desenhadas/testadas para mais de 2 níveis).

### `users`
| Coluna | Tipo | Observação |
|---|---|---|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants |
| nome | text | |
| email | text | único (global, não só por tenant) |
| senha_hash | text | bcrypt, custo 12 |
| papel | enum | `super_admin` \| `gestor` \| `tecnico` |
| status | enum | `ativo` \| `inativo` |
| criado_em | timestamp | |
| reset_token_hash, reset_token_expires_at | nullable | plumbing do fluxo "esqueci minha senha" — não fazia parte da lista de colunas pedida originalmente, adicionado por ser infraestrutura mínima necessária para o requisito de reset de senha |

### `instances`
| Coluna | Tipo | Observação |
|---|---|---|
| id | uuid | PK |
| tenant_id | uuid | FK → tenants |
| tipo | enum | `zabbix` \| `grafana` \| `glpi` |
| url | text | |
| status | enum | `ativo` \| `pausado` |
| metadata | jsonb | hoje só guarda `{credenciais: "docs/ACCESS.md#âncora"}` — **nunca a senha em si**, só a referência de onde ela mora |
| criado_em | timestamp | |

## Provisionamento self-service de instâncias (2026-07-13)

Maior item do roadmap fechado nesta sessão: criar uma instância
(Zabbix/Grafana/GLPI) de dentro do painel, sem depender de alguém rodar
comandos manuais no host. Rota `/tenants/<id>/instances/new` (só
`super_admin`, mais conservador que a regra de "gestor no próprio
tenant" usada em usuários/branding — provisionar mexe em infraestrutura
real).

**Decisão de arquitetura: sem `docker.sock` no portal.** Em vez de montar
o socket Docker diretamente no container do portal (superfície de ataque
grande num app internet-facing), o portal fala com a **API do
Portainer** (`src/lib/portainer.ts`) — Portainer já tem `docker.sock`
(ver `portainer/docker-compose.yml`) e expõe `POST
/api/stacks/create/standalone/string` para subir uma stack a partir do
conteúdo bruto de um `docker-compose.yml`. Testado e confirmado (criação
+ atualização + remoção de stack via API, ponta a ponta) antes de
integrar na feature real.

**Dois mounts novos, aprovados explicitamente pelo responsável do
projeto** (nenhum dos dois é `docker.sock`):
- `/opt/npx-platform/clients:/host-clients` (rw) — só para o portal
  gravar o `docker-compose.yml` gerado no mesmo lugar onde os stacks
  manuais (`demo`, `flua`) já vivem, mantendo consistência para quem for
  mexer manualmente depois.
- `/opt/npx-platform/docs:/host-docs` (rw) — para `src/lib/port-registry.ts`
  ler/escrever `docs/PORT-REGISTRY.md` automaticamente quando uma
  instância Zabbix pedir porta de trapper dedicada (mesma regra
  permanente de nunca reutilizar porta, ver `CLAUDE.md`).

**Achado real que exigiu correção durante o teste ponta a ponta:** o
container do portal roda como `root` por padrão (Dockerfile não tinha
`USER`) — arquivos escritos nos mounts acima nasciam com dono `root`, e o
operador humano (`suporteti`, sem sudo sem senha) não conseguia nem
apagar esses arquivos depois. Corrigido adicionando `user: "1000:1000"`
no `docker-compose.yml` do portal. Isso por sua vez expôs um segundo
problema pré-existente (não causado por esta mudança, só descoberto por
causa dela): `.env` **não estava no `.dockerignore`**, então uma cópia
dele acabava embutida na imagem via `.next/standalone` (comportamento do
Next.js), root-only e ilegível pelo novo usuário não-root — corrigido
adicionando `.env`/`.env.*` ao `.dockerignore` do portal (também uma
melhoria de segurança por si só: segredos não deveriam nunca ficar
embutidos numa camada de imagem Docker).

**Health-check via rede Docker `edge`, nunca via domínio público —
achado central do teste ponta a ponta.** A primeira versão da feature
esperava o domínio público (`https://<tipo>.<tenant>.npxit.com.br`)
responder antes de prosseguir — e nunca funcionava para tenant novo,
porque **DNS de um subdomínio novo não existe até alguém criar
manualmente** (mesmo padrão já documentado para
`zabbix-master`/`grafana-master` na Fase 4: este projeto não controla
DNS). Corrigido: como o portal já está na mesma rede `edge` que todo
container voltado a Traefik, ele consegue checar o container **direto
pelo nome + porta interna** (`http://<slug>-grafana:3000`,
`http://<slug>-zabbix-web:8080`, `http://<slug>-glpi:80`) — confirmado
funcionando via teste direto (`fetch` de dentro do container do portal)
antes de reescrever a lógica de produção. O domínio público só passa a
responder depois que o DNS for criado manualmente (fluxo humano
separado, não bloqueia o provisionamento).

**Fluxo completo** (`src/lib/provisioning.ts`):
1. Gera o fragmento de serviço(s) (`compose-templates.ts`) — mesmas
   imagens/variáveis/labels Traefik já usadas manualmente em `demo`/`flua`,
   senhas geradas com `crypto.randomBytes`.
2. Se for Zabbix e o operador marcar "precisa de porta de trapper":
   aloca a próxima porta livre em `docs/PORT-REGISTRY.md`
   (`port-registry.ts`) — nunca decide porta sem checar o arquivo
   primeiro, mesma regra permanente de sempre.
3. Grava/mescla o `docker-compose.yml` em `clients/<slug>/` (parseia o
   existente com a lib `yaml` se o tenant já tiver outras instâncias, e
   só adiciona os serviços novos — não sobrescreve o que já existia).
4. Sobe a stack via API do Portainer.
5. Espera o container responder na rede `edge` (timeout 90s).
6. Cria o usuário `suporteti` automaticamente (ver `docs/DECISIONS.md` —
   política de conta de suporte padrão) direto na API de cada
   ferramenta, usando o admin recém-criado (Zabbix `Admin`/`zabbix`
   padrão, Grafana com a senha gerada no passo 1, GLPI `glpi`/`glpi`
   padrão).
7. Só registra a linha em `instances` **depois** de todo o resto ter
   confirmado sucesso — se qualquer etapa falhar, a tela mostra o erro
   exato (`step` + `detail`) e **nada é fingido como concluído**.

**Limite honesto, real, encontrado durante o teste — schema do MySQL do
Zabbix demora mais que o esperado:** MySQL leva ~45s só para inicializar
na primeira subida (container novo, banco vazio), e a importação do
schema completo do Zabbix (`create.sql.gz`, ~170 tabelas + dados
iniciais + constraints) some minutos inteiros depois disso — o timeout
de 90s usado na criação do `suporteti` para Zabbix **não é suficiente**
em todos os casos; validado que o processo realmente está avançando
(`SHOW PROCESSLIST` no MySQL, não travado) mas precisa de mais tempo que
o previsto. GLPI e Grafana não têm esse problema na mesma escala
(GLPI é mais rápido que Zabbix; Grafana com SQLite embutido nem tem esse
passo). Ação recomendada para uma sessão futura: aumentar o timeout
específico do Zabbix (ex: 240s) ou, melhor, trocar o polling por uma
checagem mais barata (ex: `SELECT 1 FROM dbversion` via um endpoint que
não dependa do JSON-RPC completo).

**Testado ponta a ponta nesta sessão**: tenant de teste `teste-portal`,
instância Grafana provisionada do zero pelo painel — container subiu,
respondeu na rede interna, `suporteti` criado e confirmado com login
real (`isGrafanaAdmin: true`), linha registrada em `instances`. Limpo
depois do teste (stack, volume, arquivo, linha no banco, tenant).
Zabbix testado também — pipeline completo até a subida do container
funcionou; a criação do `suporteti` expôs o achado de timeout acima
(ver `docs/STATE.md` para o resultado final consolidado).

**Não implementado nesta fase, de propósito** (conforme escopo pedido):
branding avançado na instância recém-criada, integração automática entre
serviços (Zabbix↔Grafana↔GLPI), gestão de proxies Zabbix multi-unidade.

## Módulo de integração genérico entre apps (Fase 2, 2026-07-15)

Antes desta fase, a integração entre apps de um tenant (webhook
Zabbix→GLPI, datasource Zabbix→Grafana) era só infraestrutura pontual
configurada manualmente, sem nenhuma tela de painel — confirmado
investigando o código antes de começar (não havia nenhuma rota/lib de
"health"/"reconnect" em `portal/src/`), apesar do `docs/ROADMAP.md` já
listar isso como pedido desde a sessão de 2026-07-12.

**Modelo de dados** (`Integration`, `portal/prisma/schema.prisma`):
liga duas `Instance` (`sourceInstanceId`/`targetInstanceId`) por um
`tipo` (string livre, não enum — resolvida contra o registry em código,
não o schema, para adicionar uma integração nova não exigir migração).
Campos `ativo`/`status`/`ultimaChecagemEm`/`ultimoErro` são sempre
**derivados de uma checagem ao vivo**, nunca setados manualmente — a
linha é um cache do que foi observado da última vez que a tela
carregou, não um controle que liga/desliga a integração de verdade.

**Registry extensível** (`portal/src/lib/integrations/registry.ts`):
cada integração é uma entrada com `checkHealth()` (só leitura nas
ferramentas) e `reconnect()` (a única operação que escreve). Duas
implementadas nesta fase:
- **`zabbix_to_grafana_datasource`**: `checkHealth` chama o endpoint de
  health nativo do Grafana (`/api/datasources/uid/<uid>/health`) pro
  datasource Zabbix já cadastrado; `reconnect` cria (se ausente) ou
  corrige (`PUT`, se existente) o datasource, sempre apontando pra
  `suporteti`/`SUPORTETI_PASSWORD` — **não** para o usuário dedicado
  `grafana-reader` usado na configuração manual original (simplificação
  consciente: `suporteti` já existe garantido em toda instância por
  política permanente, evita ter que provisionar/descobrir outro
  usuário só pra isso).
- **`zabbix_to_glpi_webhook`**: `checkHealth` confirma que o media type
  "GLPI (custom webhook)" e a action existem e estão habilitados
  (`status=0`) via API do Zabbix, e que o GLPI responde a
  `initSession`. `reconnect` **só reabilita** o media type/action se
  estiverem desabilitados — de propósito, **não recria do zero** se
  estiverem ausentes (exigiria replicar credencial dedicada do GLPI +
  script webhook customizado com segurança, fora do que este módulo
  decide sozinho) — nesse caso devolve `nao_configurado` com instrução
  de configurar manualmente, nunca finge sucesso.

**Autenticação dos checks**: como `suporteti` (Super admin/Admin/Super-Admin)
já existe em toda instância por política permanente (ver `CLAUDE.md`),
reaproveitado aqui em vez de provisionar mais um usuário de serviço só
pra monitoramento.

**Bloqueio por tenant** (`portal/src/lib/integrations/engine.ts`,
`INTEGRATION_WRITE_BLOCKLIST`): `reconnectIntegration` recusa (erro
`IntegrationGuardError`, nunca silencioso) rodar contra o tenant `flua`
— pedido explícito do responsável do projeto em 2026-07-15 ("não ative
nem reative NENHUMA integração na FLUA"). `syncTenantIntegrations`
(a checagem de leitura) continua rodando normalmente pra todo tenant —
só a escrita é bloqueada. Lista hardcoded de propósito (não existe hoje
conceito de "tenant travado" no schema); promover pra coluna em
`Tenant` se isso se repetir.

**Tela**: `/tenants/[id]/integrations` — visível a qualquer papel que
possa ver o tenant (`canViewTenant`), botão "Reconectar" só pra
`super_admin` (`canManageTenants`, mesmo padrão do provisionamento —
mexe em infraestrutura real).

**Achado real da validação ponta a ponta desta fase:** ao testar contra
o tenant NPX (autorizado explicitamente, ao contrário da FLUA), o
`checkHealth` do datasource Zabbix→Grafana do `demo` retornou
`com_erro`: `"Incorrect user name or password or account is temporarily
blocked"` — a credencial hoje configurada nesse datasource (do usuário
`grafana-reader` original da Fase 4) não está mais autenticando. Não
corrigido nesta sessão (nenhum `reconnect` foi chamado) — ver
`docs/STATE.md` para o registro e o que falta decidir.

**Erro cometido e corrigido durante o teste desta fase:** a validação
inicial ponta a ponta foi feita contra a FLUA (não o tenant NPX pedido),
disparando `checkHealth` real contra Zabbix/Grafana/GLPI da FLUA — sem
nenhuma escrita nas ferramentas (só leitura, mesmas chamadas já feitas
manualmente na Fase 1), mas ainda assim fora do que foi autorizado. O
responsável do projeto, avisado, optou por manter as duas linhas
gravadas (só observação real, sem risco) em vez de apagá-las. Ver
`docs/DECISIONS.md`.

## Grupos de segurança e permissões granulares (Fase 3, 2026-07-15)

Antes desta fase, autorização era só o `papel` (super_admin/gestor/tecnico),
hardcoded em `authz.ts`. Adicionado um nível opcional em cima: `SecurityGroup`
por tenant, com 3 flags (`podeCriarUsuario`, `podeCriarInstancia`,
`somenteVisualizacao`). Um usuário **sem grupo continua se comportando
exatamente como antes desta fase** — `lib/permissions.ts`
(`defaultPermissionsForPapel`) reproduz o comportamento antigo tal e qual.
Quando um grupo é atribuído, ele **substitui** (não soma) o default do
papel pra essas 3 ações específicas.

**Onde é calculado:** no login (`app/login/actions.ts`), uma vez, e
embutido no JWT (`SessionPayload.permissoes`, campo opcional — tokens
emitidos antes desta fase não têm esse campo e caem pro default via
`authz.effectivePermissions`). Mudar o grupo de um usuário só tem efeito
no próximo login dele — mesma limitação que mudar `papel` já tinha antes
(não é uma limitação nova).

**Mudança de postura de segurança digna de nota:** "criar instância" era
hardcoded só `super_admin` (`canManageTenants`), decisão consciente
registrada quando o provisionamento self-service foi construído
("provisionar mexe em infraestrutura real, mais conservador"). Agora
`canCreateInstance` permite que um `gestor` ganhe essa capacidade via
grupo — o default sem grupo continua `false` (nada muda pra quem não tem
grupo atribuído), mas a trava deixou de ser absoluta. Pedido explícito do
responsável do projeto na Fase 3 (grupos citam "quem pode criar
instância" como exemplo de permissão granular). Ver `docs/DECISIONS.md`.

**Escopo deliberadamente limitado desta fase:** só `criarUsuario` (gate
de `createUserAction`) e `criarInstancia` (gate de `provisionInstanceAction`)
usam o sistema de grupo. Editar/excluir usuário continua só em
`canManageUsersInTenant` (papel puro, inalterado); branding e reconectar
integração também continuam nos gates antigos. Se isso precisar ficar
granular também, é extensão futura, não implementada agora.

**Tela:** `/tenants/[id]/groups` — CRUD simples (lista com formulário de
edição inline por linha + formulário de criação), gate
`canManageUsersInTenant` (mesma autoridade de quem já gerencia usuários
do tenant). Excluir um grupo com usuários nele não bloqueia — os usuários
voltam ao default do papel (FK opcional).

## Cota de instâncias por tenant (Fase 3, 2026-07-15)

`TenantQuota` (tenantId, tipo, maxInstancias) — configurável só pelo
tenant mestre (NPX) via `/tenants/[id]/quotas`, gate `canManageTenants`
(super_admin), bloqueado para o próprio tenant raiz (não é cliente de
ninguém). Lógica em `lib/quotas.ts`.

**Rollout sem quebrar tenants existentes:** um tenant **sem nenhuma
linha** em `TenantQuota` fica irrestrito (mesmo comportamento de antes
desta fase — FLUA e qualquer tenant já provisionado nunca tiveram cota).
Só a partir do momento em que o tenant mestre salva a tela pela primeira
vez esse tenant entra em modo "allow-list": tipo sem linha = 0 (não
permitido). O formulário sempre salva as 3 linhas (zabbix/grafana/glpi)
de uma vez, nunca parcial.

**Aplicado em dois pontos, servidor sempre reconfere (nunca confia só na
tela):**
- `instances/new/page.tsx`: filtra/desabilita os cards de serviço cuja
  cota não permite, com o motivo exato (não cota atingida genérico vs
  tipo não permitido).
- `instances/actions.ts` (`provisionInstanceAction`): rechecha
  `canProvisionTipo` antes de criar a linha `provisionando` — se a tela
  tiver sido manipulada (ex: request direto sem passar pelo form), a
  ação recusa com mensagem clara, nunca erro genérico.

**Limite real e conhecido, não resolvido nesta fase:** o exemplo dado
pelo responsável do projeto foi "Vaultwarden: 2" (mais de uma instância
do mesmo tipo por tenant) — mas o modelo `Instance` tem
`@@unique([tenantId, tipo])`, ou seja, **hoje é fisicamente impossível
ter uma segunda instância do mesmo tipo para o mesmo tenant**, qualquer
que seja a cota configurada. `TenantQuota.maxInstancias` aceita qualquer
inteiro (schema pronto pro futuro), mas configurar 2 pra zabbix/grafana/glpi
hoje só resultaria numa segunda tentativa de provisionamento falhando na
constraint do banco (com a mensagem "já existe" já existente, não um erro
novo) antes mesmo de chegar na checagem de cota. Suportar múltiplas
instâncias do mesmo tipo de verdade exige repensar nomenclatura de
container/domínio/porta (hoje `${tipo}.${slug}.npxit.com.br`,
`${slug}-zabbix-server`, sem índice) — fora do escopo desta fase, e mais
relevante quando Vaultwarden (que nem está no `SERVICE_CATALOG`/tem
compose template ainda) for de fato implementado. Registrado em
`docs/ROADMAP.md`.

## Documentação por tenant (Fase 4, 2026-07-15)

Duas telas, deliberadamente separadas (não um toggle na mesma página) pra
tornar impossível misturar o conteúdo por engano:

- **`/tenants/[id]/docs`** — segura para o próprio cliente ver (gate
  `canViewTenant`: gestor/tecnico do próprio tenant, ou super_admin).
  Mostra, por instância ativa: nome/descrição da ferramenta (reaproveita
  `SERVICE_CATALOG`), URL pública, e detalhe operacional relevante ao
  cliente — hoje só a porta de trapper Zabbix quando existe
  (`metadata.trapperPort`), com instrução de pra onde apontar o Proxy
  Zabbix dele (`NPX_WAN_IP:porta` — IP público já conhecido, não é
  segredo). **Nunca mostra**: FortiGate, IP interno (172.16.x.x), nome de
  container, ou qualquer credencial — só orienta "peça a senha ao seu
  gestor NPX".
- **`/tenants/[id]/docs/technical`** — só super_admin (`canManageTenants`).
  Mesma lista de instâncias com mais detalhe: nomes de container
  (`containersForInstance`), se foi provisionado via portal, ponteiro pra
  `docs/ACCESS.md`/`docs/PORT-REGISTRY.md` (nunca a senha em si — mesmo
  padrão do `Instance.metadata.credenciais`, que já era só uma referência
  antes desta fase), e histórico de provisionamento.

**Decisão de design que evitou repetir o erro da Fase 2:** nenhuma das
duas páginas chama a API de nenhuma ferramenta (Zabbix/Grafana/GLPI) —
só lê o próprio banco do portal. Isso significa que abrir a documentação
de qualquer tenant (inclusive FLUA) é sempre seguro, sem o risco de
disparar uma checagem indevida contra infraestrutura de cliente (ver
`docs/DECISIONS.md`, entrada sobre o erro de escopo da Fase 2). Status
de integração entre apps continua vivendo só em `/tenants/[id]/integrations`
(que tem seu próprio guarda-corpo pra FLUA) — a página técnica só linka
pra lá, não embute a checagem inline.

**Validado ao vivo:** carregado com sucesso contra a FLUA nas duas
variantes (200, conteúdo confirmado sem nenhuma string sensível —
checado via grep por "fortigate"/IP interno/senha, nada encontrado) —
seguro porque, como descrito acima, essas páginas não tocam
infraestrutura de cliente nenhuma, só leem o banco do próprio portal.

## Ações operacionais, diagnóstico, domínio e provisionamento assíncrono (Fase A, 2026-07-15)

Lote grande de correções/features pedidas depois de uso real do painel
(não hipotético — vários itens vieram de bugs encontrados ao vivo,
inclusive um GLPI real provisionado pelo responsável do projeto durante
a sessão, fora do controle deste agente). Resumo técnico:

**A1 — ações operacionais** (`operations-actions.ts`,
`InstanceCard.tsx`): Iniciar/Parar/Reiniciar/Ver logs via novas funções
em `lib/portainer.ts` (`startContainer`/`stopContainer`/`restartContainer`/
`getContainerLogs`/`inspectContainer`), mesmo padrão de proxy Docker via
Portainer já usado no resto do projeto (sem `docker.sock` no portal).
Restrito a `super_admin` (`canManageTenants`) por pedido explícito — a
página de instâncias já é inteira restrita a isso, então a variável
`canOperate` existe separada só pra ficar óbvio onde abrir quando o
sistema de grupos (Fase 3) ganhar uma permissão dedicada pra isso
(recomendo um campo novo, não reaproveitar `podeCriarInstancia` — são
ações semanticamente diferentes).

**A2 — provisionamento assíncrono**: antes, `provisionInstanceAction`
dava `await` na chamada inteira de `provisionInstance` (até 10 minutos
pro Zabbix), travando a requisição HTTP e a tela do navegador junto.
Agora a promise é disparada sem `await` e a action redireciona
imediatamente — funciona porque o portal roda como processo Node
persistente (`next start` num container de vida longa), não
serverless/lambda, então a promise continua executando depois da
resposta HTTP já ter sido enviada. `provisionInstance` ganhou um
parâmetro `onStep` opcional, chamado a cada mudança de etapa, que
persiste `ProvisioningAudit.ultimaEtapa` em tempo real. A tela de
instâncias (`ProvisioningHistory.tsx`, client component) faz polling a
cada 3s enquanto houver alguma linha `sucesso === null`, parando sozinha
quando não há mais nada em andamento. Concorrência: cada chamada de
`provisionInstance` é uma promise independente — o Node intercala a I/O
de várias delas normalmente (mesmo mecanismo que já lida com
requisições HTTP concorrentes), então duas criações simultâneas de
tenants diferentes não se bloqueiam.

**A3 — auto-refresh de status**: `InstanceCard.tsx` faz polling a cada
20s (dentro da faixa 15-30s pedida) via `getInstanceLiveStatusAction`,
reconsultando DNS (`checkDnsReady`) e status real do container principal
(`inspectContainer`). Corrige o bug relatado: "Aguardando DNS" ficava
congelado até reload manual, mesmo com o DNS e o site já respondendo.

**A4 — emissão automática de certificado**: **nenhum código novo foi
necessário.** O Traefik já reconsulta ACME em loop pra qualquer domínio
sem certificado válido — confirmado ao vivo nos logs do próprio Traefik
(erro de DNS repetindo a cada ~10s pra `zabbix-master`/`grafana-master`,
que não têm DNS ainda). Assim que o DNS de um domínio passa a resolver,
a próxima iteração desse loop (que já roda sozinho, sempre) emite o
certificado sem nenhuma ação do portal. A2/A3 só garantem que a *tela*
reflita isso rápido, não que a emissão em si dependa do portal.

**A5 — redirect HTTP→HTTPS**: ver `docs/DECISIONS.md` (achado: era
platform-wide, não específico do GLPI — corrigido no Traefik, não no
gerador de compose).

**A6 — credenciais GLPI**: ver `docs/DECISIONS.md` e `docs/ACCESS.md`
(achado: `suporteti` sempre funcionou, só a documentação nunca foi
escrita de fato — `metadata.credenciais` agora grava um valor real
desde a criação, não mais um placeholder).

**A7 — diagnóstico de instância** (`getInstanceDiagnosticsAction`,
seção "Diagnóstico" no `InstanceCard.tsx`): status real de cada
container da instância (`inspectContainer`, incluindo `Health.Status`
quando a imagem define health check) + últimas 50 linhas do container
principal — o suficiente pra identificar causa sem SSH no servidor.
Achado ao investigar o caso concreto citado (Zabbix da NPX
"indisponível"): o único "Zabbix" que o painel rastreia pro tenant NPX
é `zabbix.demo` (registrado ali desde a fundação do portal), e está
saudável agora (`running`, `healthy`, respondendo). Se a referência era
`zabbix-master.npxit.com.br` (a stack de monitoramento própria da NPX,
em `monitoring/npx-zabbix/`) — essa **não é uma `Instance` rastreada
pelo portal** (é infra manual, fora do modelo self-service), e o motivo
dela estar inacessível por domínio público já é conhecido e documentado
há dias: falta criar o registro DNS. Não confundir os dois "Zabbix".

**A8 — domínio escolhido pelo usuário** (`ServiceAndDomainPicker.tsx`,
client component novo substituindo `ServiceCard.tsx`): antes o domínio
nem era um campo de formulário — `suggestDomain()` era usado só pra
exibição e recalculado por conta própria dentro de `provisionInstance`.
Agora é um `<input>` de verdade, pré-preenchido com a sugestão ao trocar
de serviço, mas nunca sobrescrito depois que o usuário edita manualmente
(`domainTouched`). `provisionInstance`/`provisionInstanceAction` passam
a receber o domínio como parâmetro em vez de recalculá-lo. Validação
nova em `lib/validation.ts` (`isValidDomain`/`validateDomainOrThrow`).

**A9 — trocar domínio de instância existente** (`updateInstanceDomain`
em `provisioning.ts`, ação `updateInstanceDomainAction`, form inline no
`InstanceCard.tsx`): edita só a label `rule=Host(...)` do serviço certo
dentro do compose já existente do tenant e redeploya a stack via
Portainer (`mergeCompose` já garante que só o que mudou é recriado).
Certificado novo sai sozinho — mesma automação do Traefik descrita em
A4, sem código extra.

## Credenciais de instância visíveis por tenant (2026-07-16)

Antes desta fase, credencial de instância (Zabbix/Grafana/GLPI de
cada cliente) existia só em `docs/ACCESS.md`, texto puro, sem acesso do
tenant. Nova tela `/tenants/[id]/credentials` — decisão de criptografia,
chave mestra e exclusão explícita do `suporteti` documentadas em
`docs/DECISIONS.md` (entrada 2026-07-16, com as 4 perguntas de segurança
feitas e confirmadas antes de implementar).

**Modelo:** `InstanceCredential` (1:1 com `Instance`, `username` em
texto puro — não é segredo, é convenção conhecida — `encryptedValue`
sempre cifrado) + `CredentialRevealLog` (auditoria, nunca apagado).
`lib/crypto.ts`: AES-256-GCM, `CREDENTIAL_ENCRYPTION_KEY` (`.env`, 32
bytes base64).

**Fluxo de revelar:** senha nunca chega ao HTML/estado do cliente até
o clique explícito em "Revelar" — `CredentialRow.tsx` (client) chama
`revealCredentialAction` sob demanda, que decifra e **grava o log de
auditoria antes de devolver o valor** (mesmo se algo falhar depois, o
registro de quem pediu já existe).

**Autorização:** `canViewCredentials` (novo em `lib/authz.ts`) — alias
semântico de `canManageUsersInTenant` (gestor do próprio tenant, ou
super_admin). Técnico não vê esta tela nem o link pra ela.

**Migração:** 5 credenciais nativas conhecidas com certeza (Zabbix +
Grafana de `demo` e `flua`, GLPI de `flua`) migradas via script que lê
diretamente de `docs/ACCESS.md` (nunca colou secret em argumento de
linha de comando/shell history — leu do arquivo montado no container).
`npx-glpi` ficou de fora de propósito — senha nativa nunca confirmada
(ver Fase A6, 2026-07-15), não inventada. Validado: valor bruto no
banco é ciphertext ilegível (confirmado via `psql` direto); decrypt
round-trip testado pras 5 linhas (tamanho da senha decifrada bate com o
valor real conhecido, sem imprimir o valor em si); tela testada ao vivo
contra a FLUA real (só lê o banco do próprio portal, nunca toca
Zabbix/Grafana/GLPI da FLUA — diferente do módulo de integração da Fase
2, sem risco de repetir aquele erro de escopo).

## Autorização

Toda a lógica vive em `src/lib/authz.ts`, consultada em toda rota/action:

- **`super_admin`**: `isSuperAdmin(session)` → acesso irrestrito, nenhum
  filtro de tenant é aplicado (`tenantScopeFilter` retorna `{}`).
- **`gestor`**: `canManageUsersInTenant` só retorna `true` se
  `session.tenantId === tenantId` do alvo — CRUD de usuários só no próprio
  tenant.
- **`tecnico`**: nunca passa em `canManageUsersInTenant` (só super_admin ou
  gestor do próprio tenant) — visualiza a lista de usuários do próprio
  tenant (read-only, sem botões de ação) e as próprias instâncias.
- **Isolamento entre tenants filhos**: `canViewTenant` e
  `tenantScopeFilter` sempre escopam por `session.tenantId` quando o papel
  não é `super_admin` — não existe caminho no código que deixe um
  gestor/tecnico consultar dados de um `tenantId` diferente do da própria
  sessão.

**Atualizado na Fase 6 (2026-07-13):** a limitação abaixo, descrita como
"documentado para não ser esquecido", foi de fato encontrada e corrigida
durante a validação da Fase 6 — `createUserAction`/`updateUserAction`
(`portal/src/app/tenants/[id]/users/actions.ts`) agora rebaixam
`papel=super_admin` para `gestor` no servidor sempre que o tenant alvo
não for o raiz, independente do que a UI oferece. Ainda não é uma
constraint de banco (Prisma/Postgres não impede o dado em si) — é uma
regra de aplicação, na camada de escrita. Continua sendo simplificação
consciente, não um bug pendente:

A autorização é 100% baseada no campo `papel` do usuário, não em checar
se o tenant do usuário é efetivamente a raiz (`parentTenantId IS NULL`).
Ou seja, nada no **schema do banco** impede — a nível de dado — um
usuário com papel `super_admin` dentro de um tenant filho; a convenção
"super_admin vive no tenant raiz" agora é mantida por uma checagem ativa
na camada de aplicação (server actions) e pelo seed, não por uma
constraint de banco. Documentado aqui para não ser esquecido se isso
virar um problema real mais adiante (ex: alguém escrever direto no banco
via SQL, fora do caminho normal da aplicação).

**Validado nesta sessão** (via requisições HTTP diretas, não só leitura de
código): usuário `gestor` só vê as instâncias do próprio tenant no
dashboard; tentativa de acessar a lista de usuários de outro tenant
(`/tenants/<outro-id>/users`) resulta em redirect para `/dashboard`;
tentativa de acessar `/tenants/new` (CRUD de tenant, só super_admin)
também redireciona.

## Autenticação

- Login: formulário em `/login` → Server Action `loginAction` → busca
  usuário por e-mail, `bcrypt.compare`, gera JWT (`jose`, HS256, 8h de
  validade) com `{sub, tenantId, papel, nome}`, seta cookie httpOnly
  `npx_session` (`Secure`, `SameSite=lax`).
- Middleware (`src/middleware.ts`) protege todas as rotas exceto
  `/login`, `/forgot-password`, `/reset-password/*` — verifica a
  assinatura do JWT antes de deixar passar (não faz autorização
  fina, só "está logado ou não").
- "Esqueci minha senha": gera token aleatório (32 bytes), guarda só o
  **hash SHA-256** do token no banco (`reset_token_hash`) com expiração de
  30 minutos, envia o link com o token em texto puro por e-mail (só quem
  recebeu o e-mail tem o valor que bate com o hash guardado).
- **SMTP pendente de configuração**: as variáveis `SMTP_USER` e
  `SMTP_PASSWORD` estão vazias em `/opt/npx-platform/portal/.env` —
  aguardando o usuário fornecer a senha de aplicativo do Office 365 (não
  inventada, como instruído). Até lá, o token de reset é gerado
  normalmente no banco, mas o e-mail não é enviado (`sendPasswordResetEmail`
  retorna `{sent: false}` silenciosamente — a tela sempre mostra a mesma
  mensagem genérica de sucesso, por design, para não vazar quais e-mails
  existem).

## Infraestrutura

- `docker-compose.yml`: serviços `portal` (app) e `portal-db` (Postgres).
  `portal-db` só na rede `internal` (isolada); `portal` está em
  `internal` + `edge` (precisa de `edge` para o Traefik rotear).
- Segredos (senha do Postgres, `JWT_SECRET`) ficam em
  `/opt/npx-platform/portal/.env` (chmod 600), referenciados no
  `docker-compose.yml` via `${VAR}` — não hardcoded no compose. Essa é uma
  melhoria de padrão em relação aos stacks anteriores (`demo`, `flua`), que
  hardcodavam senha direto no `docker-compose.yml`; vale considerar
  replicar esse padrão para os stacks antigos numa sessão futura (não feito
  agora — fora do escopo desta fase).
- Dockerfile multi-stage (`deps` → `builder` → `runner`), imagem final
  baseada em `node:20-slim` com output `standalone` do Next.js.
  **Detalhe que mordeu na hora do deploy**: o servidor standalone do Next,
  por padrão, escuta em `localhost` dentro do container — é preciso
  `ENV HOSTNAME=0.0.0.0` explicitamente no Dockerfile, senão o Traefik
  recebe 502 (não consegue alcançar o processo pela rede Docker).
- Migração de schema: `npx prisma db push` (não usa o sistema de
  migrations versionadas do Prisma ainda — razoável para uma fase de
  fundação com schema simples, mas vale migrar para `prisma migrate` assim
  que o schema começar a evoluir com mais frequência/múltiplos ambientes).

## O que NÃO foi implementado nesta fase (de propósito)

Ver `docs/ROADMAP.md` para a lista completa. Explicitamente fora do
escopo desta sessão: provisionamento self-service de instâncias, gestão de
proxies Zabbix, auto-integração/status de conexão entre serviços, domínio
próprio configurável por cliente, coleta de logs/métricas por instância,
branding avançado (o campo `branding` existe no schema, mas nenhuma tela
usa/aplica esse valor ainda).

## i18n (Fase C — 2026-07-13)

- `src/lib/i18n.ts`: dicionário mínimo (`SUPPORTED_LOCALES = ['pt-BR']`),
  estrutura pronta para adicionar outros idiomas depois (só adicionar o
  código à lista + um novo objeto de dicionário). Não é roteamento
  internacionalizado (sem `/en/dashboard`), é só strings + o mapeamento
  `localeToToolLanguage` (converte `"pt-BR"` do portal para o código que
  cada ferramenta espera: `pt_BR` no GLPI/Zabbix, `pt-BR` no Grafana —
  cada uma usa uma convenção diferente).
- Campo `idioma` na tabela `tenants` (default `pt-BR`), com seletor nos
  formulários de criar/editar tenant.
- **Aplicação automática real, testada, nas instâncias existentes**
  (`demo` e `flua`) nesta sessão:
  - **Zabbix**: `settings.update` com `default_lang=pt_BR` — cobre
    usuários com `lang=default` (todos, por padrão) e novos usuários.
    Confirmado visualmente (tela de login em português, "Senha").
  - **Grafana**: descoberta importante — a chave correta é `language`
    (não `locale`, que existe no payload mas não é persistido nesta
    versão/13.0.2). Setado via `PUT /api/org/preferences` no nível da
    organização (aplica a todos os usuários que não têm preferência
    pessoal própria). Confirmado numa sessão autenticada:
    `"language":"pt-BR"`.
  - **GLPI**: `php bin/console config:set --context=core language pt_BR`
    — comando oficial de CLI, aplicado diretamente no container.
    Confirmado (tela de login em português, "Senha").
- **Limite honesto:** a propagação automática via `branding.ts` (a mesma
  action que já aplica logo/cor/tema) cobre Zabbix e Grafana (ambos
  têm API testada). **GLPI não está automatizado nessa ação** — não achei
  um endpoint REST para a config global `language` do GLPI (é config
  `core`, não por Entity), só o CLI oficial funciona, e o portal não tem
  acesso de `docker exec` ao host. Aplicado manualmente nesta sessão;
  fica registrado como limitação para uma sessão futura resolver (talvez
  expondo um endpoint próprio, ou aceitando que GLPI precisa desse passo
  manual mesmo quando o resto for self-service).

## Permissões, autenticação e visual (2026-07-16)

### Modelo de permissões: recurso × nível

Substitui os 3 booleans fixos de `SecurityGroup` (removidos). Novo modelo:

- `ResourceKey`: `usuarios` | `instancias` | `operacoes_docker` |
  `credenciais`.
- `AccessLevel`: `nenhum` | `leitura` | `leitura_escrita`.
- `SecurityGroupPermission`: linha por `groupId × resource → nivel`.
- `lib/permissions.ts`: `resolveResourcePermissions()` calcula o efetivo
  (permissões explícitas do grupo, com fallback pros defaults de
  `defaultResourcePermissionsForPapel()` pra qualquer recurso sem linha
  explícita — super_admin sempre tem `leitura_escrita` em tudo,
  hardcoded).
- `lib/authz.ts`: `canViewResource(session, resource)` /
  `canWriteResource(session, resource)` são os gates usados nas telas;
  funções antigas (`canManageUsersInTenant`, `canCreateInstance`, etc.)
  viraram wrappers finos em cima dessas duas.

### Multi-tenant por usuário

- `UserTenantAccess` (`userId × tenantId`, unique): grants explícitos além
  do tenant "casa" do usuário. Só editável por super_admin, e só pra
  usuários cujo tenant "casa" é o tenant raiz (NPX) — usuários de tenant
  cliente nunca têm essa opção na UI nem no cálculo do JWT.
- `SessionPayload.accessibleTenantIds`: calculado uma vez no login
  (`lib/session-helpers.ts:issueFullSession`), embutido no JWT. Toda
  checagem de "esse usuário pode ver esse tenant" (`authz.ts:
  hasAccessToTenant`) usa essa lista do JWT, nunca consulta o banco em
  tempo real por request.
- Tenant ativo: cookie `ACTIVE_TENANT_COOKIE` (separado do JWT de sessão),
  setado por `tenant-switch/actions.ts:switchActiveTenantAction`, lido por
  `lib/auth.ts:getActiveTenantId(session)`. **Toda tela que exibe dados
  específicos de tenant precisa usar `getActiveTenantId(session)`, não
  `session.tenantId` direto** — esse foi um bug real achado e corrigido em
  `dashboard/page.tsx` nesta sessão (ver `docs/DECISIONS.md`); vale
  conferir esse mesmo padrão em telas novas no futuro.

### Autenticação: CAPTCHA, 2FA, política de senha

- `lib/turnstile.ts`: Cloudflare Turnstile, fail-open sem chaves
  configuradas (`TURNSTILE_SITE_KEY`/`TURNSTILE_SECRET_KEY` no `.env`).
- `lib/totp.ts`: TOTP RFC 6238 implementado à mão (HMAC-SHA1 nativo do
  Node + Base32), sem lib externa. Secret do usuário fica cifrado
  (`lib/crypto.ts`, AES-256-GCM) em `User.totpSecret`.
- `PlatformSettings.totpFeatureEnabled` (singleton, default `false`):
  toggle geral controlado pelo tenant mestre em Configurações →
  Segurança. Só com o toggle ligado é que usuários individuais podem
  ativar a própria 2FA (`User.totpEnabled`).
- Login em duas fases quando `totpEnabled` é true: `login/actions.ts`
  emite um cookie `PENDING_2FA_COOKIE` (JWT curto, só `userId`) em vez da
  sessão completa; `login/2fa/actions.ts` valida o código TOTP e só então
  chama `issueFullSession`.
- `User.mustChangePassword`: setado na criação de usuário quando o
  super_admin escolhe "enviar senha temporária por e-mail" (padrão);
  alternativa é definir a senha manualmente (`PasswordModeFields.tsx`
  controla qual campo aparece no formulário). `middleware.ts` força
  redirect pra `/change-password` enquanto a flag estiver true.

### SSO por tenant

- `TenantSsoConfig` (`tenantId × provider`, `ativo`, `config Json`):
  configuração de SSO fica por tenant, não é chave global da plataforma.
- `lib/sso.ts`: `applyGrafanaSso()` edita variáveis de ambiente OIDC no
  compose do tenant e chama `deployStack` (Grafana não tem API de runtime
  pra isso — precisa de redeploy). `applyZabbixSso()` chama
  `authentication.update` direto na API do Zabbix (SAML tem API de
  verdade, sem precisar de redeploy).
- GLPI fica de fora — ver `docs/ROADMAP.md` (precisaria de proxy de
  autenticação extra, não tem OIDC/SAML nativo).

### Visual: sidebar, SPA, temas

- `components/SidebarNav.tsx` substitui o antigo `NavBar.tsx`: 4 seções
  fixas (Monitoramento/Instâncias/Documentação/Acessos-Configurações),
  seletor de tenant ativo (só aparece quando `accessibleTenantIds` tem
  mais de 1 tenant), drawer mobile via hamburguer (`useState` +
  `usePathname` pra highlight do link ativo). Toda navegação interna usa
  `next/link` (SPA real, sem reload de página completa).
- `components/AppShell.tsx`: calcula tenant ativo + opções de tenant,
  renderiza `SidebarNav` + layout flex.
- Paletas: `lib/theme.ts` (`PALETTES`, cookie `PALETTE_COOKIE`), aplicadas
  via atributo `data-palette` no `<html>` + CSS custom properties
  (`--color-accent`/`-dark`/`-light`) em `globals.css`. Tailwind
  (`tailwind.config.js`) usa `rgb(var(--color-accent) / <alpha-value>)`
  em vez de hex fixo, pra permitir opacidade nas paletas. NPX continua o
  padrão de fábrica.

### Segurança de aplicação

- `next.config.js` → `headers()`: CSP, HSTS, `X-Frame-Options: DENY`,
  `X-Content-Type-Options: nosniff`, `Referrer-Policy`.
- `lib/rate-limit.ts`: janela deslizante em memória (`Map`, por processo —
  **não sobrevive a múltiplas réplicas**, ver limitação em
  `docs/DECISIONS.md`), aplicado em `/login`, `/login/2fa`,
  `/forgot-password`.
- Cookies de sessão (`npx_session`, `PENDING_2FA_COOKIE`,
  `ACTIVE_TENANT_COOKIE`): `httpOnly`, `sameSite=lax`, `secure` em
  produção.
