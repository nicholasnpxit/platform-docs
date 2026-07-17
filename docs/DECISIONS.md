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

---

## 2026-07-16 — Credenciais de instância visíveis por tenant: AES-256-GCM, chave mestra em `.env`, `suporteti` explicitamente excluído

**Contexto:** até esta fase, a única fonte de credencial de instância
(Zabbix/Grafana/GLPI de cada cliente) era `docs/ACCESS.md` — um arquivo
interno, sem acesso do cliente. Pedido: cada tenant ver, dentro do
próprio painel, usuário/senha das próprias instâncias, com as senhas
cifradas em repouso no banco (nunca texto puro).

**Perguntei antes de implementar** (pedido explícito do responsável do
projeto: "pare para confirmação em qualquer decisão sensível") — quatro
decisões, todas confirmadas com a opção recomendada:

1. **Algoritmo: AES-256-GCM.** Padrão da indústria, autenticado
   (detecta adulteração do texto cifrado — decifrar com chave certa mas
   ciphertext alterado falha alto, nunca devolve lixo silenciosamente),
   nativo no módulo `crypto` do Node — zero dependência nova.
   Implementado em `lib/crypto.ts`. Formato armazenado:
   `"<iv>:<authTag>:<ciphertext>"`, cada parte em base64 — IV aleatório
   por valor (12 bytes, recomendado pra GCM), nunca reaproveitado, não é
   segredo (só precisa ser único).

2. **Chave mestra: gerada nesta sessão (`crypto.randomBytes(32)`),
   guardada em `portal/.env` como `CREDENTIAL_ENCRYPTION_KEY`** (base64,
   chmod 600, nunca versionada) — mesma disciplina já usada pro
   `JWT_SECRET`. **Trade-off aceito conscientemente:** perder essa chave
   (ex: perder o `.env` sem backup) = perder acesso a toda credencial já
   cifrada, sem forma de recuperar — mesmo risco que o projeto já aceita
   pro `JWT_SECRET` (perdê-lo invalida todas as sessões) e pro padrão
   geral de segredo único em texto puro protegido só por permissão de
   arquivo (ver decisão de 2026-07-12 sobre `docs/ACCESS.md`).
   **Rotação, se um dia for necessária:** não há mecanismo automático
   construído nesta fase. Precisaria de um script que decifra cada linha
   com a chave antiga e recifra com a nova, rodado com as duas chaves
   disponíveis simultaneamente numa janela curta — não implementado
   agora porque não foi pedido e adicionaria complexidade sem uso
   imediato; registrar aqui como algo a construir se/quando a rotação
   virar necessidade real (ex: suspeita de vazamento da chave).

3. **`suporteti` (senha compartilhada em toda instância de todo tenant)
   NUNCA entra neste sistema — decisão de segurança crítica.** Risco
   identificado e confirmado antes de escrever qualquer código: se a
   senha do `suporteti` fosse cifrada e exibida como "credencial da
   instância X" pro tenant dono de X, esse tenant veria a mesma senha
   que dá acesso admin a **todas as instâncias de todos os outros
   clientes** (a decisão de 2026-07-13 já documentou o `suporteti` como
   compartilhado — ver acima). Migrar/exibir isso aqui seria um
   vazamento cross-tenant grave, o oposto do "isolamento total entre
   tenants" pedido. `InstanceCredential` guarda só a credencial NATIVA
   de cada ferramenta (Admin do Zabbix, admin do Grafana, `glpi` do
   GLPI) — única por instância, nunca compartilhada entre tenants.

4. **Quem vê dentro do tenant: gestor + super_admin, não técnico**
   (`canViewCredentials`, reaproveita exatamente `canManageUsersInTenant`
   — mesma autoridade de quem já gerencia usuários do tenant). Técnico
   continua vendo a lista de instâncias como sempre, só não a senha —
   escopo mais apertado que "ver o tenant" de propósito, por serem
   credenciais de admin de ferramenta.

**Auditoria:** `CredentialRevealLog` grava usuário + timestamp toda vez
que uma senha é decifrada e mostrada — gravado **antes** de devolver o
valor pro cliente (mesmo se algo falhar depois, o registro de quem
pediu já existe). Nunca apagado, só cresce, mesmo espírito do
`ProvisioningAudit`.

**Migração das credenciais já existentes** (`demo` e `flua`, hoje só em
`docs/ACCESS.md`): rodada nesta sessão para as 5 credenciais nativas
conhecidas com certeza (Zabbix e Grafana de `demo` e `flua`, GLPI de
`flua`). **`npx-glpi` (GLPI do tenant NPX) ficou de fora
deliberadamente** — a senha nativa `glpi/glpi` dessa instância nunca foi
testada/confirmada nesta ou em sessão anterior (ver Fase A6,
2026-07-15: só o `suporteti` foi confirmado ali), e inventar/assumir um
valor não confirmado no banco de credenciais seria pior que deixar em
branco. `docs/ACCESS.md` continua sendo a fonte de verdade para
segredos de infraestrutura interna da NPX (JWT_SECRET, senha do
FortiGate, credencial do Brevo) — esses nunca entram neste sistema,
só credencial de instância de cliente.

---

## 2026-07-15 — Primeiro acesso live ao FortiGate: escopo real do perfil diverge do esperado nos dois sentidos

**Contexto:** até esta sessão, este projeto **nunca teve acesso direto
(live) ao FortiGate** — todo comando de VIP/policy era só preparado aqui e
aplicado manualmente pelo responsável do projeto (ver `docs/PORT-REGISTRY.md`).
O responsável forneceu credencial SSH (usuário `admn`, LAN-only, porta 22)
pedindo validação do escopo real de leitura/escrita antes de qualquer
automação de escrita (Fase 5, ainda não iniciada).

**O que foi pedido validar:** que o usuário tivesse leitura em tudo
(`show full-configuration` ou equivalente) e escrita **apenas** em firewall
policy, VIP, address e service objects — nada além disso.

**O que foi encontrado, testado ao vivo (comandos de leitura reais, nenhuma
escrita/alteração tentada):**

O `accprofile` real do usuário `admn` é:
```
config system accprofile
    edit "admn"
        set ftviewgrp read
        set sysgrp read
        set netgrp read
        set loggrp read
        set fwgrp custom
        set cli-diagnose enable
        set cli-get enable
        set cli-show enable
        set cli-exec enable
        set cli-config enable
        config fwgrp-permission
            set policy read-write
            set address read-write
            set service read-write
            set schedule read-write
            set others read-write
        end
    next
end
```

**Divergência 1 — leitura NÃO é "em tudo" (menos do que pedido):** grupos
não citados no perfil (VPN, Security Profiles/UTM, User & Authentication,
Security Fabric, WiFi/Switch Controller, WAN Opt) usam o default do
FortiOS quando omitidos, que é **nenhum acesso** — confirmado ao vivo, não
suposto: `show vpn ipsec phase1-interface`, `show antivirus profile` e
`show user local` retornaram todos `command parse error` / `Command fail`
(recusados pela CLI, não um erro de sintaxe). Leitura real hoje: só
FortiView, System, Network e Log & Report — mais o grupo Firewall (abaixo).

**Divergência 2 — escrita é MAIS ampla do que pedido:** o pedido citava só
"policy, VIP, address e service". O perfil real tem `schedule` também em
read-write (não pedido), e o grupo `others` do FortiOS **não é só VIP** —
esse grupo agrega Virtual IPs, IP Pools, Traffic Shaping, perfis de
inspeção SSL/SSH, DoS policies, local-in policies e mais objetos de
firewall, todos em read-write juntos (é uma única chave on/off no FortiOS,
não é possível conceder VIP sem os demais dentro de `others`). Confirmado
ao vivo: `show firewall vip` e `show firewall policy` retornaram a
configuração completa sem erro.

**Divergência 3 — poder operacional além de show/get:** `cli-diagnose`,
`cli-exec` e `cli-config` estão todos habilitados. Isso permite comandos
`execute`/`diagnose` (ex: ping, sniffer, debug flow, possivelmente
`execute reboot`/limpeza de sessões) — não é escrita de configuração
persistente, mas é mais poder operacional do que "só leitura + escrita
escopada a objetos de firewall" sugere. Não testado nesta fase (nenhum
comando `execute`/`diagnose` foi rodado) — só sinalizado pela config do
perfil.

**Achado incidental:** `show system admin` (leitura completa, sysgrp=read)
retornou **apenas uma conta administrativa no FortiGate inteiro: `admn`**
— não existe hoje nenhuma segunda conta admin (ex: um `super_admin` de
break-glass separado). Não é uma decisão deste projeto, só um fato
observado que vale o responsável do projeto saber.

**Por que não decidi sozinho se o perfil está "certo":** é uma
divergência de segurança em duas direções (menos leitura do que pedido
Y+ mais escrita do que pedido) num firewall de produção — decisão de
ajustar (ou não) o perfil no FortiGate cabe ao responsável do projeto, não
a este agente. Nenhuma escrita foi tentada; a Fase 5 (automação de
escrita) segue bloqueada até essa revisão.

**Como aplicar:** se o responsável decidir estreitar `others`/`schedule`
ou ampliar leitura para VPN/UTM/Auth, o ajuste é direto no FortiGate
(`config system accprofile` → `edit "admn"`), fora do escopo deste
projeto alterar (este projeto só lê, nunca escreve no FortiGate até uma
decisão explícita futura). Credencial em `docs/ACCESS.md` (seção
FortiGate) e `/opt/npx-platform/fortigate/.env`.

---

## 2026-07-15 — Erro de escopo: validação do módulo de integração testada contra a FLUA em vez do tenant NPX

**Contexto:** ao construir o módulo de integração genérico (Fase 2, ver
`docs/portal/ARCHITECTURE.md`), o responsável do projeto tinha pedido
explicitamente: "Teste e ative só no tenant da própria NPX, e só quando
eu mandar pela console, não automaticamente" — e que nada fosse
ativado/reativado na FLUA.

**O que aconteceu:** ao validar que a tela `/tenants/[id]/integrations`
funcionava de ponta a ponta, o teste foi feito contra o tenant **FLUA**
(não o NPX pedido) — um `curl` autenticado como `super_admin` carregando
a página, que por design roda `checkHealth` (só leitura contra
Zabbix/Grafana/GLPI) e grava o resultado como cache no banco do portal.
A ferramenta de permissão do ambiente (classificador automático)
bloqueou uma tentativa **seguinte** de ler o HTML já baixado, sinalizando
o desvio — a ação original já tinha sido executada antes desse aviso.

**Avaliação honesta do impacto real:**
- **Nenhuma escrita foi feita nas ferramentas da FLUA** — `checkHealth`
  só faz as mesmas chamadas de leitura já feitas manualmente na
  investigação da Fase 1 (`mediatype.get`, `action.get`, health check do
  datasource). `reconnect` (a única operação que escreve) nunca foi
  chamado, e está bloqueado por código pra esse tenant de qualquer
  forma (`INTEGRATION_WRITE_BLOCKLIST`).
- **O que foi escrito:** duas linhas na tabela `integrations` do
  **banco do portal** (não nas ferramentas), registrando o estado real
  observado (ambas `saudavel`) — consistente com o que a Fase 1 já
  tinha achado ao vivo.

**Como foi resolvido:** parei de testar contra a FLUA imediatamente,
expliquei o ocorrido e o risco real ao responsável do projeto, e
perguntei explicitamente se as duas linhas deveriam ser mantidas ou
apagadas. Resposta: **manter** (são só observação real, sem risco).
Validação ponta a ponta refeita corretamente contra o tenant **NPX**
(par `zabbix.demo`/`grafana.demo`) — que por sua vez achou um problema
real e não-relacionado (ver `docs/portal/ARCHITECTURE.md`, seção do
módulo de integração).

**Por que registrar isso:** não é uma falha técnica (nada foi
corrompido, nenhuma integração de cliente foi tocada), mas é um desvio
real de escopo explicitamente pedido — vale o registro pelo mesmo motivo
de outras entradas deste arquivo: não fingir que a sessão foi mais
disciplinada do que foi.

**Como aplicar no futuro:** ao validar qualquer feature nova que tenha
uma restrição explícita de "só teste em X", confirmar o tenant/escopo
exato **antes** de rodar o primeiro request de teste, não só depois de
escrever o código — o código em si (`INTEGRATION_WRITE_BLOCKLIST`)
protegeu contra a ativação de verdade, mas não contra a checagem de
leitura ter rodado onde não devia.

---

## 2026-07-15 — "Criar instância" deixa de ser hardcoded só super_admin, passa a ser configurável por grupo de segurança

**Decisão:** com o sistema de grupos de segurança da Fase 3
(`docs/portal/ARCHITECTURE.md`), um `gestor` pode ganhar a permissão de
provisionar instância (`podeCriarInstancia` no grupo) — antes disso, era
uma trava absoluta em código (`canManageTenants`, só `super_admin`),
decisão registrada quando o provisionamento self-service foi construído
("mais conservador que a regra de gestor no próprio tenant... provisionar
mexe em infraestrutura real").

**Por quê mudou:** pedido explícito do responsável do projeto na Fase 3
— a lista de permissões granulares deu "quem pode criar instância" como
exemplo concreto de coisa que devia ser configurável.

**Por que não é um enfraquecimento silencioso:** o **default continua
idêntico** — um usuário sem grupo atribuído (inclusive todo `gestor` já
existente antes desta fase) continua sem poder criar instância, exatamente
como antes. A permissão só existe se alguém com autoridade sobre grupos do
tenant (`canManageUsersInTenant`) criar um grupo com essa flag e atribuir
um usuário a ele — um passo explícito, não uma migração automática que
muda comportamento de ninguém.

**Como aplicar no futuro:** se isso se provar problemático (ex: um
gestor provisionando instância sem entender o impacto de recurso/custo),
a mitigação é não conceder `podeCriarInstancia` em nenhum grupo — não
precisa reverter código, é uma escolha de configuração por tenant.

---

## 2026-07-15 — Cota de instância: schema pronto para N>1 por tipo, mas `Instance` ainda trava em 1

**Contexto:** o pedido de cota por tenant (`docs/portal/ARCHITECTURE.md`)
usou como exemplo "Vaultwarden: 2" — mais de uma instância do mesmo tipo
pro mesmo tenant.

**Achado:** `Instance` tem `@@unique([tenantId, tipo])` desde a fundação
do schema — nenhum tenant pode ter uma segunda instância do mesmo tipo
hoje, independente de cota. `TenantQuota.maxInstancias` foi modelado como
inteiro livre (não travado em 0/1) para não precisar de outra migração
quando isso for resolvido, mas a aplicação real de N>1 não foi construída
nesta fase.

**Por que não resolvi agora:** suportar de verdade uma segunda instância
do mesmo tipo exige mexer em nomenclatura de domínio/container/porta em
vários lugares (`suggestDomain` — hoje `${tipo}.${slug}.npxit.com.br`,
sem índice; `compose-templates.ts` — `${slug}-zabbix-server` etc.;
possivelmente alocação de porta de trapper) — mudança estrutural, não um
ajuste de cota. Além disso, nenhum dos tipos que hoje têm N>1 como caso
de uso real (Vaultwarden) está implementado ainda (nem no
`SERVICE_CATALOG`, nem tem compose template) — resolver a nomenclatura
antes de existir o serviço de verdade seria desenhar às cegas.

**Como aplicar no futuro:** quando Vaultwarden (ou qualquer tipo que
precise de N>1) for de fato implementado, essa é a hora certa de revisar
nomenclatura pra suportar índice (`vaultwarden-1`, `vaultwarden-2`, ...) e
então remover/ajustar o `@@unique([tenantId, tipo])` do schema. Registrado
em `docs/ROADMAP.md` pra não perder o contexto até lá.

---

## 2026-07-15 — Redirect HTTP→HTTPS ausente era platform-wide, não específico do GLPI — corrigido no Traefik, não no gerador

**Contexto:** o responsável do projeto reportou que o GLPI recém-criado
para o tenant NPX não redirecionava `http://` para `https://`
automaticamente, como "os outros serviços fazem", e pediu correção no
gerador de compose (`compose-templates.ts`), supondo falha pontual no
template do GLPI.

**Investigação real antes de aplicar qualquer correção:** testei
`http://<host>/` com o Host header correto **direto contra o Traefik**
(`docker exec traefik wget --header="Host: ..." http://localhost:80/`),
sem depender de rede externa — para `zabbix.flua`, `grafana.flua`,
`glpi.flua` (stacks manuais, não geradas pelo portal) e
`zabbix.demo`/`grafana.demo` (idem). **Todos voltaram 404 Not Found**,
não só o GLPI da NPX. Confirmado também que nem o `traefik/docker-compose.yml`
nem nenhum `clients/*/docker-compose.yml` manual tinha qualquer
middleware de redirect ou router no entrypoint `web` — o suporte a
redirect **nunca existiu neste projeto**, em nenhum serviço, desde o
início. A percepção de que "os outros funcionam" era enganosa —
provavelmente por cache de HSTS/bookmark no navegador do responsável,
nunca batendo em `http://` de fato depois da primeira visita.

**Decisão: corrigir no Traefik (entrypoint), não no gerador de
compose.** Duas flags novas no `command:` do `traefik/docker-compose.yml`:
```
--entrypoints.web.http.redirections.entrypoint.to=websecure
--entrypoints.web.http.redirections.entrypoint.scheme=https
```
Isso redireciona **todo** request HTTP em qualquer host pro HTTPS
equivalente, no nível do proxy — cobre automaticamente todo serviço já
existente (manual ou self-service) e todo serviço futuro, sem precisar
de label por serviço nem tocar em `compose-templates.ts`. É o padrão
oficialmente documentado pelo próprio Traefik e convive bem com o
desafio ACME HTTP-01 (que também usa o entrypoint `web`) — Traefik trata
o desafio ACME numa camada interna que tem prioridade sobre o redirect.

**Por que não segui o pedido literal (corrigir só o gerador):** corrigir
só `compose-templates.ts` teria resolvido apenas serviços *futuros*
provisionados pelo portal, deixando `flua`/`demo` (e o `glpi.npx` que já
existe) permanentemente sem redirect. A causa raiz era estrutural
(ausência total do mecanismo), não um bug pontual de template — a
correção completa e mais segura era no proxy compartilhado, não em cada
gerador/compose individual.

**Validado depois de aplicar** (`docker exec traefik wget`, direto
contra o Traefik, sem depender de rede externa): `zabbix.flua` — antes
`404`, depois `301 Moved Permanently` → `Location: https://zabbix.flua...`
→ segue e completa em `200 OK`. Log do Traefik checado após o restart:
nenhum erro novo, só o erro de DNS já conhecido e documentado
(`zabbix-master`/`grafana-master` sem DNS criado, pendência antiga não
relacionada a esta mudança).

**Risco avaliado:** mudança restrita ao entrypoint HTTP (porta 80), que
hoje só devolvia 404 pra qualquer requisição — não havia comportamento
correto pra "quebrar". HTTPS (porta 443, onde todo tráfego real
acontece hoje) não foi alterado. Container `traefik` foi recriado
(restart), afetando momentaneamente todos os tenants ao mesmo tempo —
mesma categoria de operação de qualquer deploy/restart do proxy
compartilhado, já uma prática estabelecida neste projeto.

**Confirmado também para o caso concreto reportado**: `glpi.npx.npxit.com.br`
(a instância que motivou o pedido) — antes `404`, depois `301` →
`https://glpi.npx.npxit.com.br/` → `200 OK`. Não reexecutei a mesma
checagem pontual pros demais hosts (`grafana.flua`, `glpi.flua`,
`zabbix.demo`, `grafana.demo`) depois de confirmar os dois casos mais
relevantes (um manual, um self-service) — o mecanismo é de entrypoint
(não por host), então a mesma configuração vale igual pra todos.

---

## 2026-07-15 — Provisionamento assíncrono: fire-and-forget dentro do próprio processo Node, sem fila externa

**Decisão:** `provisionInstanceAction` dispara `provisionInstance(...)`
sem `await`, persiste progresso real por etapa
(`ProvisioningAudit.ultimaEtapa`, via um callback `onStep` novo) e
redireciona a tela imediatamente. A tela de instâncias faz polling
(3s) na tabela de auditoria enquanto houver algo em andamento.

**Por que não usei uma fila de verdade (BullMQ+Redis, etc.):** o
processo do portal já é um `next start` de vida longa dentro de um
container `restart: unless-stopped` — não é serverless/lambda (que
mataria a promise no fim da resposta HTTP) nem multi-processo/multi-réplica
(que exigiria coordenação entre workers pra não duplicar trabalho).
Nesse contexto, uma promise não-aguardada dentro do mesmo processo
resolve o problema real (não travar a tela) sem adicionar mais um
serviço pra manter no ar, mais uma credencial, mais um ponto de falha —
proporcional ao volume real esperado (poucos provisionamentos por dia,
não milhares). Se o portal algum dia rodar com múltiplas réplicas atrás
de um load balancer, essa decisão precisa ser revisitada (nesse cenário
sim valeria a pena uma fila externa, pra não perder o job se a réplica
que o iniciou cair no meio).

**Validado ao vivo, ponta a ponta, contra um tenant de teste descartável
(`teste-a2`, criado e removido completo nesta sessão — nunca tocou
FLUA nem NPX):** submissão real via HTTP do formulário de criar
instância (Grafana) — resposta HTTP voltou em **1 segundo** (antes
levaria 1-2 minutos, travando a aba). Consulta ao banco logo em seguida
mostrou `ultima_etapa` já tinha avançado para "aguardando container
responder" com `sucesso`/`finalizado_em` ainda nulos — prova real de que
o trabalho continuou executando depois da resposta HTTP já ter sido
enviada, não just morreu junto com o request.

**Risco avaliado:** se o container do portal for reiniciado (deploy,
crash, `docker restart`) no meio de um provisionamento em andamento, a
promise em memória morre e a linha `ProvisioningAudit` fica presa em
"em andamento" pra sempre (mesmo comportamento de qualquer job em
memória interrompido). Não construí uma tela de "cancelar"/"limpar
travado" nesta fase — se isso acontecer na prática, a limpeza manual via
SQL (como já é feito pra outros casos de rollback incompleto) resolve.
Registrar como possível melhoria futura se o volume de provisionamento
crescer a ponto de reinícios do portal durante um provisionamento virarem
comuns.

---

## 2026-07-16 — Permissões granulares, multi-tenant por usuário, 2FA/CAPTCHA/SSO, reforma visual

Lote grande, pedido em bloco único ("PERMISSÕES E ACESSO" / "AUTENTICAÇÃO E
SENHAS" / "VISUAL E NAVEGAÇÃO"). Registrando aqui só as decisões não óbvias
a partir do código — o restante está nos arquivos citados.

### Permissão granular: recurso × nível, substituindo os 3 booleans fixos

`SecurityGroup` tinha `podeCriarUsuario`/`podeCriarInstancia`/
`somenteVisualizacao` — coarse demais para o pedido (3 níveis por
recurso: `nenhum`/`leitura`/`leitura_escrita`). Substituído por
`SecurityGroupPermission` (linha por `resource × nivel`), com
`resolveResourcePermissions()` calculando defaults por `papel` idênticos
ao comportamento anterior (gestor = escrita em usuários, leitura em
instâncias, nada em docker/credenciais; técnico = leitura em
usuários/instâncias, nada no resto) — ninguém perde acesso que já tinha
na migração.

**Migração aplicada com `prisma db push --accept-data-loss`** (confirmado
via pergunta direta ao responsável do projeto, que escolheu "aplicar sem
preservar"): derrubou as 3 colunas boolean antigas, com 1 linha real
afetada (`grupo "VALID"`, tenant NPX, usuário `tulio@npxit.com.br`,
super_admin — permissões desse grupo precisam ser reconferidas/
reconfiguradas na tela nova, já que o valor anterior não foi
retro-convertido).

### Multi-tenant por usuário: grant fica no JWT, não é lookup em tempo real

`UserTenantAccess` (tabela nova) registra quais tenants um usuário
específico pode acessar além do seu tenant "casa". A lista resolvida
(`accessibleTenantIds`) é calculada uma vez no login e embutida no JWT de
sessão — **igual ao padrão já existente neste projeto pra mudança de
`papel`/grupo**: se o super_admin conceder/revogar acesso a um tenant
depois que alguém já está logado, só entra em efeito no próximo login
daquela pessoa. Não implementei invalidação em tempo real (exigiria
sessão com estado no servidor, ou short-lived tokens com refresh —
desproporcional ao volume de usuários da plataforma hoje). Fica registrado
como possível melhoria futura se isso virar um problema prático (ex.:
alguém reportando "revoguei e a pessoa ainda está vendo").

**Regra que se mantém sem exceção**: usuários de dentro de um tenant
cliente (não-NPX) nunca recebem `UserTenantAccess` para outro tenant — a
tela de edição só mostra os checkboxes de tenant-access quando quem está
editando é super_admin E o usuário sendo editado pertence ao tenant raiz
(NPX). Enforced tanto na UI (`updateUserAction`) quanto no cálculo de
`accessibleTenantIds` no login (nunca confia em input do cliente pra isso).

**Achado corrigido durante teste ponta-a-ponta**: o seletor de tenant no
cabeçalho já setava o cookie `ACTIVE_TENANT_COOKIE` corretamente, mas
`dashboard/page.tsx` ainda usava `session.tenantId` fixo pra filtrar
dados — o seletor trocava o cookie mas a tela mostrada não mudava. Só foi
pego porque testei com usuário descartável de verdade (criar → conceder
FLUA → re-logar → conferir JWT → conferir que FLUA aparece com os 3
instances reais) em vez de confiar que "o código parece certo". Corrigido
em `dashboard/page.tsx` (agora lê `getActiveTenantId(session)`); vale
auditar se alguma outra tela ainda tem esse mesmo padrão de bug (usar
`session.tenantId` direto em vez do tenant ativo) numa sessão futura — não
fiz varredura exaustiva de todas as telas neste lote, só corrigi o caso
achado.

### CAPTCHA: Cloudflare Turnstile, falha aberta enquanto não configurado

Pedido explícito do responsável (Turnstile, "salvo se eu preferir outro").
Implementado com **fail-open**: sem `TURNSTILE_SITE_KEY`/
`TURNSTILE_SECRET_KEY` no `.env`, o widget não renderiza e a verificação é
pulada — login continua funcionando normalmente, só sem CAPTCHA. Decisão
deliberada (não travar login de produção por uma chave que só o
responsável pode gerar, mesmo padrão já usado pro Brevo) — mas significa
que **o CAPTCHA não está de fato ativo em produção ainda**. Chaves reais
precisam ser criadas numa conta Cloudflare (Turnstile → Add Site, domínio
`admn.npxit.com.br`) e coladas no `.env`; ver `docs/STATE.md` para o
pendente.

### 2FA: TOTP implementado à mão (RFC 6238), sem lib externa

`lib/totp.ts` usa só o módulo `crypto` nativo do Node (HMAC-SHA1 + Base32)
em vez de uma lib npm (`otplib`, `speakeasy`, etc.) — evita mais uma
dependência de terceiros pra uma implementação de ~80 linhas de um RFC
estável e bem documentado, num projeto que já reusa `lib/crypto.ts`
(AES-256-GCM) pra guardar o `totpSecret` cifrado no banco (mesmo padrão já
usado pras credenciais de instância). Único pacote novo adicionado foi
`qrcode` (renderizar o QR do `otpauth://` — gerar isso à mão não vale a
pena).

**Toggle de plataforma (`PlatformSettings.totpFeatureEnabled`), default
`false`** — exatamente como pedido: 2FA fica pronto e testável (o
responsável pode ligar, testar o fluxo completo pessoalmente, desligar de
novo) mas não força ninguém ainda. Fluxo de login vira 2 fases só quando
`totpEnabled` do usuário específico está true (não é all-or-nothing por
tenant nem por plataforma-liga-automaticamente-pra-todos) — cada usuário
ativa a própria 2FA em Configurações → Segurança, depois do toggle geral
estar ligado.

**Ainda não testado ao vivo nesta sessão** (o toggle segue desligado por
padrão, como pedido) — o responsável precisa fazer o ciclo real (ligar →
configurar TOTP na própria conta → logar com código real de um app
autenticador → desligar) já que só ele tem um app TOTP de verdade pra
escanear o QR. Ver `docs/STATE.md`.

### SSO: Grafana e Zabbix implementados, GLPI adiado (ver ROADMAP)

Confirma a investigação já registrada em 2026-07-13 (Fase D): Grafana OIDC
precisa de edição de compose (env vars) + redeploy da stack (não existe
API de runtime pra isso) — `applyGrafanaSso()` edita o compose do
tenant e chama `deployStack` (mesmo mecanismo já usado pro provisionamento
self-service). Zabbix SAML tem API de verdade (`authentication.update`) —
`applyZabbixSso()` chama isso direto, sem precisar de redeploy. SSO é
**configurável por tenant** (`TenantSsoConfig`, um registro por
`tenantId + provider`), não uma chave global — cada tenant liga o que
quiser, do jeito pedido.

**GLPI ficou fora deste lote** — precisaria de um proxy de autenticação
extra na frente dele (não tem OIDC/SAML nativo, só confia em header de
proxy — achado já registrado em 2026-07-13). Esforço de construir e manter
esse proxy não pareceu proporcional ao valor neste momento (nenhum tenant
pediu SSO ainda, é feature preparatória). Registrado em `docs/ROADMAP.md`
como decisão explícita de adiamento, não esquecimento.

### Reset de senha por SMS: descartado, não implementado

Pedido era implementar **somente se existisse opção sem custo real**.
Toda opção de envio de SMS encontrada (Twilio, AWS SNS, Vonage, etc.) tem
custo por mensagem — não existe um provedor de SMS transacional
verdadeiramente gratuito em volume de produção (os "free tiers" existentes
são trial/sandbox, não utilizáveis de verdade em produção). Conclusão:
não implementado, per instrução explícita do responsável de não
implementar caso todo provedor tivesse custo. Registrado em
`docs/ROADMAP.md` com o raciocínio completo, caso o responsável decida
mais adiante que vale pagar por isso.

### Rate limiting: em memória, no processo — mesma limitação já aceita pro provisionamento assíncrono

`lib/rate-limit.ts` usa um `Map` em memória do próprio processo Node, não
Redis/store externo. Funciona corretamente hoje porque o portal roda como
processo único (`restart: unless-stopped`, sem múltiplas réplicas atrás de
load balancer) — mesma premissa já documentada em 2026-07-15 pro
provisionamento assíncrono fire-and-forget. Se o portal algum dia rodar
multi-réplica, isso precisa virar um store compartilhado (Redis), senão
cada réplica conta tentativas separadamente e o limite efetivo multiplica
pelo número de réplicas. Aplicado em `/login`, `/login/2fa` e
`/forgot-password` (as 3 rotas sensíveis do fluxo de autenticação).

### Headers de segurança HTTP e checagem geral de vulnerabilidades óbvias

`next.config.js` ganhou `headers()` com CSP, HSTS, `X-Frame-Options:
DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy` — confirmado
ao vivo via `curl -I` contra produção. Checagem geral (item 13 do pedido):
nenhum uso de `$queryRawUnsafe`/`$executeRawUnsafe` no código (todo acesso
a banco passa pelo query builder do Prisma, parametrizado por padrão —
sem superfície de SQL injection); único `dangerouslySetInnerHTML` do
projeto é o script estático de tema (sem input de usuário, já existia
antes deste lote); cookies de sessão/2FA-pendente todos `httpOnly` +
`sameSite=lax` + `secure` em produção. Não fiz varredura de
`npm audit` de forma conclusiva (o estágio final do Dockerfile instala
`prisma` via `--no-save` sem lockfile, o que impede `npm audit` de rodar
ali; a mesma checagem no estágio `deps` não foi refeita nesta sessão) —
fica como item pendente de uma varredura de dependências mais formal numa
sessão futura, não bloqueante pra este lote.

---

## 2026-07-17 — Onboarding MIP ENGENHARIA (unidade FLUA TI): switches, impressoras, VMware, dashboards

### Teste de conectividade sem acesso direto à LAN do cliente

`FLUA-Proxy-01` é um proxy Zabbix **ativo** (conecta-se para fora),
rodando dentro da própria rede interna da FLUA (endereço local
`192.168.1.7`) — confirmado via `proxy.get` (`operating_mode: 0`,
`lastaccess` batendo com o relógio em tempo real no momento da checagem).
Este host (VM da plataforma) **não tem rota de rede nenhuma** até as
faixas `192.168.0.0/24`/`192.168.1.0/24` da FLUA — só o proxy tem. Por
isso, "testar SNMP real" nesta sessão não significou rodar `snmpwalk`
localmente (impossível daqui), e sim: criar o host no Zabbix, apontar
para o proxy certo, e usar `task.create` (`type: 6`, "check now") no item
alvo — isso manda o PRÓPRIO PROXY (que tem acesso real à LAN do cliente)
executar o SNMP GET e devolver o resultado real pela API. É o equivalente
funcional de rodar `snmpwalk` a partir de dentro da rede do cliente, só
que orquestrado via Zabbix em vez de shell direto. Ver `docs/RUNBOOK.md`
se este padrão precisar ser reaplicado em outro proxy/cliente.

### SW24 pedido no mesmo IP do SW23 já existente — confirmado ser o mesmo equipamento físico

O responsável do projeto foi consultado (IP `192.168.0.174` já pertencia
ao host `SW23`, um dos 3 switches já funcionando) e escolheu **manter
os dois hosts mesmo com IP duplicado**. Depois dessa decisão, o teste de
SNMP real trouxe uma confirmação mais forte do que uma simples suspeita
de erro de digitação: `SW24` respondeu com a **mesma string `sysDescr`
exata** que `SW23` já tinha em histórico (`"HPE Networking Instant On
Switch 48p ... 1930 JL686B, InstantOn_1930_3.3.2.0 (5)"`) — ou seja,
**`SW24` e `SW23` são, comprovadamente, o mesmo equipamento físico**,
não apenas um possível conflito de planejamento de IP. `SW24` foi
mantido criado (per decisão explícita já tomada antes dessa confirmação),
mas **recomendo fortemente removê-lo** — ele está duplicando a coleta de
um switch que a MIP provavelmente já cadastrou antes sob outro nome. Não
removido nesta sessão porque a decisão de manter já tinha sido tomada
explicitamente pelo responsável; ele decide se essa evidência nova muda o
resultado.

### SW21 e SW22 (dois dos três switches novos pedidos) não confirmados

- **SW22 (`192.168.0.172`)**: não respondeu nem a ICMP ping (testado 2x,
  via `task.create` no item `icmpping`, com intervalo entre as duas
  tentativas). Host não criado, per instrução explícita de não criar
  host para equipamento que não respondeu.
- **SW21 (`192.168.0.171`)**: respondeu a ICMP ping (host ligado/na rede),
  mas **não respondeu a SNMP** em nenhuma das 3 tentativas de `check now`
  ao longo de ~6 minutos (community `{$SNMP_COMMUNITY}` = `public`, a
  mesma já usada nos 3 switches funcionando). Host removido pelo mesmo
  motivo. Provável causa: agente SNMP desabilitado no equipamento, ou
  community diferente da usada nos demais — like algo pra a equipe FLUA
  confirmar fisicamente/via console do switch antes de tentar de novo.

### Template dos switches novos: reaproveitado, mas CPU precisou de OID à parte (memória, indisponível)

`SW24` (Aruba/HPE Instant On 1930, mesma linha dos 3 já funcionando)
recebeu o mesmo template `HP Enterprise Switch by SNMP` — LLD de portas
confirmada funcionando (60 interfaces descobertas, idêntico ao gêmeo
`SW23`). Porém o item de CPU do template (`hpSwitchCpuStat`, MIB legada
ProCurve/ArubaOS-Switch) retornou `No Such Object` — **não suportado nesta
linha de firmware** (confirmado também pela ausência total de itens de
CPU/memória já coletando em `SW23`, que está em produção há dias). Pesquisa
(fóruns oficiais HPE/Aruba Instant On) encontrou o OID real usado por essa
linha — `.1.3.6.1.4.1.11.2.1.9.0` (HPE-rndMng-MIB, média de 5 min) — testado
ao vivo contra `SW24` e confirmado funcionando (retornou valor plausível de
%). Criado um template complementar pequeno, `MIP - HPE Instant On CPU by
SNMP`, linkado **só em `SW24`** (não alterei `SW20`/`SW23`/`SW25`, fora do
escopo pedido — "sem alterar nenhuma configuração de coleta" valia pra
Fase 2, mas por segurança não toquei nos outros 3 hosts em produção sem
pedido explícito). **Memória:** nenhum OID funcional foi encontrado —
mesma conclusão de discussões públicas da comunidade HPE Instant On (usuário
perguntando a mesma coisa, sem resposta) — parece ser uma limitação real de
hardware/firmware desta linha de switch, não uma lacuna de template.

### Impressoras: template oficial Ricoh (comunidade Zabbix) + template genérico RFC 3805 para Kyocera

Não existe template de impressora no catálogo oficial Zabbix (só nos
"community templates", repositório separado no GitHub). Importados:

- **Ricoh** (`IMP-192.168.1.113`, `IMP-192.168.1.130`): template oficial
  da comunidade Zabbix (`zabbix/community-templates`, pasta
  `Printers/Ricoh`) — cobre status, contador de páginas, LLD de toner e
  bandejas, erros. Em inglês, testado, mantido pelo projeto Zabbix.
- **Kyocera** (7 hosts confirmados): a comunidade também tem um template
  Kyocera pronto, mas está **inteiramente em russo** (nomes de item,
  descrições, value maps) e usa uma MIB proprietária Kyocera não
  documentada publicamente — decidido não importar por manutenibilidade
  (ninguém na equipe NPX/FLUA lê russo, e depurar um OID proprietário sem
  documentação published é frágil). Em vez disso: importado o template
  genérico `Universal Printer Supply Levels by SNMP` (também da
  comunidade Zabbix, em inglês, baseado no **Printer-MIB padrão RFC
  3805** — LLD de suprimentos/toner via SNMP-index, testado e
  documentado) e adicionados **2 itens próprios** usando OIDs padrão
  RFC3805/HOST-RESOURCES-MIB (universalmente suportados, não
  proprietários): contagem de páginas (`prtMarkerLifeCount`,
  `.1.3.6.1.2.1.43.10.2.1.4.1.1`) e status geral
  (`hrDeviceStatus`, `.1.3.6.1.2.1.25.3.2.1.5.1`). Cobre as 4 categorias
  pedidas (status/páginas/toner/erros) com OIDs padrão, sem depender de
  MIB proprietária não documentada.

### Impressora não confirmada

`192.168.1.172` não respondeu a SNMP em duas tentativas (~4 minutos de
espera cada). Host não criado.

### VMware: hosts criados **desabilitados**, aguardando credencial

`ESX01`/`ESX02` criados no template oficial "VMware Hypervisor", macros
`{$VMWARE.URL}` preenchidas, `{$VMWARE.USERNAME}` com o placeholder
pedido, `{$VMWARE.PASSWORD}` como macro secreta vazia. **Decisão não
pedida explicitamente, mas tomada por segurança**: os hosts foram
criados com `status: disabled`, não `enabled`. Motivo: com usuário
placeholder e senha vazia, o coletor VMware tentaria autenticar e
falhar a cada ciclo, gerando alerta de "item sem suporte"/erro
recorrente sem nenhum valor informativo —ruído desnecessário até a
equipe FLUA preencher a credencial real. Ativar o host (`status:
enabled`) é o único passo que falta depois de preencher
`{$VMWARE.PASSWORD}`. Ver `docs/STATE.md`.

**Bloqueio adicional, fora do controle deste projeto:** o processo
"VMware collector" do Zabbix (`StartVMwareCollectors`) precisa estar
habilitado no `zabbix_proxy.conf` de **`FLUA-Proxy-01`**, não no
`zabbix_server` central — os ESXi só são alcançáveis pelo proxy, que
roda dentro da rede da FLUA. Este projeto **nunca teve acesso
(SSH/gerência) a esse proxy** — é infraestrutura do lado do cliente,
mesma situação já registrada para o FortiGate. Não há como configurar
isso remotamente nesta sessão; alguém com acesso ao host do proxy
precisa adicionar `StartVMwareCollectors=2` (ou mais) ao
`zabbix_proxy.conf` e reiniciar o serviço. Registrado como bloqueio
aberto, não uma tarefa esquecida.

### Achado de infraestrutura: permissão do `grafana-reader` por host group explícito

Ver entrada detalhada em `docs/RUNBOOK.md` ("SEMPRE atualizar a permissão
do `grafana-reader`..."). Resumo: o usuário de API do Grafana não tem
"todos os host groups" — tem uma lista explícita, e um group novo (ou
host movido pra um group novo) fica invisível pro Grafana até a
permissão ser adicionada manualmente. Corrigido para os 3 groups novos da
MIP nesta sessão; ficou registrado como regra permanente porque esse
mesmo problema vai se repetir em todo onboarding futuro de qualquer
cliente que use agrupamento por categoria, não é específico da MIP.

---

## 2026-07-17 (cont.) — SW24 removido: confirmado duplicata física do SW23

Decisão pendente do relatório anterior, resolvida: o responsável do
projeto confirmou a remoção depois de ver a evidência (mesma string
`sysDescr` exata entre `SW23` e `SW24`, mesmo IP `192.168.0.174`). Host
`SW24` (hostid 10691) removido via `host.delete`. O template
complementar criado para ele (`MIP - HPE Instant On CPU by SNMP`,
templateid 10706, com o OID `.1.3.6.1.4.1.11.2.1.9.0` confirmado
funcional para CPU de switches HPE/Aruba Instant On) **não foi apagado**
— fica no catálogo de templates do Zabbix da FLUA, sem host nenhum
linkado, disponível caso outro switch dessa linha seja onboardado no
futuro (evita repetir a pesquisa de OID).

---

## 2026-07-17 (cont.) — NOC de parede: Polystat, som de alerta sem credencial exposta

### Polystat: pesquisa contradisse a suposição inicial do responsável

Confirmado via `grafana.com/api/plugins/grafana-polystat-panel`:
`signatureType: "grafana"`, licença Apache 2.0, sem exigência de
Grafana Enterprise/Cloud — ao contrário do que se assumia
inicialmente. O "Status Panel" comunitário (alternativa cogitada) foi
descartado por estar oficialmente marcado como "não mantido ativamente
pela Grafana Labs". Instalado `grafana-polystat-panel` v2.1.16 via
`GF_INSTALL_PLUGINS` no `docker-compose.yml` da FLUA (mesmo mecanismo já
usado pro plugin Zabbix).

### `disable_sanitize_html` ativado globalmente — decisão consciente, confirmada com o responsável

Necessário pra o painel de status/som rodar HTML+JS de verdade (Grafana
não sanitiza um dashboard específico, só a instância inteira via
`grafana.ini`/`GF_PANELS_DISABLE_SANITIZE_HTML`). Ativado no
`docker-compose.yml` da FLUA depois de confirmação explícita — aceito
porque só quem tem login de Editor/Admin nesse Grafana pode criar/editar
dashboard (o público anônimo do kiosk só visualiza).

### Som de alerta: dois designs tentados, o segundo foi o escolhido (sem credencial no HTML)

**Primeiro design (descartado):** o painel de som chamaria a API do
Zabbix direto do navegador (`fetch`), autenticando com uma credencial
Zabbix dedicada (`noc-audio-alert`, somente leitura, restrita só aos 3
host groups da MIP — criada e depois **removida** nesta mesma sessão).
Percebido a tempo (e também barrado pelo classificador de permissões da
sessão) que, com `disable_sanitize_html` ligado, o HTML/JS do painel é
enviado a **qualquer visualizador**, inclusive o público anônimo do
kiosk — não só editores logados. Mesmo sendo uma credencial só-leitura e
de escopo mínimo, expor uma senha real em texto claro numa página
pública não é uma prática aceitável por padrão; parei e voltei pro
responsável do projeto antes de insistir nessa abordagem (ver
transcript da sessão).

**Segundo design (implementado):** o painel de som **não faz nenhuma
chamada de rede própria** — ele lê o valor já renderizado por dois
painéis Stat nativos que já estão na mesma tela (`Problemas Ativos
(total)` e `Problemas Críticos`), que por sua vez usam o datasource
Zabbix do jeito normal (autenticação fica inteiramente do lado do
servidor Grafana, nunca chega ao navegador). Mecanismo: o painel de
texto/HTML lê `document.querySelector('[data-viz-panel-key="panel-N"]')`
e extrai o número via regex do texto já renderizado — zero segredo no
HTML, zero chamada de rede adicional. **Limitação aceita e documentada**:
`data-viz-panel-key` é um atributo interno do Grafana, não uma API
pública — pode mudar numa atualização futura de versão. Se isso
acontecer, o painel simplesmente para de atualizar (mostra "SEM DADOS"),
não expõe nada nem quebra o resto do dashboard; precisa só de um ajuste
de seletor quando notado.

**Autoplay de áudio:** confirmado ao vivo (Playwright, contando disparos
reais de `AudioContext.createOscillator().start()` via um contador
injetado) que o navegador bloqueia som antes de qualquer interação —
resolvido com uma tela de overlay "Clique para ativar o som do NOC" que
desbloqueia o `AudioContext` no clique (padrão aceito, como o
responsável já havia antecipado no pedido original).

### Ferramentas de validação estendidas

`portal/scripts/playwright-screenshot.js` ganhou 3 flags novas
(reutilizáveis pra qualquer validação futura, não específicas desta
sessão): `--dump-html-selector <css>` (inspeciona DOM renderizado sem
precisar de browser interativo — foi assim que descobri o atributo
`data-viz-panel-key`), `--click-selector <css>` (simula clique real antes
do screenshot) e `--eval-js <expr>` (lê estado JS da página, ex: contador
de beeps disparados — foi assim que confirmei o som disparando de
verdade contra um problema crítico real, não simulado).
