# Estrutura do monorepo (árvore completa)

> Seção original: PLAN.md §5. Layout de arquivos do monorepo Turborepo + pnpm.

A base do projeto é `C:\Users\Alexsander\Desktop\redana\`.

```
C:\Users\Alexsander\Desktop\redana\
├── apps/
│   ├── landing/                      # Astro — marketing (migrado do projeto notacerta)
│   │   ├── src/
│   │   │   ├── pages/index.astro
│   │   │   ├── layouts/BaseLayout.astro
│   │   │   ├── components/
│   │   │   │   └── TestimonialCarousel.astro
│   │   │   └── styles/global.css
│   │   ├── public/
│   │   │   ├── favicon.svg
│   │   │   ├── robots.txt
│   │   │   ├── sitemap.xml
│   │   │   └── llms.txt
│   │   ├── astro.config.mjs
│   │   └── package.json
│   │
│   ├── web/                          # Next.js 14 — Dashboard do aluno
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── layout.tsx                # Root (fonts, ThemeProvider)
│   │   │   │   ├── page.tsx                  # Redireciona /login ou /dashboard
│   │   │   │   ├── (auth)/
│   │   │   │   │   ├── layout.tsx
│   │   │   │   │   ├── login/page.tsx
│   │   │   │   │   ├── signup/page.tsx
│   │   │   │   │   ├── recuperar/page.tsx
│   │   │   │   │   └── redefinir/page.tsx
│   │   │   │   ├── (dashboard)/
│   │   │   │   │   ├── layout.tsx            # Shell com sidebar (aluno)
│   │   │   │   │   ├── page.tsx              # Home: evolução + últimas redações
│   │   │   │   │   ├── nova-redacao/page.tsx
│   │   │   │   │   ├── redacoes/
│   │   │   │   │   │   └── [id]/page.tsx    # Resultado da correção
│   │   │   │   │   ├── planos/page.tsx
│   │   │   │   │   ├── conta/page.tsx
│   │   │   │   │   └── suporte/page.tsx
│   │   │   ├── components/
│   │   │   │   ├── ui/                       # Button, Card, Input, Modal, ...
│   │   │   │   ├── auth/
│   │   │   │   │   ├── LoginForm.tsx
│   │   │   │   │   ├── SignupForm.tsx
│   │   │   │   │   └── GoogleButton.tsx
│   │   │   │   ├── redacao/
│   │   │   │   │   ├── EssayForm.tsx          # Texto + upload .txt/.docx
│   │   │   │   │   ├── FreeTierCounter.tsx
│   │   │   │   │   └── SubmissionProgress.tsx # SSE/polling
│   │   │   │   ├── correcao/
│   │   │   │   │   ├── ScoreHeader.tsx
│   │   │   │   │   ├── CompetencyBars.tsx     # C1-C5 com barras (igual landing)
│   │   │   │   │   ├── CompetencyFeedback.tsx
│   │   │   │   │   ├── NextSteps.tsx
│   │   │   │   │   └── OverallScoreBadge.tsx
│   │   │   │   ├── dashboard/
│   │   │   │   │   ├── EvolutionChart.tsx    # Recharts
│   │   │   │   │   ├── EssayHistory.tsx
│   │   │   │   │   ├── EssayRow.tsx
│   │   │   │   │   ├── PlanStatusCard.tsx
│   │   │   │   │   └── UpgradeBanner.tsx
│   │   │   │   ├── planos/
│   │   │   │   │   ├── PricingCards.tsx       # Reusa styles da landing
│   │   │   │   │   └── CurrentPlanBadge.tsx
│   │   │   │   ├── layout/
│   │   │   │   │   ├── Sidebar.tsx
│   │   │   │   │   ├── Topbar.tsx
│   │   │   │   │   └── UserMenu.tsx
│   │   │   ├── lib/
│   │   │   │   ├── supabase/
│   │   │   │   │   ├── client.ts             # Browser
│   │   │   │   │   ├── server.ts             # Server (cookies)
│   │   │   │   │   └── middleware.ts
│   │   │   │   ├── api.ts                    # fetch wrapper
│   │   │   │   ├── abacatepay.ts             # Helpers de checkout
│   │   │   │   └── utils.ts
│   │   │   ├── hooks/
│   │   │   │   ├── useAuth.ts
│   │   │   │   ├── useEssays.ts
│   │   │   │   ├── useSubscription.ts
│   │   │   │   └── useAdmin.ts              # Hooks admin (RBAC gated)
│   │   │   └── styles/globals.css
│   │   ├── middleware.ts                     # Auth redirects (incl. guard /admin)
│   │   ├── next.config.js
│   │   ├── tailwind.config.ts
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   └── api/                          # FastAPI — backend
│       ├── app/
│       │   ├── main.py                       # App FastAPI + routers
│       │   ├── config.py                     # Settings (pydantic-settings)
│       │   ├── database.py                   # Async SQLAlchemy
│       │   ├── dependencies.py
│       │   ├── admin/                        # NOVO v2.0 — Admin module
│       │   │   ├── dependencies.py            # get_current_admin (RBAC)
│       │   │   ├── audit.py                   # decorator/context para audit log
│       │   │   └── routers/
│       │   │       ├── users.py              # /admin/users
│       │   │       ├── essays.py             # /admin/essays
│       │   │       ├── corrections.py        # /admin/corrections
│       │   │       ├── subscriptions.py      # /admin/subscriptions
│       │   │       ├── payments.py           # /admin/payments
│       │   │       ├── analytics.py          # /admin/analytics
│       │   │       ├── system.py             # /admin/system
│       │   │       ├── prompts.py            # /admin/prompts
│       │   │       ├── plans.py              # /admin/plans
│       │   │       ├── feature_flags.py      # /admin/feature-flags
│       │   │       ├── emails.py             # /admin/emails/broadcast
│       │   │       └── logs.py               # /admin/logs
│       │   ├── models/
│       │   │   ├── user.py
│       │   │   ├── essay.py
│       │   │   ├── correction.py
│       │   │   ├── subscription.py
│       │   │   ├── admin_audit.py           # NOVO v2.0
│       │   │   ├── feature_flag.py          # NOVO v2.0
│       │   │   ├── prompt_template.py       # NOVO v2.0
│       │   │   ├── plan.py                  # NOVO v2.0 (config de preços)
│       │   │   ├── webhook_event.py         # NOVO v2.0 (log de webhooks)
│       │   │   └── email_log.py             # NOVO v2.0 (history de emails)
│       │   ├── schemas/
│       │   │   ├── auth.py
│       │   │   ├── essay.py
│       │   │   ├── correction.py
│       │   │   ├── subscription.py
│       │   │   └── admin.py                 # NOVO v2.0
│       │   ├── routers/
│       │   │   ├── auth.py                   # /auth/*
│       │   │   ├── essays.py                 # /essays/*
│       │   │   ├── corrections.py            # /corrections/*
│       │   │   ├── subscriptions.py          # /subscriptions/*
│       │   │   ├── webhooks_abacatepay.py    # /webhooks/abacatepay
│       │   │   └── account.py                # /account/* (LGPD: export/delete)
│       │   ├── services/
│       │   │   ├── correction_engine.py      # Orquestra LLM
│       │   │   ├── prompt_templates.py       # Prompts ENEM C1-C5
│       │   │   ├── abacatepay_service.py     # Cliente AbacatePay
│       │   │   ├── subscription_service.py   # Estado da assinatura
│       │   │   ├── rate_limiter.py           # Free tier / paid caps
│       │   │   ├── email_service.py          # Resend
│       │   │   ├── text_parser.py            # .txt/.docx extraction
│       │   │   ├── analytics_service.py      # NOVO v2.0 — queries métricas
│       │   │   └── prompt_template_service.py # NOVO v2.0
│       │   └── utils/
│       │       ├── enem_scoring.py           # Validação de scores
│       │       ├── webhook_signature.py      # HMAC AbacatePay
│       │       └── admin_rbac.py            # NOVO v2.0 — require_admin
│       ├── alembic/
│       │   ├── env.py
│       │   └── versions/                    # Migrations versionadas
│       ├── tests/
│       ├── Dockerfile
│       ├── requirements.txt
│       ├── pyproject.toml
│       └── .env.example
│
│   └── admin/                         # Next.js 14 — Admin Panel (admin.redana.com.br)
│       ├── src/
│       │   ├── app/
│       │   │   ├── layout.tsx                  # Root layout admin (fonts, ThemeProvider)
│       │   │   ├── page.tsx                    # Dashboard admin (KPIs)
│       │   │   ├── login/page.tsx              # Login admin (role-gated)
│       │   │   ├── usuarios/
│       │   │   │   ├── page.tsx                # Lista de usuários
│       │   │   │   └── [id]/page.tsx           # Detalhe do usuário
│       │   │   ├── redacoes/
│       │   │   │   ├── page.tsx                # Todas as redações (filtro)
│       │   │   │   └── [id]/page.tsx           # Detalhe + correção humana
│       │   │   ├── assinaturas/
│       │   │   │   ├── page.tsx                # Lista de assinaturas
│       │   │   │   └── [id]/page.tsx           # Edit assinatura
│       │   │   ├── pagamentos/page.tsx         # AbacatePay + webhooks
│       │   │   ├── metricas/page.tsx           # Charts (MRR, custo LLM, conversão)
│       │   │   ├── correcoes/page.tsx          # Fila de correção humana
│       │   │   └── sistema/
│       │   │       ├── page.tsx                # Prompt templates, flags, precos
│       │   │       ├── logs/page.tsx           # API logs + health
│       │   │       └── audit/page.tsx          # Audit log
│       │   ├── components/
│       │   │   ├── AdminSidebar.tsx
│       │   │   ├── AdminTopbar.tsx
│       │   │   ├── UserTable.tsx
│       │   │   ├── EssayTable.tsx
│       │   │   ├── SubscriptionTable.tsx
│       │   │   ├── WebhookEventTable.tsx
│       │   │   ├── PromptTemplateEditor.tsx
│       │   │   ├── FeatureFlagToggle.tsx
│       │   │   ├── PlanEditor.tsx
│       │   │   ├── BroadcastEmailForm.tsx
│       │   │   ├── AuditLogTable.tsx
│       │   │   ├── MetricCard.tsx
│       │   │   ├── MrrChart.tsx
│       │   │   ├── LlmCostChart.tsx
│       │   │   ├── BreakdownChart.tsx
│       │   │   └── HumanCorrectionForm.tsx
│       │   ├── lib/
│       │   │   ├── supabase/
│       │   │   │   ├── client.ts               # Browser (admin role)
│       │   │   │   ├── server.ts               # Server (cookies, role check)
│       │   │   │   └── middleware.ts
│       │   │   ├── admin-api.ts                # fetch wrapper /admin/*
│       │   │   └── utils.ts
│       │   ├── hooks/
│       │   │   ├── useAdminUsers.ts
│       │   │   ├── useAdminEssays.ts
│       │   │   └── useAnalytics.ts
│       │   └── styles/globals.css
│       ├── middleware.ts                       # Auth redirects + role gate
│       ├── next.config.js
│       ├── tailwind.config.ts
│       ├── tsconfig.json
│       └── package.json
│
├── packages/
│   ├── shared/
│   │   ├── src/
│   │   │   ├── types/            # essay, correction, user, subscription, admin
│   │   │   ├── constants/        # competencies.ts, plans.ts
│   │   │   └── tokens.ts        # Design tokens (cores do Redana)
│   │   ├── tsconfig.json
│   │   └── package.json
│   ├── db/
│   │   ├── schema.sql
│   │   ├── seed.sql
│   │   └── migrations/
│   └── email/
│       ├── templates/
│       │   ├── welcome.tsx
│       │   ├── reset-password.tsx
│       │   ├── correction-ready.tsx
│       │   ├── upgrade-reminder.tsx
│       │   └── broadcast.tsx          # NOVO v2.0
│       └── package.json
│
├── .github/
│   └── workflows/
│       └── ci.yml                # lint + test api/web em paralelo
├── .env.example
├── .gitignore
├── docs/                         # Esta documentação (v2.0)
│   ├── plan.md
│   ├── overview/
│   ├── architecture/
│   ├── data/
│   ├── api/
│   ├── frontend/
│   ├── features/
│   ├── admin/
│   └── process/
├── package.json                  # Raiz — workspaces pnpm
├── pnpm-workspace.yaml
├── turbo.json
└── PLAN.md                       # Documento histórico (v1.1) — superseded por docs/
```

> **Marcados `NOVO v2.0`:** adições arquiteturais para suportar o Admin Panel. Ver [`admin/`](../../admin/overview.md).