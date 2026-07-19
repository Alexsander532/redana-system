# Fase 5 — Admin Panel

## Proposal
Construir o painel administrativo em app Next.js 14 separado (`apps/admin`, dominio `admin.redana.com.br`). Admin unico (owner) com 2FA TOTP, IP allowlist, audit log imutavel, e controle total sobre usuarios, redacoes, correcoes, assinaturas, pagamentos, metricas e configuracao do sistema.

## Why
O admin panel permite operar o negocio sem depender de acesso direto ao banco de dados. Correcao humana, gestao de assinaturas problematicas, metricas de negocio e broadcast de emails sao essenciais para escalar a operacao.

## Tasks

### Infra + Auth
- [ ] 5.1 Migrations: users.role, admin_audit_log, feature_flags, prompt_templates, broadcast_emails, webhook_events, llm_calls, pricing_plans
- [ ] 5.2 Scaffolding apps/admin Next.js 14 + config (admin.redana.com.br, Vercel)
- [ ] 5.3 Admin auth: login gate (role=admin + 2FA TOTP), middleware, CORS restrito
- [ ] 5.4 FastAPI admin module: dependencies (get_current_admin, require_admin_no_impersonation, audit decorator)

### Usuarios e Redacoes
- [ ] 5.5 API admin/users: list, search, get, suspend, ban, unban, change role
- [ ] 5.6 API admin/essays: list all, filter by status/user, get detail, reprocess AI
- [ ] 5.7 Front admin/usuarios: UserTable + UserDetail (essays, subscription, actions)
- [ ] 5.8 Front admin/redacoes: EssayTable + EssayDetail

### Correcao Humana
- [ ] 5.9 API admin/corrections: PUT manual override (C1-C5 + feedbacks)
- [ ] 5.10 Front HumanCorrectionForm + fila de correcoes pendentes

### Assinaturas
- [ ] 5.11 API admin/subscriptions: list, force status, grant corrections, grant plan, extend, refund
- [ ] 5.12 Front admin/assinaturas: SubscriptionTable + SubscriptionEdit

### Pagamentos
- [ ] 5.13 API admin/payments: webhook event log, reconciliation trigger
- [ ] 5.14 Front admin/pagamentos: WebhookEventTable + replay

### Analytics
- [ ] 5.15 API admin/analytics: MRR, active subs, conversion, corrections/day, LLM cost/day, avg latency, failure rate
- [ ] 5.16 Front admin/metricas: KPI cards + charts (Recharts)

### Sistema
- [ ] 5.17 API admin/system: prompt templates CRUD, feature flags CRUD, pricing config
- [ ] 5.18 API admin/emails: broadcast send + history
- [ ] 5.19 Front admin/sistema: PromptTemplateEditor, FeatureFlagToggle, PlanEditor, BroadcastEmailForm, AuditLogTable

### Impersonation + Audit
- [ ] 5.20 API admin/impersonate: start (scoped token 30min) + end
- [ ] 5.21 Front impersonation banner + restricoes de UI durante impersonation
- [ ] 5.22 Audit log: read-only view no admin/sistema/audit

### Seguranca
- [ ] 5.23 IP allowlist middleware para /api/v1/admin/*
- [ ] 5.24 Rate limit agressivo no admin (5 tentativas login/10min, 120 req/min)
- [ ] 5.25 Testes E2E admin: login → suspend user → human correction → grant plan → broadcast

## Dependencies
- Fases 0-4 completas (MVP)
- Supabase MFA/TOTP habilitado

## Effort
~12 dias-calendario (2 devs)
