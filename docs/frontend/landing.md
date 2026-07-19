# Landing (Astro)

> Documento NOVO (v2.0). O app `apps/landing` — o que contém e como é deployado.

---

## O que é

App de marketing estático construído em **Astro 5**, hospedado em `redana.com.br`. Já está em produção (migrado do projeto anterior "NotaCerta" e renomeado para Redana).

Função:
- Comunica o produto (correção de redação do ENEM por competências)
- Captura emails / leva visitantes ao signup
- SEO + performance (Lighthouse 95+)

---

## Conteúdo

| Seção | Conteúdo |
|---|---|
| Hero | Headline + subheadline + CTA "Corrija sua redação grátis" → `app.redana.com.br/signup` |
| Como funciona | 3 passos (envie → IA corrige → evolua por competências) |
| Competências | Cards C1-C5 explicando cada uma em PT-BR (referência: rubrica INEP) |
| Pricing | Reaproveitado no dashboard via `PricingCards` (mesmo CSS via `packages/shared` tokens) |
| Depoimentos | `TestimonialCarousel.astro` |
| FAQ | Accordion de objeções comuns |
| Footer | Política/Privacidade/LGPD/DPO + link Google OAuth |

---

## Arquivos relevantes

```
apps/landing/
├── src/
│   ├── pages/index.astro
│   ├── layouts/BaseLayout.astro
│   ├── components/
│   │   └── TestimonialCarousel.astro
│   └── styles/global.css
├── public/
│   ├── favicon.svg
│   ├── robots.txt
│   ├── sitemap.xml
│   └── llms.txt             # AI SEO — ver ai-seo (pós-MVP)
├── astro.config.mjs
└── package.json
```

---

## Integração com restante do monorepo

- Importa `tokens.ts` e `constants/plans.ts` de `packages/shared` (via pnpm workspace).
- CTAs apontam para `https://app.redana.com.br/signup` (env var `DASHBOARD_SIGNUP_URL`).

> **Importante:** A landing é estática — não faz chamadas à API. Não tem auth, não tem estado de usuário.

---

## SEO

- Sitemap em `public/sitemap.xml` (gerado na build via `@astrojs/sitemap`)
- `robots.txt` permite crawl da landing e bloqueia `/dashboard/*` (mesmo que não tenha route real na landing, evita indexar o dominio do app)
- Meta tags OG/Twitter no `BaseLayout.astro` usando content específico por section
- JSON-LD `SoftwareApplication` schema (ver [schema.md](../api/contracts.md) — schema no sentido de structured data; detalhes pós-MVP)

## Lighthouse

| Métrica | Meta |
|---|---|
| Performance | > 95 |
| Accessibility | > 95 |
| Best Practices | > 95 |
| SEO | > 95 |

---

## Deploy

- **Vercel** — projeto separado do `apps/web`
- Build: `pnpm --filter landing build`
- Output: `dist/` estático
- Domain mapping: `redana.com.br`

## Pré-Changelog (Fase 4)

- [ ] Linkar todos CTAs para `app.redana.com.br/signup` (env var `DASHBOARD_SIGNUP_URL`)
- [ ] Adicionar link para `/login` no canto "Já tenho conta"
- [ ] Garantir que `PricingCards` use valores vindos de `packages/shared` (single source of truth para preços)