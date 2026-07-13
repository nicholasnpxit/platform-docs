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
`docs/portal/ARCHITECTURE.md`) herda esses valores. A implementação de
propagação real (o que cada ferramenta — GLPI/Zabbix/Grafana —
efetivamente suporta) está descrita na Fase 2 deste mesmo documento
(seção abaixo, adicionada quando essa fase rodou nesta sessão).
