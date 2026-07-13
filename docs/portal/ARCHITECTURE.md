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

**Simplificação assumida nesta fase:** a autorização é 100% baseada no
campo `papel` do usuário, não em checar se o tenant do usuário é
efetivamente a raiz (`parentTenantId IS NULL`). Ou seja, nada no código
impede — a nível de dado — criar um usuário com papel `super_admin` dentro
de um tenant filho; a convenção "super_admin vive no tenant raiz" é
mantida pela tela de criação de usuário (só oferece a opção `super_admin`
no seletor) e pelo seed, não por uma constraint de banco. Documentado aqui
para não ser esquecido se isso virar um problema real mais adiante.

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
