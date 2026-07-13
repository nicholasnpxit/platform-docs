# Sessão 2026-07-12 — brand-kit, publicação, branding, observabilidade

Changelog de alto nível (não é log bruto de terminal, nenhum segredo).
Atualizado ao final de cada fase da sessão.

## Fase 0 — brand-kit NPX IT

Manual de identidade visual e papel timbrado lidos por completo. Brand-kit
estruturado criado (`portal/brand/npxit.json`) com cores oficiais
(`#ED3237` vermelho, `#373435` preto), tipografia (Orbitron Black +
Avenir LT Std 45 Book) e regras de uso. Logos copiados para
`portal/public/brand/npxit/`. Documentado em `docs/portal/BRANDING.md`.

## Fase 1 — publicação da documentação

Dois repositórios GitHub configurados com propósitos separados:
- `admn` (privado) — backup completo do projeto via `scripts/backup-source.sh`.
- `platform-docs` (público) — só documentação sanitizada via
  `scripts/publish-docs.sh`, com checagem automática de padrões de segredo
  antes de cada push.

Ver `docs/DECISIONS.md` para o racional completo da separação e o risco
aceito conscientemente sobre incluir segredos reais no backup privado.
