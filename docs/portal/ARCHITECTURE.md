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
