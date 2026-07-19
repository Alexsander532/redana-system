# Admin Panel — Visão geral

> Documento NOVO (v2.0). Entrada para o painel administrativo. Specs detalhadas em [`admin/`](../admin/overview.md).

---

## O que é

O **Admin Panel** é a interface onde o owner/operador do Redana controla todo o sistema: usuários, redações, correções, assinaturas, pagamentos, métricas, conteúdo e configuração. Até a v1.1 estava listado como "Fase 5 (pós-MVP deferrido)"; a partir da v2.0 passou a fase de primeira classe, totalmente documentada.

> **Decisão confirmada (Julho 2026):** app Next.js separado **`apps/admin`** em **`admin.redana.com.br`**, deploy na Vercel, isolamento total de auth/bundle/CORS. Ver [admin-architecture-addendum.md](../admin-architecture-addendum.md).

---

## Funções suportadas

| Função | Detalhe |
|---|---|
| Usuários | Listar, buscar, suspender, banir, impersonar, ver redações/assinaturas | [`admin/users-and-essays.md`](../admin/users-and-essays.md) |
| Redações | Navegar todas, filtrar por status, ver content + correction, reprocessar IA, editar correção | [`admin/users-and-essays.md`](../admin/users-and-essays.md) |
| Correções | Modo correção humana: admin marca C1-C5 e escreve feedback (override AI) | [`admin/corrections-admin.md`](../admin/corrections-admin.md) |
| Assinaturas | Ver todas, forçar status, grant de correções, grant/estender planos, disparar refund | [`admin/subscriptions-admin.md`](../admin/subscriptions-admin.md) |
| Pagamentos | Webhook event log, reconciliação, dashboard de pagamentos falhados | [`admin/payments-and-webhooks.md`](../admin/payments-and-webhooks.md) |
| Analytics | MRR, assinantes ativos, conversão Free→Pago, correções/dia, custo LLM/dia, latência média, taxa de falha | [`admin/analytics.md`](../admin/analytics.md) |
| Configuração de conteúdo | Prompt templates, planos (preço/limites), feature flags | [`admin/system-config.md`](../admin/system-config.md) |
| Emails | Broadcast emails, history | [`admin/system-config.md`](../admin/system-config.md) |
| Sistema | API logs, health checks, environment info, feature toggles | [`admin/system-config.md`](../admin/system-config.md) |

---

## Acesso / RBAC

- Coluna `users.role` (`user` vs `admin`) controla acesso (ver [schema.md](../data/schema.md))
- Admin separado: `/admin/login` (cookie isolado). Ver [auth-and-authorization.md](../api/auth-and-authorization.md)
- API: dependência `get_current_admin` valida `role='admin'` e `status='active'`
- RLS em todas tabelas tem policies admin-all (escrita sometimes) + user-self (leitura)

---

## Audit log

Toda mutação admin é logada em **`admin_audit_log`** (imutável — sem update/delete). Ver [`admin/overview.md`](../admin/overview.md).

Campos: `admin_id, action, target_type, target_id, payload (jsonb), ip_address, user_agent, created_at`.

---

## Diferença vs dashboard do aluno

| Aspecto | Dashboard aluno | Admin panel |
|---|---|---|
| URL | `app.redana.com.br/{dashboard,redacoes,...}` | `admin.redana.com.br/*` (app dedicado) |
| Auth | `user.role = 'user'` | `user.role = 'admin'` |
| Dados acessíveis | Próprios | Todos (RLS admin bypass) |
| Mutations | Em própria conta | Em qualquer usuário / sistema |
| Logging | events table | events table + admin_audit_log |
| Audit | — | Imutável, sempre registrado |

---

## API

Rotas admin: `/api/v1/admin/*`. Ver [contracts.md § Admin](../api/contracts.md).

Cada endpoint admin:
1. Validar `role='admin'` (RLS garante dual-layer)
2. Executar mutation
3. Comparison antes/depois (`payload` no audit log)
4. Retornar `audit_id`

---

## Primeiros passos para construir (resumo)

1. Migrations `0002` → `add_role_to_users` + `admin_audit_log` (ver [migrations.md](../data/migrations.md))
2. Migrations `0004`, `0005`, `0006` → `prompt_templates`, `feature_flags`, `plans_config`
3. FastAPI: `apps/api/app/admin/` module + routers + dependencies + audit decorator
4. Next.js admin: scaffolding `apps/admin` (app dedicado em `admin.redana.com.br`, Vercel) + middleware de role + componentes
5. Testes E2E: login admin → suspend user → reprocess essay → grant free corrections → broadcast email

→ Plano em fases: [phasing.md Fase 5](../process/phasing.md).

---

## Documentação detalhada

- [`admin/overview.md`](../admin/overview.md) — O que é, RBAC, audit log
- [`admin/users-and-essays.md`](../admin/users-and-essays.md) — Gestão de usuários e redações
- [`admin/corrections-admin.md`](../admin/corrections-admin.md) — Correção humana
- [`admin/subscriptions-admin.md`](../admin/subscriptions-admin.md) — Assinaturas, grants, reembolsos
- [`admin/analytics.md`](../admin/analytics.md) — Métricas e charts
- [`admin/payments-and-webhooks.md`](../admin/payments-and-webhooks.md) — AbacatePay admin view
- [`admin/system-config.md`](../admin/system-config.md) — Prompts, flags, pricing, emails broadcast, logs