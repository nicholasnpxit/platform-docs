# Estado atual — npx-platform

Última atualização: 2026-07-13 — **todas as fases planejadas (0-6 e A-D)
concluídas** nesta sessão longa: brand-kit, publicação docs, branding,
monitoramento da própria NPX, correções, portas, PT-BR, SSO (investigado),
NOC Grafana + mestre NPX, biblioteca de templates, e validação final
ampla. Ver seção "Sessão de branding/publicação/observabilidade" para as
Fases 0-3, "Correções e nova infraestrutura" para A-D e 4-6.

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
