# Stack e decisões técnicas

> Seção original: PLAN.md §6. Decisões de stack e rationale AbacatePay + limitações.

---

## Stack

| Camada | Tecnologia | Por quê |
|---|---|---|
| Landing | Astro 5 | Já em produção, zero-JS, Lighthouse 95+. |
| Dashboard | Next.js 14 (App Router) | SSR/RSC, middleware auth, deploy Vercel nativo. |
| Estilos | Tailwind CSS v4 + design tokens (`packages/shared`) | Utility-first, alinha chips visualmente com a landing. |
| API | Python FastAPI 0.110+ | Async, Pydantic v2, OpenAPI automático. |
| ORM | SQLAlchemy 2.0 (async) + Alembic | ORM maduro, migrations versionadas. Ver [migrations.md](../data/migrations.md). |
| Auth | Supabase Auth (PKCE) | Email + Google, JWT, RLS. |
| DB | Supabase PostgreSQL | Postgres gerenciado, RLS, real-time, Storage. |
| LLM | Anthropic Claude 3.5 Sonnet (primário) + GPT-4o (fallback) | Rubrica estruturada em PT-BR. |
| **Pagamentos** | **AbacatePay** | Gateway brasileiro, assinaturas (cycle MONTHLY/ANNUALLY), PIX + Cartão, webhooks HMAC, valores em centavos BRL. |
| Email | Resend + React Email | Templates em React, entrega rápida. |
| Monorepo | Turborepo + pnpm | Builds paralelos, cache, workspaces. |
| Deploy Front | Vercel | Front já lá; Next.js nativo. |
| Deploy API | Railway | Dockerfile simples, persistência opcional. |
| Observabilidade | Logflare (Supabase) + Sentry | Logs + erros. Ver [observability.md](./observability.md). |
| Charts (admin) | Recharts (Next.js) | Mesmo lib do `EvolutionChart` do dashboard do aluno. |
| Admin UI (tabelas) | `@tanstack/react-table` + shadcn/ui | Filtros/paginação/sort prontos. |

---

## Por que AbacatePay ao invés de Stripe

- **Foco em Brasil**: PIX + Cartão + Boleto nativos. Sem taxa de cross-border.
- **Assinaturas via produto com `cycle`**: `MONTHLY` e `ANNUALLY` suportados. Trimestral pode ser cobrado como três checkouts `ONE_TIME` ou um produto avulso.
- **Transparent Checkout opcional**: Embutido PIX no app (gera `brCode` + QR PNG na hora).
- **Webhooks HMAC**: Verificação de assinatura confiável. Ver [webhooks-abacatepay.md](../api/webhooks-abacatepay.md).
- **SDKs**: Oficial para Node.js e Python → integra direto no backend.
- **Baixa fricção de setup**: Sem processo de underwriting tão pesado quanto Stripe Brasil.

---

## Limitações conhecidas da AbacatePay

| Limitação | Impacto | Mitigação |
|---|---|---|
| **Sem `SEMIANNUALLY` para trimestral** — só WEEKLY/MONTHLY/SEMIANNUALLY/ANNUALLY | Trimestral não encaixa direto no `cycle` | **Decisão:** Trimestral vira checkout recorrente com `cycle: MONTHLY` + metadata `plan=quarterly` no DB; `period_end = now() + 90d` controlado pela API. Ver [subscriptions.md](../features/subscriptions.md). |
| **Sem Customer Portal pronto** | Usuário não auto-gerencia cartão | Implementar tela "Meu plano" no dashboard com botão Cancelar via API `POST /subscriptions/cancel`. Mudança de cartão → gerar novo checkout. Ver [payments-abacatepay.md](../features/payments-abacatepay.md). |
| **Charge automatic retry?** | Falta clareza na doc | Monitorar webhook `subscription.cancelled` para degradar plano. Recuperação via email de "atualize seu cartão" (ver [emails.md](../features/emails.md)). |
| **Reembolso só integral** | Sem partial refund | Política: 7 dias para cancelar integral dentro do trial. Admin tem trigger manual de refund integral (ver [admin/subscriptions-admin.md](../../admin/subscriptions-admin.md)). |

---

## Decisões complementares (v2.0)

| Decisão | Razão |
|---|---|
| Admin dentro do app Next.js (`/admin/*`) em vez de app dedicado | Reduz duplicação de shell, lib e hooks. Ainda há "app dedicado" como alternativa pendente de confirmação do usuário. |
| `@tanstack/react-table` para tabelas admin | Sort/filtro/paginação server-side prontos; mantém_stack unificada. |
| Feature flags em tabela (`feature_flags`) em vez de env vars | Toggle em runtime pelo admin, sem redeploy. Ver [admin/system-config.md](../../admin/system-config.md). |
| Prompt templates em tabela (`prompt_templates`) | Permite teste A/B de prompts sem redeploy. |
| Audit log em `admin_audit_log` (imutável) | Compliance + LGPD — todo admin action registrado. Ver [admin/overview.md](../../admin/overview.md). |

---

## Quando trocar / evoluir

| Componente | Gatilho de troca |
|---|---|
| AbacatePay | Custo > 5% ou múltiplas indisponibilidades/mês |
| BackgroundTasks → Celery+Redis | Latência de fila > 10s ou fila > 50 pendentes |
| Supabase Free → Pro | Passar 500MB DB ou 50k MAU |
| RLS → sem RLS (rewrites) | Nunca no MVP — RLS é defesa em profundidade |