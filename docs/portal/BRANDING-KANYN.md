# Branding KANYN — manual de marca do produto

**Vigente desde 2026-09-04.** Fonte de verdade do **produto**. O arquivo
`docs/portal/BRANDING.md` é **histórico** da NPX IT (que continua existindo
só como tenant cliente).

## Nome

| Constante | Onde | Valor padrão |
|---|---|---|
| `BRAND_NAME` | `portal/src/lib/brand.ts` | `KANYN` (override: `NEXT_PUBLIC_BRAND_NAME`) |
| `BRAND_TAGLINE` | idem | Plataforma de instâncias prontas para uso |
| `BRAND_TITLE` | idem | `{BRAND_NAME} — Painel` |

**Regra:** nenhum arquivo escreve o nome do produto em literal. UI usa
`<Wordmark />` ou importa `BRAND_NAME`.

## Símbolo (v4)

- Geometria: só retas, `fill-rule: evenodd` — espelho em
  `portal/src/components/Wordmark.tsx` (`MARK_PATH`) e
  `scripts/render-brand-mark.py` (`PIECES`).
- Se a geometria mudar: atualize **os dois** e rode
  `python3 scripts/render-brand-mark.py`.

### Assets servidos

Diretório: `portal/public/brand/kanyn/`

| Arquivo | Uso |
|---|---|
| `mark.svg` | Símbolo no portal (Nativa / Brasa) |
| `mark-obsidiana.svg` | Símbolo na direção Obsidiana / Prisma |
| `mark-64.png` / `mark-256.png` / `mark-512.png` | Raster Nativa |
| `mark-obsidiana-*.png` | Raster Obsidiana |
| `favicon.ico` | Favicon do portal e fallback injetado nas ferramentas |

Base absoluta para URLs fora do portal (`lib/branding.ts`):
`BRAND_ASSET_BASE` / `NEXT_PUBLIC_BRAND_ASSET_BASE`
(hoje `https://admn.npxit.com.br` até a virada de DNS KANYN).

## Cores

### Assinatura (só IA)

Gradiente **Brasa** (Nativa) / **Prisma** (Obsidiana) — classes
`sig-fill`, `sig-text`, `sig-glow` em `globals.css`. **Único** lugar onde
gradiente é permitido na UI.

### Cor de marca nas ferramentas (fallback)

Quando o tenant **não** define branding próprio:

- Hex: `#1F6FE5` (azul KANYN — `resolveBranding` em `lib/branding.ts`)
- PDFs comerciais / login: mesmo fallback (`commercial-pdf.ts`,
  `auth-branding.ts`)

### Tokens semânticos da UI

Em `portal/src/app/globals.css`: `--color-accent*`, `--color-success`,
`--color-warning`, `--color-danger`, `--color-info`. Nunca cor Tailwind
numérica (`bg-red-500` etc.) em tela de produto.

## Direções visuais

| Direção | Tema | Quando |
|---|---|---|
| **Nativa** | escuro (padrão) / claro | produto principal |
| **Obsidiana** | escuro / claro | opção do usuário em Aparência |

Seletor: `/settings/appearance`. Miniatura `ThemePreview` usa cor literal
**de propósito** (exceção documentada).

## Tipografia

- Display: fonte do design system Nativa (`font-display` no Tailwind)
- Corpo: tipografia do sistema via tokens
- Dado técnico: sempre `font-mono` + `tabular-nums`

## O que cada ferramenta aceita

Matriz testada (válida independente da marca) — ver segunda metade de
`docs/portal/BRANDING.md`:

| Ferramenta | logo | cor | favicon | tema |
|---|---|---|---|---|
| GLPI | sim (CSS Entity) | sim | volume | n/a |
| Zabbix | não | não | volume | sim (API) |
| Grafana OSS | não (Enterprise) | não | volume | sim (API) |

Aplicação: `portal/src/lib/branding.ts`. Retroação em instâncias antigas:
`scripts/rebrand-instances.py` (dry-run / `--apply`; nunca toca white-label).

## Arquivo da marca antiga (NPX IT)

Aposentado em 2026-09-06:

- `portal/brand/_archived_npxit_2026-09-06/` (json + PNGs)
- **Não** está mais em `public/brand/npxit/` (não é servido)

NPX como tenant pode ter branding próprio no jsonb — isso é dado de
cliente, não identidade do produto.
