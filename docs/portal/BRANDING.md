# Branding — brand-kit padrão NPX IT

Última atualização: 2026-07-12 (Fase 0 da sessão de branding/publicação).

## Fonte

Extraído de `/opt/npx-platform/portal/FILES/NPX/DOCS/`:
- `Manual Identidade Visual.pdf` (16 páginas, material original de ago/2019)
- `PAPEL TIMBRADO.docx` (modelo de papel timbrado — usado para confirmar
  dados de contato oficiais: telefone, e-mail, site)

O brand-kit estruturado vive em `/opt/npx-platform/portal/brand/npxit.json`
(consumido pelo código). Este documento é a versão legível/comentada do
mesmo conteúdo — se os dois divergirem no futuro, o `.json` é a fonte de
verdade (é o que o código lê).

## Cores

| Cor | Hex | RGB | CMYK | Pantone | Significado (manual) |
|---|---|---|---|---|---|
| Vermelho (primária) | `#ED3237` | 237,50,55 | 0,100,100,0 | P 48-8 C | "Representa atenção — atenção que temos com as soluções, suporte e segurança da informação." |
| Preto (secundária) | `#373435` | 55,52,53 | 0,0,0,100 | Process Black C | "Remete à informação que passamos de forma simples e prática." |

## Tipografia

- **Orbitron Black** — títulos e elementos que dialogam com o símbolo da
  marca (a fonte tem traços geométricos que ecoam o ícone quadrado).
  Fallback web: `'Orbitron', 'Arial Black', sans-serif`.
- **Avenir LT Std 45 Book** — texto corrido: folhetos, papelaria, folders,
  banners, impressos em geral, todo material de comunicação.
  Fallback web: `'Avenir', 'Segoe UI', system-ui, sans-serif`.

Nenhuma das duas é uma web font gratuita/padrão do sistema — em produção
web, o fallback é o que realmente vai renderizar na maioria das máquinas a
menos que se compre/embuta a fonte licenciada. Isso é uma limitação real,
não uma escolha de design: registrado aqui para não ser esquecido quando
alguém for implementar a UI com a tipografia "oficial".

## Conceito da marca

"A marca criada identifica e projeta visualmente o que a empresa NPX IT
proporciona para seus clientes: a tecnologia e segurança da informação.
Por isso o símbolo do Wi-Fi é a referência para a marca. Sua forma foi
alterada para a forma quadrada para dialogar com a fonte da marca."

## Logos disponíveis

Copiados para `/opt/npx-platform/portal/public/brand/npxit/` (servidos
pelo Next.js em `/brand/npxit/...`):

| Arquivo | Origem (FILES/NPX/LOGOS) | Descrição |
|---|---|---|
| `logo-light.png` | `logo_npxit_nova_fund_transp.png` | Ícone + "npx it" em preto, sem tagline. Uso geral em fundo claro. |
| `logo-dark.png` | `LOGOs_V2.0_fonte_Branca_transp_simp.png` | Ícone + "npx it" em branco, sem tagline. Uso geral em fundo escuro. |
| `logo-full-light.png` | `LOGOs_V2.0_completa_transp_preto.png` | Versão completa com tagline "Management & Security", texto escuro. Uso formal (login, rodapé). |
| `logo-full-dark.png` | `LOGOs_V2.0 BRANCO.png` | Versão completa com tagline, texto branco. Uso formal em fundo escuro. |
| `icon.png` | `icone_npxit.png` | Só o símbolo (quadrado vermelho), sem wordmark. Espaços reduzidos. |
| `favicon.ico` | `icon.ico` | Favicon oficial. |

A pasta `FILES/` original **continua existindo** (não foi apagada), mas o
código/documentação nunca deve depender dela — só de
`public/brand/npxit/` e `brand/npxit.json`, que são a estrutura definitiva.

## Regras de uso (uso incorreto, direto do manual)

1. Não rotacionar a marca
2. Não alterar a proporção
3. Não utilizar o traço (contorno) no logotipo
4. Não alterar o logotipo
5. Não utilizar fundo degradê que comprometa a legibilidade
6. Não utilizar fundo que comprometa a legibilidade em geral

**Área de resguardo:** distância mínima "x" (altura da letra N do
logotipo) livre de qualquer elemento ao redor da marca.

**Tamanho mínimo:** 15mm de largura em impressos.

**Fundo colorido:** usar a versão a traço ou a traço negativa, a que der
mais contraste. A versão preferencial (cores normais) só em fundo branco
ou preto.

## Dados de contato oficiais (confirmados no papel timbrado)

- Razão social: NPX IT INOVAÇÃO EM TECNOLOGIA LTDA.
- Telefone: (31) 2342-0809
- E-mail: contato@npxit.com.br
- Site: www.npxit.com.br

## Como isso se conecta ao branding por tenant

Este brand-kit é o **fallback padrão**: qualquer tenant que não definir o
próprio `branding` (campo jsonb em `tenants`, ver
`docs/portal/ARCHITECTURE.md`) herda esses valores.

---

## Fase 2 — branding por tenant: o que cada ferramenta REALMENTE suporta

Investigado e **testado diretamente** contra o stack `flua` em produção
(não é suposição de documentação — cada linha da tabela foi confirmada
com uma chamada de API real ou inspeção do código-fonte/arquivos dentro
do container rodando).

| Ferramenta | Logo | Cor | Favicon | Tema claro/escuro |
|---|---|---|---|---|
| **GLPI** | ✅ nativo | ✅ nativo | ✅ volume mount | n/a |
| **Zabbix** | ❌ não existe | ❌ não existe | ✅ volume mount | ✅ nativo (API) |
| **Grafana OSS** | ❌ Enterprise-only | ❌ Enterprise-only | ✅ volume mount (mais barato que o esperado — ver nota) | ✅ nativo (API) |

### GLPI — logo e cor: nativo via `custom_css_code` da Entity

GLPI (11.0.8, CE) tem um campo nativo por Entity: `enable_custom_css` +
`custom_css_code` (tabela `glpi_entities`, gerenciável via
`PUT /apirest.php/Entity/<id>`). O logo do GLPI é renderizado como
`<span class="glpi-logo">` com `background-image` via CSS — então
`custom_css_code` pode sobrescrever isso apontando para uma URL externa
(o asset do tenant, servido pelo próprio portal), e também sobrescrever a
variável `--glpi-mainlibcolor` para a cor de marca.

**Testado ao vivo**: apliquei em `glpi.flua.npxit.com.br` (Entity raiz,
id 0) `enable_custom_css=1` com CSS apontando para o brand-kit padrão da
NPX (já que a FLUA ainda não definiu o próprio) — confirmei via `grep` no
HTML servido que a URL do logo e a cor `#ED3237` aparecem no `<style>`
renderizado da página de login.

Favicon do GLPI é um arquivo estático fixo
(`/var/www/glpi/public/pics/favicon.ico` dentro do container) — troca via
volume mount, não via API/config.

### Zabbix — sem white-label nativo, só tema

Confirmado via `settings.get`/`settings.update` da API: existe
`default_theme` (`blue-theme` ou `dark-theme`, nativo, **testado com
sucesso**), mas não há nenhum campo de logo/cor de marca em lugar nenhum
da API ou dos arquivos estáticos com nome óbvio de "logo" (procurei —
só existem ícones fixos tipo `apple-touch-icon-*.png`, sem um "logo
principal" separado e substituível). **Não implementei logo/cor porque a
capacidade não existe** — não é limitação da implementação, é limitação
real do produto.

Favicon: arquivo estático fixo (`/usr/share/zabbix/favicon.ico` dentro do
container `zabbix-web`) — troca via volume mount.

### Grafana OSS — sem white-label nativo, só tema (e o favicon é mais barato que o previsto)

Confirmado: `defaults.ini` do Grafana OSS **não tem a seção
`[white_labeling]` de jeito nenhum** (ela só existe em builds Enterprise
com licença válida) — trocar logo/nome da aplicação não é possível sem
comprar Enterprise ou recompilar o frontend do zero (fora de cogitação).
Tema (`light`/`dark`) é nativo via `PUT /api/org/preferences`, **testado
com sucesso**.

**Achado que muda a previsão inicial:** o favicon do Grafana **não
precisa de rebuild de imagem** — são arquivos estáticos simples e
previsíveis (`/usr/share/grafana/public/build/img/fav32.png` e
`apple-touch-icon.png`), trocáveis por volume mount como os outros dois.
A suposição inicial de que isso exigiria recompilar a imagem Docker
**estava errada para melhor** — é tão barato quanto GLPI/Zabbix.

### Implementação

- `portal/src/lib/branding.ts` — funções `applyGlpiBranding`,
  `applyZabbixBranding`, `applyGrafanaBranding`, cada uma aplicando só o
  que a ferramenta suporta de verdade (o resto fica em `skipped` com o
  motivo, nunca finge sucesso).
- Tela `/tenants/<id>/branding` no portal — super_admin (qualquer tenant)
  ou gestor (só o próprio) pode disparar a aplicação. Pede a credencial
  admin de cada ferramenta no momento (não fica guardada — arquitetura
  atual do portal não armazena senha de instância, só referência de onde
  ela mora, ver `docs/portal/ARCHITECTURE.md`).
- **Limite honesto desta fase**: a lógica de cada chamada de API foi
  testada diretamente (curl) e confirmada funcionando; a tela em si
  (`BrandingForm`, componente client-side com `useFormState`) foi
  validada por compilação + carregamento da página (200), mas não por um
  teste de ponta a ponta via navegador real nesta sessão — o mecanismo de
  submissão de Server Action client-side não é trivial de reproduzir via
  curl puro (diferente das actions server-only testadas em fases
  anteriores). Recomendo validar visualmente antes de anunciar a feature
  para clientes.

### Favicon via volume mount — padrão a aplicar quando um tenant subir o próprio

Ainda não aplicado no `docker-compose.yml` do FLUA porque ele não tem
favicon próprio (montar um volume apontando pra um arquivo que não existe
quebra o container). Quando o tenant subir os arquivos em
`clients/<tenant>/branding/`, adicionar ao serviço correspondente:

```yaml
# GLPI
volumes:
  - ../clients/<tenant>/branding/favicon.ico:/var/www/glpi/public/pics/favicon.ico:ro

# Zabbix web
volumes:
  - ../clients/<tenant>/branding/favicon.ico:/usr/share/zabbix/favicon.ico:ro

# Grafana
volumes:
  - ../clients/<tenant>/branding/favicon.ico:/usr/share/grafana/public/build/img/fav32.png:ro
```

### Estrutura de pasta para upload futuro

`/opt/npx-platform/clients/<tenant>/branding/` — convenção:
`logo-light.png`, `logo-dark.png`, `favicon.ico`. Criada para o FLUA como
referência (`clients/flua/branding/README.md`), ainda vazia (FLUA não
definiu branding próprio).

Validação de formato/tamanho: `portal/src/lib/branding-upload.ts`
(`validateBrandingFile`) — valida tipo MIME e tamanho de arquivo. **Não
valida dimensões reais em pixels** (exigiria decodificar a imagem, ex.
biblioteca `sharp`, dependência nativa pesada — não incluída nesta fase
de fundação). A tela de upload em si (rota que recebe o multipart e
grava o arquivo) **não foi construída** — depende de provisionamento
self-service, que é item de `docs/ROADMAP.md`. A validação já fica pronta
para quando essa tela existir.
