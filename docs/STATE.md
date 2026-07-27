# Estado atual — npx-platform

Última atualização: 2026-07-15 — sessão de realinhamento pós-reboot da VM
+ **Fase 1 (FortiGate SSH, só leitura) concluída, aguardando revisão do
responsável antes da Fase 5 (automação de escrita)** + **Fase 2 (módulo
de integração genérico) implementada e validada** + **Fase 3 (grupos de
segurança + cota por tenant) implementada e validada, incluindo item 9
(reset de senha via Brevo) testado de ponta a ponta com sucesso real**
+ **Fase 4 (documentação por tenant, cliente + técnica) implementada e
validada**. Ver seções "FortiGate — primeiro acesso live (2026-07-15)",
"Módulo de integração genérico (Fase 2, 2026-07-15)", "Grupos de
segurança e cota por tenant (Fase 3, 2026-07-15)" e "Documentação por
tenant (Fase 4, 2026-07-15)" abaixo.

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
