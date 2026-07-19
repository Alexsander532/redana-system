# Dashboard — rotas e telas-chave

> Seção original: PLAN.md §9. Rotas Next.js e descrição das telas do dashboard do aluno. Rotas admin ver [admin/](../../admin/overview.md).

---

## Rotas Next.js (App Router) — aluno

```
/login                          → LoginForm (email + Google OAuth)
/signup                         → SignupForm
/recuperar                      → Solicitar reset
/redefinir                      → Nova senha (token query param)

/                               → redirect to /login ou /dashboard
/dashboard                      → Home: evolução + últimas redações + plano
/nova-redacao                   → EssayForm
/redacoes/[id]                  → Resultado da correção
/planos                         → PricingCards + upgrade
/conta                          → Perfil + plano + danger zone (LGPD)
/suporte                        → FAQ + contato
```

### Aplicação admin (v2.0) — `admin.redana.com.br`

O painel administrativo roda em um **app Next.js 14 separado** (`apps/admin`), domínio `admin.redana.com.br`, deploy Vercel. Rotas:

```
/login                    → Login admin (mesmo Supabase Auth, gate por role=admin)
/                         → Dashboard admin (KPIs, charts)
/usuarios                 → Lista de usuários
/usuarios/[id]            → Detalhe do usuário (essays, subscription, actions)
/redacoes                 → Todas as redações (filtro por status)
/redacoes/[id]            → Detalhe + correção humana
/assinaturas              → Lista de assinaturas
/assinaturas/[id]         → Editar assinatura (force status, grant, extend, refund)
/pagamentos               → AbacatePay + webhook event log
/metricas                 → Charts (MRR, conversão, custo LLM, latência)
/correcoes                → Fila de correção humana
/sistema                  → Prompt templates, feature flags, planos
/sistema/logs             → API logs + health
/sistema/audit            → Audit log
```

> Ver [`admin/`](../../admin/overview.md) para specs detalhadas de cada tela.

---

## Telas-chave (aluno)

### Dashboard home

Cards: plano atual + correções restantes, gráfico de evolução (Recharts), últimas 5 redações com preview de nota e status.

Componentes principais: `PlanStatusCard`, `EvolutionChart`, `EssayHistory`.

### Nova redação

- **Editor de texto** — min 150 palavras, max 1500
- **OU upload** `.txt`/`.docx`
- Botão "Corrigir agora" **desabilitado** se `corrections_used >= max_corrections` → mostra `UpgradeBanner`
- Ao submeter: TODO → `POST /essays` → redirect para `/redacoes/[id]?polling=true` mostrando `SubmissionProgress`

Componentes: `EssayForm`, `FreeTierCounter`, `SubmissionProgress`, `UpgradeBanner`.

### Resultado

- `ScoreHeader` (0-1000 badged) — same visual da landing
- `CompetencyBars` C1-C5 — accordion com `CompetencyFeedback`
- `NextSteps` — cards com area/action/priority
- Compartilhar / baixar PDF (pós-MVP)

### Planos

- Reaproveita `PricingCards` da landing (mesmo CSS)
- Clique → `POST /subscriptions/checkout` → AbacatePay redirect → `returnUrl`
- Logo após sucesso, webhook `subscription.completed` ativa o plano → tela `/planos?status=success` mostra `CurrentPlanBadge`

### Conta

- Avatar, nome, email (read-only, Supabase)
- Editar nome (retificação LGPD)
- Status da assinatura (`CurrentPlanBadge`)
- Botão cancelar → `POST /subscriptions/cancel`
- Exportar dados (LGPD) → `POST /account/export` → download JSON
- Excluir conta → modal confirmar → `POST /account/delete` (grace period 30 dias)

### Suporte

- FAQ accordion + email contato (DPO)

---

## UX states

| Estado | Comportamento |
|---|---|
| Loading | Skeletons (shimmer) no lugar de charts/tables |
| Empty | Ilustração + "Envie sua primeira redação" |
| Error | Error boundary + toast "Tente novamente" |
| Rate limited | Toast "Muitas redações em 1 hora, tente mais tarde" |
| Sem correções restantes | Redirect para `/planos` com banner |
| Plano expirou | Banner no topbar "Renove sua assinatura" |

---

## Considerações

| Item | Detalhe |
|---|---|
| Auth flow | Server Components; `cookies()` de sessão; `getSession()` no `lib/supabase/server.ts` |
| Data fetching | `fetch` para a API com cache `no-store` (dados sensíveis em tempo real) |
| Form actions | Server Actions onde possível; mutations via API REST quando envolve LLM |
| Streaming de status | SSE via `/essays/{id}/status` ou polling 3s em `/redacoes/[id]?polling=true` |
| Mobile | Sidebar vira bottom-nav, `EssayHistory` vira accordion stacked |
| Error page | `app/error.tsx` global + `not-found.tsx` |