# Plano de entrega por fases

> Seção original: PLAN.md §15. Distribuído em fases sequenciais. **Admin Panel agora Fase 5 legítima (não deferrido)**.

---

## Fase 0 — Scaffolding (1 dia)

| # | Tarefa | Dep. |
|---|---|---|
| 0.1 | Iniciar monorepo: `pnpm-workspace.yaml`, `turbo.json`, root `package.json`, `.gitignore`, `.env.example` | — |
| 0.2 | Criar `packages/shared` com types + constants (competências, planos) + design tokens | 0.1 |
| 0.3 | Criar `packages/db/schema.sql` + RLS + seed | 0.1 |
| 0.4 | Criar `packages/email` com templates React Email | 0.1 |
| 0.5 | Migrar landing Astro para `apps/landing` | 0.1 |
| 0.6 | Git init, commit inicial, push para GitHub `redana` (repo já existe) | 0.5 |

---

## Fase 1 — Infra + Auth + Shell (3-4 dias)

| # | Tarefa | Dep. |
|---|---|---|
| 1.1 | Criar projeto Supabase, configurar auth providers (email+Google), obter URL + anon key | — |
| 1.2 | Aplicar `schema.sql` no Supabase + RLS policies | 0.3 |
| 1.3 | Scaffolding `apps/api` FastAPI: main, config, db, alembic, `/health` | 0.1 |
| 1.4 | Auth router: signup/login/me com verify Supabase JWT | 1.3 |
| 1.5 | Scaffolding `apps/web` Next.js 14 + Tailwind + configs de design tokens | 0.2 |
| 1.6 | Supabase clients (browser + server) + middleware de auth | 1.5 |
| 1.7 | Telas de login + signup (email + Google) | 1.6 |
| 1.8 | Shell do dashboard: Sidebar + Topbar + UserMenu + responsivo | 1.6 |

---

## Fase 2 — Submissão + Correção IA (5-6 dias)

| # | Tarefa | Dep. |
|---|---|---|
| 2.1 | API: `POST /essays`, `GET /essays`, `GET /essays/{id}`, `DEL /essays/{id}` | 1.4 |
| 2.2 | Prompt templates ENEM (C1-C5) com schema JSON de saída | — |
| 2.3 | CorrectionEngine: chiamada Claude + fallback GPT-4o + validação + retry | 2.2 |
| 2.4 | Rate limiter: verificar `corrections_used < max_corrections` | 2.1 |
| 2.5 | Background task: enqueue + status pending→processing→completed | 2.3 |
| 2.6 | Front `EssayForm` (texto + upload .txt/.docx) + `FreeTierCounter` | 1.8, 2.1 |
| 2.7 | Front `EssayHistory` na home do dashboard | 1.8, 2.1 |
| 2.8 | Front `redacoes/[id]`: ScoreHeader + CompetencyBars + NextSteps | 1.8, 2.1 |
| 2.9 | Front `EvolutionChart` (Recharts) na home | 2.7 |

---

## Fase 3 — Pagamentos AbacatePay (4-5 dias)

| # | Tarefa | Dep. |
|---|---|---|
| 3.1 | Criar conta AbacatePay + produtos (Monthly, Annual, Quarterly) | — |
| 3.2 | `abacatepay_service.py` (SDK Python: criar checkout, cancel assinatura, list) | 1.3 |
| 3.3 | `subscriptions` router: checkout/cancel/change-plan | 3.2 |
| 3.4 | Webhook handler com HMAC verify + handlers de eventos | 3.2 |
| 3.5 | Job de reconciliação diário (sync subscription status via API list) | 3.4 |
| 3.6 | Front `/planos` com PricingCards + integração checkout | 1.8, 3.3 |
| 3.7 | `UpgradeBanner` quando free tier esgota | 2.4 |
| 3.8 | Tela `/conta` com status, cancelar, trocar, exportar (LGPD) | 1.8, 3.3 |

---

## Fase 4 — Polimento + Integração (3-4 dias)

| # | Tarefa | Dep. |
|---|---|---|
| 4.1 | Linkar landing CTAs para `app.redana.com.br/signup` | 0.5, 1.7 |
| 4.2 | Emails transacionais (Resend): welcome, correção-pronta, reset | 0.4, 2.5 |
| 4.3 | Loading skeletons + error boundaries + toasts | Todas as telas |
| 4.4 | Responsividade fina (test 320, 768, 1024, 1440) | Todas |
| 4.5 | SEO das páginas públicas (login/signup) + robots.txt app | 1.5 |
| 4.6 | Testes E2E: fluxo completo signup → redação → correção → upgrade | Todas |
| 4.7 | Lighthouse + performance | Todas |

> Total MVP core = Fases 0-4 = ~4-6 semanas.

---

## Fase 5 — Admin Panel + Correção Humana (5-7 dias) — PRIMEIRA CLASSE

> Antes descrita como "pós-MVP deferrido". A partir da v2.0 é parte legitima do produto.

| # | Tarefa | Dep. | Detalhe |
|---|---|---|---|
| 5.1 | Migration `add role/status to users` + `admin_audit_log` | Fase 1.2 | [schema.md](../data/schema.md) |
| 5.2 | Migration `webhook_events`, `prompt_templates`, `feature_flags`, `plans_config`, `email_logs`, `events` | Fase 1.2 | [migrations.md](../data/migrations.md) |
| 5.3 | `apps/api/app/admin/` module: dependencies (`get_current_admin`), audit decorator, routers | Fase 1.3 | [auth-and-authorization.md](../api/auth-and-authorization.md) |
| 5.4 | Admin routers: users, essays, corrections, subscriptions, payments, analytics, system | 5.3 | [contracts.md § Admin](../api/contracts.md) |
| 5.5 | Admin RLS policies em todas tabelas (admin_all select/write) | 5.1 | [schema.md](../data/schema.md) |
| 5.6 | Next.js: `/admin/login` separado + middleware guard `/admin/*` | Fase 1.6 | [dashboard-routes.md](../frontend/dashboard-routes.md) |
| 5.7 | Next.js: shell admin (`AdminSidebar`, `AdminTopbar`) + components admin (`UserTable`, `EssayTable`, etc.) | 5.6 | [ui-components.md](../frontend/ui-components.md) |
| 5.8 | Tela `/admin/usuarios` + `/admin/usuarios/[id]` + ações suspend/ban/impersonate | 5.4, 5.7 | [admin/users-and-essays.md](../admin/users-and-essays.md) |
| 5.9 | Tela `/admin/redacoes` + `/admin/redacoes/[id]` com `HumanCorrectionForm` | 5.4, 5.7 | [admin/corrections-admin.md](../admin/corrections-admin.md) |
| 5.10 | Tela `/admin/assinaturas` + grant/refund/extend/status | 5.4, 5.7 | [admin/subscriptions-admin.md](../admin/subscriptions-admin.md) |
| 5.11 | Tela `/admin/pagamentos` + webhook event log + replay | 5.4, 5.7 | [admin/payments-and-webhooks.md](../admin/payments-and-webhooks.md) |
| 5.12 | Tela `/admin/metricas` + charts (`MrrChart`, `LlmCostChart`, `BreakdownChart`) | 5.4, 5.7 | [admin/analytics.md](../admin/analytics.md) |
| 5.13 | Tela `/admin/sistema` + prompts editor + flags + pricing + broadcast + logs | 5.4, 5.7 | [admin/system-config.md](../admin/system-config.md) |
| 5.14 | Audit log table view + filtros | 5.4, 5.7 | [admin/overview.md](../admin/overview.md) |
| 5.15 | Testes E2E: admin login → suspend user → reprocess essay → human correction → grant → refund → broadcast | Todas | [definition-of-done.md](./definition-of-done.md) |

---

## Fase 6 — MercadoPago + PIX avulso (2-3 dias, pós-MVP)

| # | Tarefa | Dep. |
|---|---|---|
| 6.1 | Implementar `mercadopago_service.py` (interface idêntica a `abacatepay_service`) | Fase 3 |
| 6.2 | Roteio de pagamento por feature flag `payments.provider` | 5.12 |
| 6.3 | Pix avulso (sem assinatura) para pacote de N correções ONE_TIME | Fase 3 |
| 6.4 | webhook handler MercadoPago | 6.1 |

---

## Total

- **MVP core (Fases 0-4):** ~4-6 semanas
- **+ Admin (Fase 5):** +5-7 dias → **~5-7 semanas no total (v2.0)**
- **+ MercadoPago (Fase 6):** +2-3 dias (pós-MVP)