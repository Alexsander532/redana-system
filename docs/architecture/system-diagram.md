# Diagrama do sistema e responsabilidades

> Seção original: PLAN.md §4 (4.1 diagrama + 4.2 responsabilidades).

---

## 4.1 Diagrama do sistema

```mermaid
flowchart LR
  subgraph Frontend
    L[Landing<br/>Astro<br/>redana.com.br]
    W[Dashboard<br/>Next.js<br/>app.redana.com.br]
  end
  subgraph Backend
    A[API FastAPI<br/>api.redana.com.br]
    Q[Fila de correção<br/>Background Tasks]
  end
  subgraph Servicos
    S[(Supabase<br/>Auth+DB+Storage)]
    AB[AbacatePay<br/>Assinaturas+Pix+Card]
    LLM[Claude / GPT-4o]
    R[Resend<br/>Emails]
  end

  U([Usuário]) -->|visita| L
  L -->|CTA| W
  U -->|usa| W
  W -->|REST| A
  A -->|SQL| S
  A -->|enqueue| Q
  Q -->|prompt| LLM
  A -->|assinatura| AB
  AB -->|webhook| A
  A -->|transactional| R
  R --> U
```

### Diagrama estendido (com Admin Panel — v2.0)

```mermaid
flowchart LR
  subgraph Frontend
    L[Landing<br/>Astro]
    W[Dashboard Next.js<br/>app.redana.com.br<br/>inclui /admin/*]
  end
  subgraph Backend
    A[API FastAPI]
    Q[Fila de correção<br/>Background Tasks]
  end
  subgraph Servicos
    S[(Supabase<br/>Auth+DB+Storage)]
    AB[AbacatePay]
    LLM[Claude / GPT-4o]
    R[Resend]
  end

  U([Usuário]) -->|visita| L
  L -->|CTA| W
  U -->|usa| W
  W -->|REST| A
  A -->|SQL| S
  A -->|enqueue| Q
  Q -->|prompt| LLM
  A -->|assinatura| AB
  AB -->|webhook| A
  A -->|transactional| R
  R --> U

  ADM([Admin<br/>role=admin]) -->|usa| W
  W -->|admin REST| A
  A -->|audit log| S
```

> **Decisão pendente (confirmação do usuário):** o admin live como rotas `/admin/*` dentro do mesmo app Next.js (`apps/web`), **ou** em app dedicado `apps/admin`. Documentação atual assume rota dentro do `apps/web` por simplicidade. Ver [admin/overview.md](../../admin/overview.md) § "Onde o admin vive".

---

## 4.2 Responsabilidade de cada app

| App | Função |
|---|---|
| `apps/landing` (Astro) | Marketing estático. Botões CTA → `app.redana.com.br/signup`. Ver [landing.md](../frontend/landing.md). |
| `apps/web` (Next.js 14) | Dashboard do aluno: auth, envio, visualização de correções, planos. **Inclui rotas `/admin/*`** do painel admin. Ver [dashboard-routes.md](../frontend/dashboard-routes.md). |
| `apps/api` (FastAPI) | Toda lógica: auth JWT verify, essays, corrections, subscriptions, abacatepay webhooks, **+ routers admin (`/admin/*`)**. Ver [contracts.md](../api/contracts.md). |
| `packages/shared` | Tipos TS, constantes (competências, planos), design tokens. |
| `packages/db` | `schema.sql`, migrações Supabase, seed. Ver [schema.md](../data/schema.md). |
| `packages/email` | Templates React Email (welcome, correção-pronta, etc). Ver [emails.md](../features/emails.md). |

---

## Fronteiras e responsabilidades transversais

| Responsabilidade | Dono | Detalhe |
|---|---|---|
| Auth (JWT verify) | FastAPI (via Supabase JWKS) | Web repassa JWT; API valida. RLS é backup. |
| RLS no DB | Supabase / `packages/db` | Toda tabela com `user_id` tem RLS. |
| Rate limiting | FastAPI (slowapi) | [rate-limiting.md](../api/rate-limiting.md). |
| Webhook HMAC verify | FastAPI | [webhooks-abacatepay.md](../api/webhooks-abacatepay.md). |
| Enfileiramento de correção | FastAPI BackgroundTasks | Migração para Celery+Redis é pós-MVP. |
| Audit log de admin | FastAPI + Postgres | Tabela `admin_audit_log`. Ver [admin/overview.md](../../admin/overview.md). |