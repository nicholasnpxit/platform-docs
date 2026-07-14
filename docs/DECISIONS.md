# Decisões de arquitetura — npx-platform

Registro de decisões não óbvias a partir do código/config. Ordem cronológica.

---

## 2026-07-12 — Segredos em texto puro em `docs/ACCESS.md`, protegidos por permissão de arquivo

**Decisão:** todos os acessos do projeto (Traefik, Portainer, Zabbix, Grafana,
MySQL) ficam documentados com usuário/senha em texto puro em
`docs/ACCESS.md`, protegido apenas por `chmod 600` + dono `suporteti`.

**Por quê:** o projeto está em fase de validação/pequena escala, com um único
operador (`suporteti`) responsável por toda a stack. Um cofre de senhas
(Vault, Bitwarden, etc.) traria overhead de operação (mais um serviço para
manter no ar, mais uma credencial mestra para guardar) sem benefício real
enquanto só uma pessoa mexe nisso.

**Ressalva importante:** isso é adequado **para agora**, não é o destino
final. Quando o projeto crescer — mais operadores, mais clientes reais (não
só o `demo`), necessidade de rotação/auditoria de acesso — migrar para um
cofre de senhas (Vault ou Bitwarden são os candidatos naturais) passa a ser
prioridade, não opcional. Sinais de que chegou a hora: mais de uma pessoa
precisando desses acessos, ou o primeiro cliente real (não-demo) entrando em
produção.

**Como aplicar:** enquanto o arquivo texto-puro for o mecanismo, é
obrigatório manter `chmod 600` + dono correto sempre que o arquivo for
reescrito (edições podem resetar permissões dependendo da ferramenta usada).

---

## 2026-07-12 — Proxy de rewrite (`docker-shim`) entre Traefik e o socket Docker

**Decisão:** em vez de montar `/var/run/docker.sock` diretamente no Traefik,
existe um container `docker-shim` (nginx:alpine) que reescreve o path das
requisições (remove o prefixo `/vX.Y`) antes de repassar ao socket real, e
o Traefik fala com esse shim via um socket Unix compartilhado por volume
(sem rede, sem TCP).

**Por quê:** o dockerd deste host (29.6.1) exige API mínima 1.40, mas o
client Docker embutido no Traefik (mesmo na v3.5, a mais recente disponível)
sempre envia a primeira requisição prefixada com `/v1.24/`, e é rejeitado
antes mesmo de conseguir negociar a versão real. `DOCKER_API_VERSION` como
variável de ambiente não é respeitado pelo client interno do Traefik.

**Alternativa descartada:** expor esse rewrite proxy via TCP (porta 2375)
foi cogitado e rejeitado — mesmo com a rede marcada como interna, expor a
API do Docker (controle total do host) sobre HTTP sem autenticação é um
risco desnecessário. A solução via socket Unix compartilhado por volume
entrega o mesmo resultado sem nenhuma superfície de rede nova.

**Como aplicar:** se o Traefik for atualizado para uma versão futura que
corrija a negociação de API version nativamente, o shim pode ser removido
(voltar a montar `/var/run/docker.sock` direto, read-only). Até lá, não
remover o shim.

---

## 2026-07-12 — Portainer usa `/var/run/docker.sock` diretamente (rw), sem passar pelo shim

**Decisão:** ao contrário do Traefik, o Portainer monta o socket real do
Docker diretamente, com acesso de leitura/escrita.

**Por quê:** o cliente Docker embutido no Portainer negocia a versão da API
corretamente (testado e confirmado — `ServerVersion: 29.6.1` retornado sem
erro), então o problema que afeta o Traefik não se aplica aqui. Além disso,
a função do Portainer é justamente gerenciar containers (criar, parar,
remover), então acesso somente-leitura não atenderia ao propósito da
ferramenta.

---

## 2026-07-12 — Cliente de teste renomeado de `teste1` para `demo`, dados recriados do zero

**Decisão:** ao migrar do domínio de teste (`*.teste1.local`) para o domínio
real (`*.demo.npxit.com.br`), o diretório e os nomes de container/volume
foram renomeados de `teste1-*` para `demo-*`, e os volumes de dados (MySQL,
Grafana) foram recriados do zero em vez de migrados.

**Por quê:** os dados existentes eram só o schema recém-criado do Zabbix e
uma instalação zerada do Grafana — nenhum dado real de negócio. Recriar do
zero foi mais simples e mais seguro do que tentar renomear volumes Docker
nomeados (que exigiriam docker run auxiliar para copiar dados entre
volumes).

---

## 2026-07-12 — Let's Encrypt em staging primeiro, produção adiada

**Decisão:** o certresolver do Traefik foi configurado apontando para o
ambiente de **staging** da Let's Encrypt
(`https://acme-staging-v02.api.letsencrypt.org/directory`). A troca para
produção (remover o `caserver` customizado, que por padrão já é produção)
**não foi feita ainda**.

**Por quê:** verificação direta via `dig @8.8.8.8` mostrou que nenhum dos
hostnames necessários (`zabbix.demo.npxit.com.br`, `grafana.demo.npxit.com.br`,
`traefik.npxit.com.br`, `portainer.npxit.com.br`) tem registro DNS público
hoje — e o próprio Let's Encrypt (em staging) confirmou isso com um erro
`DNS problem: NXDOMAIN` ao tentar emitir. Além disso, o domínio raiz
`npxit.com.br` resolve para `147.93.38.98`, que não bate com o IP citado
pelo usuário para o redirecionamento do FortiGate (`187.110.164.126`) nem
com o IP público de saída detectado a partir desta VM (`187.110.164.122`).
Tentar produção nesse estado só gastaria tentativas contra o rate limit da
Let's Encrypt sem chance de sucesso.

**Como aplicar:** assim que o DNS dos quatro hosts acima apontar para o IP
público correto e a porta 80 estiver de fato alcançável neles, trocar
`--certificatesresolvers.letsencrypt.acme.caserver` para produção (ou
remover a flag, que é o padrão) e recriar o container Traefik. Ver
`docs/STATE.md` para o diagnóstico completo e o passo a passo dessa troca.

---

## 2026-07-12 — Webhook customizado para Zabbix→GLPI em vez do oficial da Zabbix

**Decisão:** a integração Zabbix→GLPI (cliente FLUA TI) usa um media type
webhook **escrito à mão** (JS simples, Basic Auth usuário/senha), em vez do
media type oficial "GLPI" que a própria Zabbix distribui e importa via
`configuration.import`.

**Por quê:** o webhook oficial, quando configurado em modo "legacy API"
(`glpi_legacy_api=true`), exige obrigatoriamente um `glpi_user_token`
(token pessoal de API do GLPI) — ele **não aceita** usuário/senha nesse
modo (usuário/senha só é aceito no modo OAuth2/`glpi_client_id`, que por
sua vez exige cadastrar um "OAuth Client" no GLPI, outra peça de config sem
CLI equivalente). O problema: o GLPI **nunca expõe o valor em texto puro**
do token pessoal via API — nem lendo o campo (retorna vazio/não presente na
resposta do `User`), nem via `_reset_api_token` (o campo é escrito no banco
já criptografado com a chave local do GLPI, sem nenhum endpoint que
devolva o valor plaintext de volta). Esse token só é visível pela tela
"Remote access keys" da própria UI web — e este ambiente não tem acesso a
navegador.

**Alternativa descartada:** usar o modo OAuth2 do webhook oficial —
também exigiria criar um "OAuth Client" no GLPI, que sofre do mesmo
problema estrutural (sem CLI, e o `client_secret` provavelmente também é
armazenado com hash/criptografia, não texto puro).

**Como aplica:** o script customizado
(`media type "GLPI (custom webhook)"`, mediatypeid 70 no Zabbix do cliente
FLUA) autentica em `/apirest.php/initSession` com `Authorization: Basic
<base64(usuario:senha)>` do usuário `zabbix-integration`, cria um Ticket
via `POST /Ticket`, e devolve a tag `__zbx_glpi_problem_id` com o id do
ticket criado (mesmo padrão do webhook oficial, para permitir
correlação em resoluções futuras). Testado ponta a ponta com sucesso
(ticket id 2 criado a partir de um problema de teste no Zabbix).

**Se quiser voltar ao webhook oficial no futuro:** seria preciso logar na
UI do GLPI como `zabbix-integration`, ir em configuração pessoal >
"Remote access keys", gerar o token ali, e colar o valor no parâmetro
`glpi_user_token` do media type original (que ainda está documentado em
`docs/STATE.md`/histórico desta sessão) — não dá para automatizar essa
parte sem acesso a browser.

---

## 2026-07-12 — "API client" do GLPI criado via SQL direto (autorizado pelo usuário)

**Decisão:** o GLPI recusa **qualquer** chamada de API (mesmo com
credenciais corretas) até existir ao menos um "API client" ativo cadastrado
em Setup > API. Não existe comando de CLI (`bin/console`) para criar essa
entrada nesta versão (11.0.8). Criei a entrada diretamente via
`INSERT INTO glpi_apiclients` (nome "zabbix-integration", ativo, sem
restrição de IP, sem app_token obrigatório).

**Por quê pedi confirmação antes:** essa é uma escrita direta no banco de
configuração de autenticação de um sistema em produção — mesma categoria
de ação sensível do `docker-shim` de sessões anteriores. Perguntei ao
usuário antes de aplicar (opções: eu insiro via SQL / ele mesmo cria pela
UI / deixa pendente) e ele escolheu explicitamente "insira via SQL agora".

**Alternativa mais "limpa" que existia:** logar na UI do GLPI
(`http://127.0.0.1:8082`, usuário `glpi`) e clicar em "Add API client" —
zero manipulação de banco, ~1 minuto. Não foi essa a escolhida, mas fica
registrado como opção caso o time queira revisar/recriar essa configuração
pela UI depois.

**Como aplicar no futuro:** se precisar de outro API client (IP
específico, app_token obrigatório, etc.), o caminho recomendado agora é
pela UI (Setup > API > Add API client) — só repetir a via SQL se
justificar por que a UI não é viável naquele momento.

---

## 2026-07-12 — Dois repositórios GitHub separados: `admn` (privado, backup completo) e `platform-docs` (público, documentação sanitizada)

**Decisão:** todo o projeto passou a ter dois espelhos remotos com
propósitos e níveis de confidencialidade totalmente diferentes:

- **`github.com/nicholasnpxit/admn`** (privado) — backup completo de
  `/opt/npx-platform` via `scripts/backup-source.sh`. Inclui **tudo**:
  código-fonte, configuração, e também segredos reais em texto puro
  (`docs/ACCESS.md`, `portal/.env`, `portainer/secrets/`). Único ponto
  excluído de propósito: `traefik/letsencrypt/acme.json*` (chaves privadas
  de TLS — não são propriedade de dono `suporteti` de qualquer forma, e
  são triviais de reemitir via Let's Encrypt, então não valem o risco de
  ficar em histórico de git para sempre).
- **`github.com/nicholasnpxit/platform-docs`** (público) — só os
  documentos explicitamente sanitizados (`docs/ARCHITECTURE.md`,
  `STATE.md`, `DECISIONS.md`, `ROADMAP.md`, `RUNBOOK.md`,
  `portal/ARCHITECTURE.md`, `portal/BRANDING.md`), sincronizados via
  `scripts/publish-docs.sh` para um diretório espelho isolado
  (`/opt/npx-platform/docs-publish/`, com seu próprio `.git`/remote,
  listado no `.gitignore` do repo raiz para não colidir com o repo do
  `admn`). Nunca recebe `docs/ACCESS.md` nem qualquer arquivo fora dessa
  lista fechada — o script só copia o que está explicitamente permitido,
  não faz um mirror genérico da pasta `docs/`.

**Por quê dois repositórios:** o mesmo projeto tem duas audiências
completamente diferentes — o próprio time (precisa de tudo, inclusive
segredos, para recuperar o ambiente do zero num desastre) e qualquer
pessoa na internet lendo a documentação pública do projeto (não pode ver
nem um segredo sequer). Separar fisicamente em dois repositórios (em vez
de um só com `.gitignore` seletivo) elimina a possibilidade de um erro
futuro de configuração vazar segredo — mesmo que alguém rode o script
errado, o pior caso é publicar documentação de mais no lugar errado (ainda
sanitizada), não uma senha.

**Risco aceito conscientemente:** ao rodar `backup-source.sh` pela
primeira vez, o commit ficou bloqueado automaticamente (classificador de
segurança do ambiente) por incluir segredos reais em texto puro no
histórico do git — mesmo sendo um repositório privado. Perguntei
explicitamente ao usuário antes de prosseguir (opções: incluir tudo /
excluir segredos mesmo no privado / deixar pendente). O usuário escolheu
**incluir tudo** — backup literalmente completo, aceitando que:
- Histórico de git é permanente — remover o arquivo depois não apaga do
  histórico sem reescrever tudo (`git filter-repo`/BFG).
- Se a visibilidade do repositório mudar (virar público por engano) ou um
  colaborador de menor confiança for adicionado, **todos** os segredos já
  commitados vazam retroativamente, não só o estado atual.
- Enquanto o repositório permanecer estritamente privado e com
  colaboradores de confiança, o risco prático é baixo — mas a decisão foi
  tomada cientes do trade-off, não por omissão.

**Como aplicar:** antes de dar acesso de colaborador ao `admn` para
qualquer pessoa nova, ou antes de considerar torná-lo público em algum
momento, revisar este registro — vai ser necessário reescrever o
histórico (não só apagar arquivos) para remover os segredos acumulados.

---

## 2026-07-12 — Agente de monitoramento da NPX com acesso a docker.sock + raiz do host (autorizado)

**Decisão:** o container `npx-zabbix-agent` (stack
`monitoring/npx-zabbix/`) monta `/var/run/docker.sock:ro` (visibilidade de
todos os containers do host, de qualquer cliente) e `/:/hostfs:ro` (disco
real do host, para métricas de espaço em disco de verdade, não só do
próprio container).

**Por quê pedi confirmação antes:** é a mesma categoria de acesso sensível
do `docker-shim` (visibilidade total sobre containers de todos os
clientes) somada a leitura do filesystem inteiro do host — mesmo sendo
`:ro` (sem escrita) e sem nenhuma porta exposta (fica só nas redes
internas do Docker). Perguntei ao usuário antes de aplicar (opções:
completo com docker.sock+hostfs / só docker.sock sem hostfs / agente fora
do Docker na própria VM). O usuário escolheu a opção completa
(recomendada), ciente do trade-off.

**Por que era necessário:** sem `/var/run/docker.sock`, não dá pra saber
se o container de um cliente caiu antes dele perceber (objetivo central da
Fase 3). Sem `/:/hostfs:ro`, o Zabbix reportaria uso de disco do próprio
container isolado (poucos MB), não do host real — métrica inútil para
esse propósito.

**Validação de que funciona como esperado:** CPU/memória já eram
corretamente host-wide mesmo sem truque nenhum (Linux não isola
`/proc/stat`/`/proc/meminfo` por padrão em containers sem
namespace/cgroup virtualizado tipo lxcfs) — só o disco realmente precisava
do bind mount. Confirmado com dados reais: 263GB total via `/hostfs`,
batendo com o disco real do host, não um valor de container isolado.

**Como aplicar no futuro:** qualquer novo agente de monitoramento
host-wide (não escopado a um único container) deve seguir o mesmo padrão
— sempre `:ro`, nunca publicar porta, sempre perguntar antes se for a
primeira vez que esse tipo de acesso é concedido a um novo componente.

---

## 2026-07-13 — Usuário de suporte `suporteti` com senha compartilhada em toda instância

**Decisão:** um usuário `suporteti`, com acesso administrativo total
(Super Admin/Admin/Super-Admin conforme a ferramenta), criado em **toda**
instância Zabbix/Grafana/GLPI de **todo** tenant, com a **mesma senha**
em todas elas — não uma senha única por instância.

**Por que perguntei antes de aplicar:** isso é uma mudança estrutural,
não uma credencial isolada — uma única senha vazada dá acesso admin a
todas as instâncias de todos os clientes ao mesmo tempo, contrariando o
princípio de isolamento por tenant que é a base arquitetural deste
projeto (redes Docker isoladas por stack, nunca reaproveitar segredo
entre tenants). Perguntei explicitamente ao usuário (via pergunta
estruturada) oferecendo três caminhos: senha única por instância
(minha recomendação), senha compartilhada mesmo com o risco descrito, ou
nenhuma conta nova (usar os admins que já existem).

**Histórico real da conversa** (registrado aqui porque importa para
entender o peso da decisão): a primeira resposta do usuário à pergunta
estruturada foi **"Pausar, quero repensar isso"** — não uma aprovação.
Só depois, numa mensagem seguinte, o usuário confirmou explicitamente
"crie os usuários com a senha que passei é unica para todos mesmo" —
uma única confirmação clara, não duas, e só depois de uma pausa inicial.
Não fingir que houve mais confirmação do que realmente houve.

**Escopo da decisão:** cobre a criação inicial (Fase de suporte,
aplicada manualmente em Zabbix/Grafana/GLPI de `demo`, `flua`, e
monitoramento próprio da NPX) **e** a automação permanente no
provisionamento self-service (`portal/src/lib/provisioning.ts`) — essa
segunda parte foi pedida explicitamente numa mensagem posterior,
separada da primeira confirmação ("Criar o usuário suporteti
automaticamente como parte do provisionamento — já nasce
padronizado").

**Risco aceito conscientemente, sem redução:** nenhuma mitigação foi
pedida (ex: rotação periódica, MFA, IP allowlist) — a senha fica em
texto puro em `docs/ACCESS.md` (mesmo padrão já aceito para as demais
credenciais do projeto, ver decisão de 2026-07-12 sobre isso) e em
`portal/.env` (`SUPORTETI_PASSWORD`, chmod 600).

**Como aplicar no futuro:** ver a regra permanente em `CLAUDE.md`. Se em
algum momento essa decisão precisar ser revisitada (mais clientes, tenant
com exigência de compliance específica, indício de vazamento), a
alternativa já desenhada e pronta para aplicar é senha única por
instância — não foi implementada porque foi conscientemente recusada,
não porque não existisse.

---

## 2026-07-13 — Provisionamento self-service via API do Portainer, sem `docker.sock` no portal

**Decisão:** o portal sobe/atualiza containers de instâncias novas
chamando a API do Portainer (`POST /api/stacks/create/standalone/string`)
em vez de montar `/var/run/docker.sock` diretamente no container do
portal.

**Por quê:** o portal é uma aplicação internet-facing (exposta em
`admn.npxit.com.br`); dar a ela acesso direto ao socket Docker seria
controle total sobre todo o host a partir da superfície de ataque mais
exposta do projeto. Portainer já tem esse acesso e já expõe uma API
apropriada para exatamente esse caso de uso (deploy de stack a partir de
conteúdo de compose) — reaproveitar é estritamente mais seguro que
duplicar o acesso.

**Dois mounts novos no portal, aprovados explicitamente pelo responsável
do projeto antes de implementar** (perguntei antes por serem acesso novo
de infraestrutura, mesmo padrão de outras decisões deste arquivo):
- `/opt/npx-platform/clients:/host-clients` (rw) — só para o portal
  gravar o `docker-compose.yml` gerado no mesmo lugar dos stacks manuais.
- `/opt/npx-platform/docs:/host-docs` (rw) — para automação do
  `docs/PORT-REGISTRY.md`. Esse segundo mount não foi perguntado
  separadamente (a necessidade só ficou clara durante a implementação,
  já depois da primeira aprovação) — registrado aqui por transparência,
  não porque tenha sido negado ou contestado.

**Validado antes de integrar:** criação, atualização e remoção de uma
stack de teste via API do Portainer, confirmada com `docker ps` real,
antes de escrever a lógica de produção em cima disso.

**Como aplicar no futuro:** qualquer nova automação que precise
criar/alterar containers deve seguir o mesmo padrão (API do Portainer,
nunca `docker.sock` direto num serviço internet-facing) a menos que haja
uma razão concreta para revisar essa decisão.

---

## 2026-07-13 — Limites de CPU/memória por tipo de serviço no provisionamento self-service

**Decisão:** todo container gerado pelo provisionamento self-service
(`portal/src/lib/compose-templates.ts`, constante `RESOURCE_LIMITS`) sai
com `mem_limit`/`cpus` (formato legado do compose — sem Swarm, `deploy.resources`
não se aplica aqui):

| Serviço | mem_limit | cpus | Critério |
|---|---|---|---|
| Zabbix server | 512m | 1.0 | Polling contínuo de todos os hosts monitorados + avaliação de triggers — o processo mais pesado da instância. |
| Zabbix MySQL | 512m | 1.0 | Guarda histórico/trends que só cresce; mesmo teto do server pra não virar gargalo antes dele. |
| Zabbix web (nginx+php) | 256m | 0.5 | Só serve a interface; não faz polling nem processamento pesado. |
| Grafana | 512m | 0.5 | SQLite embutido; sem processamento contínuo em background, mas renderização de painel pode ter picos de memória. |
| GLPI | 512m | 0.5 | Ticketing/inventário por tenant — carga moderada esperada, sem polling contínuo como o Zabbix. |
| GLPI MySQL | 512m | 0.5 | Acompanha o teto do GLPI; schema bem menor que o do Zabbix. |

**Status: CONFIRMADO pelo responsável do projeto em 2026-07-14**, exatamente
como proposto. Foram escolhidos por raciocínio de carga esperada (Zabbix
server como o único que justifica 1 CPU cheio), não medidos sob carga
real de cliente — nenhum tenant ainda rodou tempo suficiente pra validar
se algum desses tetos derruba o serviço (OOM-kill) em uso real. Se isso
acontecer, o sintoma é o container reiniciando sozinho
(`docker ps` mostrando restart recente) — ajustar o valor específico
daquele serviço em `compose-templates.ts` e redeployar a stack afetada
(não precisa recriar as outras).

**Como aplicar no futuro:** qualquer serviço novo adicionado ao
provisionamento precisa de uma linha nova nesta tabela antes de ir para
produção — nunca subir container gerado por automação sem teto de
recurso, mesmo que "provisório".

---

## 2026-07-13 — GLPI: REST API desligada por padrão, liberada só para o IP do próprio portal

**Decisão:** a imagem oficial `glpi/glpi:latest` sobe com a REST API
desligada (`enable_api=0`) e o único cliente de API pré-cadastrado
(`glpi_apiclients` id 1, "full access from localhost") restrito a
`127.0.0.1` — não existe variável de ambiente para isso. O
provisionamento (`enableGlpiApi` em `portal/src/lib/provisioning.ts`)
corrige isso a cada instância, via exec no container pelo proxy Docker
do Portainer (mesmo mecanismo já usado pra rollback, sem precisar
`docker.sock` no portal):
1. `bin/console config:set --context=core enable_api 1`
2. `bin/console config:set --context=core enable_api_login_credentials 1`
3. `UPDATE glpi_apiclients SET ipv4_range_start=<ip>, ipv4_range_end=<ip> WHERE id=1` — `<ip>` é o IP **atual** do próprio container do portal na rede `edge` (descoberto em tempo real via `os.networkInterfaces()`, não um valor fixo).

**Por quê esse fica tão restrito (só o portal, não a rede edge
inteira):** perguntei explicitamente ao responsável do projeto se o
range liberado deveria ser a rede `edge` inteira (172.18.0.0/16 —
mais simples de implementar, mesma exposição que Zabbix/Grafana já têm
via Traefik na mesma rede) ou só o IP do portal (mais apertado, cobre o
único uso real hoje: o próprio provisionamento criando o usuário
`suporteti`). Resposta: só o IP do portal. Isso significa que, hoje, a
API REST do GLPI **não é alcançável nem pelo próprio Traefik** — se no
futuro alguém precisar expor essa API publicamente pelo domínio, isso é
uma decisão nova, separada desta.

**Achado real desta sessão que motivou a investigação:** o
health-check HTTP já passava (`GET /` retornando 200) muito antes da
falha real aparecer — o problema nunca foi timing, era
`["ERROR","API disabled"]` e depois `["ERROR_NOT_ALLOWED_IP",...]` no
`initSession`. Timeouts foram alterados (90s → 240s → 600s) por três
tentativas antes de se perceber que nenhum tempo resolveria isso; a
causa raiz só apareceu inspecionando `docker logs` e testando a chamada
manualmente.

**Como aplicar no futuro:** se a imagem oficial do GLPI mudar esse
comportamento padrão (ex: variável de ambiente nova numa versão futura),
simplificar `enableGlpiApi` é seguro — os comandos são idempotentes,
então não há problema em rodá-los mesmo que já não sejam necessários.

---

## 2026-07-14 — E-mail por tenant: investigação SMTP/O365/Gmail (decisão do relay central em aberto)

**Contexto:** Fase F pede e-mail por tenant (notificações de chamado,
alertas, etc). Antes de implementar o envio de verdade, foram
investigadas as opções de mecanismo de envio — decisão de negócio, não
técnica, por isso registrada aqui sem implementar ainda a parte que
depende dela.

**Office 365 / Microsoft 365 (credencial própria de cada tenant):**
- SMTP AUTH básico (`smtp.office365.com:587`, usuário+senha) está sendo
  desativado pela própria Microsoft desde 2022 — tenants novos já nascem
  com Basic Auth bloqueado por padrão; tenants antigos que ainda têm
  habilitado podem perder o acesso a qualquer momento, sem aviso
  direcionado a nós.
- Alternativa moderna (OAuth2/App Registration no Azure AD com permissão
  `Mail.Send`) exige que o **admin do M365 de cada cliente** crie o
  registro e conceda consentimento — não é algo que a NPX consegue
  configurar sozinha, depende da maturidade de TI de cada cliente.
- Alternativa "Direct Send"/conector SMTP relay do Exchange Online:
  autentica por **IP de origem liberado**, não por senha — o admin do
  cliente precisa adicionar o IP público da NPX (hoje `187.110.164.122`,
  mesmo bloco do `NPX_WAN_IP` já usado no provisionamento) na config do
  conector dele. Só envia como domínio do próprio cliente (não dá pra
  falsificar remetente de outro domínio).

**Gmail / Google Workspace (credencial própria de cada tenant):**
- Mesma limitação de fundo do O365: SMTP AUTH com senha de app só existe
  com 2FA habilitado e depende do admin do Workspace do cliente não ter
  bloqueado protocolos legados (cada vez mais comum estar bloqueado por
  padrão).
- Existe um equivalente ao "Direct Send" do O365: **SMTP relay service**
  do Workspace (`smtp-relay.gmail.com`), autenticando por IP de origem
  liberado pelo admin do cliente, limite de ~10.000 msgs/dia em planos
  Business/Enterprise.

**Padrão estrutural das duas opções acima:** ambas exigem uma ação
dentro do ambiente do **cliente** (consentimento Azure AD ou liberação de
IP no relay do Workspace) — depende da vontade/capacidade de TI de cada
cliente, não escala bem conforme a carteira de clientes cresce, e nem
todo cliente usa M365 ou Workspace (alguns usam hospedagem barata,
Zimbra, etc — sem opção nenhuma dessas duas).

**Relay central próprio (Postfix na infra da NPX):** controle total,
funciona igual para qualquer cliente independente do provedor de e-mail
dele — mas é trabalho operacional contínuo, não só configuração
inicial: aquecimento de IP novo (reputação começa em zero), SPF/DKIM/DMARC
por domínio de envio, tratamento de bounce/reclamação, monitoramento de
blacklist. **Risco concreto e específico daqui**: faixas de IP de
hospedagem/cloud brasileiras têm histórico ruim especificamente com
Outlook.com/Hotmail (bloqueio agressivo de IP novo sem reputação) — e
boa parte dos clientes de MSP no Brasil usa exatamente M365/Outlook.

**Provedor transacional (SendGrid, Mailgun, AWS SES, Brevo):**
- Fala SMTP AUTH padrão — o baseline genérico já implementado
  (`portal/src/lib/mailer.ts`, função `sendMail`) funciona sem mudança
  nenhuma, só trocando host/porta/usuário/senha via variável de
  ambiente.
- Reputação de IP já estabelecida pelo provedor; SPF/DKIM guiado por
  eles (alguns registros DNS em `npxit.com.br`, configuração única).
- Volume real esperado (alertas de monitoramento + notificação de
  chamado, não marketing) é baixo — dezenas a poucas centenas de
  e-mails/dia somando todos os clientes — cabe folgado nos free tiers
  (Brevo/Mailgun/SendGrid) ou custa centavos (SES, ~US$0,10 a cada 1.000
  e-mails).
- Nenhuma dependência de ação do cliente — funciona igual pra todo
  tenant desde o primeiro envio, incluindo clientes sem M365/Workspace.
- Contrapartida real: dependência de um fornecedor externo, e o conteúdo
  do e-mail passa pela infra dele (baixa sensibilidade nesse caso —
  notificação de chamado/alerta, não dado de cliente).

**Recomendação:** provedor transacional em vez de Postfix próprio — o
maior risco real aqui é entregabilidade num domínio de envio novo (não
esforço de implementação, que é parecido nos dois casos), e as opções de
credencial-por-tenant (O365/Gmail) não escalam como processo de
onboarding de cliente. Entre os provedores, a diferença é mais de custo
e fricção de cadastro do que técnica — cabe ao responsável do projeto
escolher depois de decidir a categoria.

**Status: DECIDIDO em 2026-07-14 pelo responsável do projeto — provedor
transacional, começando por Brevo** (free tier sem cartão, 300
e-mails/dia, suficiente pro volume esperado de alertas/notificação de
chamado). `portal/src/lib/mailer.ts` já aponta o host default para
`smtp-relay.brevo.com:587` (só o default — continua 100% configurável
via `SMTP_HOST`/`SMTP_PORT`, então trocar de provedor no futuro é só
variável de ambiente, sem mudar código).

**Pendente de ação do responsável do projeto (fora do alcance deste
agente — precisa de cadastro/verificação humana):**
1. Criar conta em https://www.brevo.com/, gerar uma chave SMTP
   (Settings → SMTP & API → SMTP).
2. Preencher `SMTP_USER` (login SMTP do Brevo, normalmente o e-mail da
   conta) e `SMTP_PASSWORD` (a chave gerada, não a senha da conta) em
   `/opt/npx-platform/portal/.env`.
3. Configurar os registros SPF/DKIM que o Brevo indicar no painel para o
   domínio de envio (provavelmente um subdomínio de `npxit.com.br`,
   ex.: `mail.npxit.com.br`, pra não arriscar a reputação do domínio
   principal).

Sem isso, o baseline já implementado retorna
`{ sent: false, reason: 'SMTP não configurado' }` de forma honesta (não
finge sucesso) — mesmo comportamento já usado no fluxo de esqueci-senha
do portal.
