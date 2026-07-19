# Redana — Plano do Sistema (v2.0)

> **SaaS de correção de redação do ENEM** com feedback por competência (C1-C5), assinaturas via AbacatePay e **painel administrativo completo**.

> **Versão:** 2.0 — Reestruturação da documentação + Admin Panel como fase de primeira classe
> **Data:** Julho de 2026
> **Produto:** Redana (anteriormente NotaCerta)
> **Landing no ar:** https://redana-alexsander532s-projects.vercel.app
> **GitHub:** https://github.com/Alexsander532/redana
> **Base do projeto (monorepo):** `C:\Users\Alexsander\Desktop\redana`

Este arquivo é o **ponto de entrada** da documentação. O conteúdo detalhado está organizado em subpastas temáticas abaixo.

> **Nota sobre histórico:** O documento monolítico `PLAN.md` (v1.1, 910 linhas) foi preservado como [`archive-PLAN-v1.1.md`](./archive-PLAN-v1.1.md) para referência. Ele está **superseded** por esta estrutura. Há também um documento prévio detalhado do Admin Panel mantido como referência: [`admin-architecture-addendum.md`](./admin-architecture-addendum.md) (v1.0, 1318 linhas) — seus detalhes foram consolidados/estendidos na subpasta [`admin/`](./admin/overview.md).

---

## Resumo do produto

Redana é uma plataforma brasileira onde estudantes enviam redações do ENEM e recebem, em minutos, uma correção detalhada organizada pelas **5 competências oficiais** (C1 a C5): nota estimada por competência, feedback qualitativo e um plano de melhoria para a próxima redação. O sistema tem três públicos:

- **Estudantes** — usam o dashboard (`app.redana.com.br`) para enviar redações, ver correções, acompanhar evolução e assinar planos.
- **Administrador (owner)** — usa o painel admin para controlar usuários, redações, correções humanas, assinaturas, pagamentos, métricas, conteúdo e sistema. (Ver [`admin/`](./admin/overview.md).)
- **Visitantes** — acessam a landing (`redana.com.br`) para conhecer o produto e se cadastrar.

Stack: **Astro** (landing) + **Next.js 14** (dashboard) + **FastAPI** (API) + **Supabase** (auth+DB) + **Claude 3.5 Sonnet** (LLM) + **AbacatePay** (pagamentos) + **Resend** (email) + **Turborepo+pnpm** (monorepo).

---

## Decisões pendentes — confirmar com o usuário ANTES de construir

| # | Decisão | Default na doc | Alternativa / observação |
|---|---|---|---|
| 1 | Onde o admin vive | Rotas `/admin/*` dentro de `apps/web` | **Pré-recomendado pelo addendum (v1.0): app Next.js separado `apps/admin` em `admin.redana.com.br`**. Ver [arch/dec](./admin/overview.md) § "Onde o admin vive". |
| 2 | Coluna RBAC | `users.role text ('user','admin')` | Tabela `roles` separada (overkill p/ MVP). |
| 3 | Audit log retention | Imutável indefinidamente | Arquivamento pós 2 anos. |
| 4 | Propagar role à sessão Next.js | Consultar `users` em `getSession()` | Custom claim Supabase via trigger. |
| 5 | Impersonation | Token 15 min, `impersonating:true` | Pendente confirmação de seguro Supabase. |

Ver [process/risks-and-open-questions.md](./process/risks-and-open-questions.md) para a lista completa.

---

## Mapa da documentação

### `overview/`
- [vision-and-goals.md](./overview/vision-and-goals.md) — Visão geral, metas do MVP, fora de escopo e glossário.
- [assumptions.md](./overview/assumptions.md) — Premissas tecnológicas e de produto que guiam as decisões.
- [scope.md](./overview/scope.md) — Escopo do MVP (dentro/fora) em checklist.

### `architecture/`
- [system-diagram.md](./architecture/system-diagram.md) — Diagrama do sistema e responsabilidade de cada app.
- [monorepo-structure.md](./architecture/monorepo-structure.md) — Árvore completa do monorepo (apps + packages).
- [tech-stack.md](./architecture/tech-stack.md) — Stack e decisões técnicas (incl. rationale AbacatePay + limitações).
- [observability.md](./architecture/observability.md) — Observabilidade e métricas-chave.

### `data/`
- [domain-model.md](./data/domain-model.md) — Entidades principais e diagrama ER.
- [schema.md](./data/schema.md) — SQL do schema, RLS, invariáveis e adições do admin (`users.role`, `admin_audit_log`, `feature_flags`).
- [migrations.md](./data/migrations.md) — Workflow de migrations (Alembic + Supabase).

### `api/`
- [contracts.md](./api/contracts.md) — Contratos de API (rotas do aluno + rotas do admin).
- [auth-and-authorization.md](./api/auth-and-authorization.md) — Fluxo de auth, middleware, RBAC (incl. role admin).
- [rate-limiting.md](./api/rate-limiting.md) — Regras de rate limit.
- [webhooks-abacatepay.md](./api/webhooks-abacatepay.md) — Handlers de webhook AbacatePay em profundidade.

### `frontend/`
- [dashboard-routes.md](./frontend/dashboard-routes.md) — Rotas Next.js e telas-chave do dashboard do aluno.
- [ui-components.md](./frontend/ui-components.md) — Inventário de componentes principais.
- [landing.md](./frontend/landing.md) — O app de landing (Astro): conteúdo e deploy.

### `features/`
- [correction-engine.md](./features/correction-engine.md) — Engine de correção: fluxo, JSON schema, prompt, custo, latência.
- [essay-submission.md](./features/essay-submission.md) — Fluxo de submissão de redação (texto/upload), limites, parser.
- [payments-abacatepay.md](./features/payments-abacatepay.md) — Integração AbacatePay (setup, upgrade, env vars, tela "meu plano").
- [subscriptions.md](./features/subscriptions.md) — Modelo de planos, limites, reconciliação, change/cancel.
- [emails.md](./features/emails.md) — Integração Resend, templates, pontos de gatilho.
- [lgpd-compliance.md](./features/lgpd-compliance.md) — Segurança + LGPD combinados.
- [admin-panel.md](./features/admin-panel.md) — Visão geral do Admin Panel (entrada para os docs detalhados em `admin/`).

### `admin/` — Documentação detalhada do Admin Panel
- [overview.md](./admin/overview.md) — O que é o admin panel, quem usa, RBAC, audit log.
- [users-and-essays.md](./admin/users-and-essays.md) — Gestão de usuários e redações pelo admin.
- [corrections-admin.md](./admin/corrections-admin.md) — Modo de correção humana (manual override).
- [subscriptions-admin.md](./admin/subscriptions-admin.md) — Gestão de assinaturas, grants manuais, reembolsos.
- [analytics.md](./admin/analytics.md) — Dashboard de métricas, charts, MRR, custo LLM.
- [payments-and-webhooks.md](./admin/payments-and-webhooks.md) — Visão admin do AbacatePay, log de eventos, reconciliação.
- [system-config.md](./admin/system-config.md) — Prompt templates, pricing config, feature flags, emails broadcast, logs.

### `process/`
- [phasing.md](./process/phasing.md) — Plano de entrega por fases (incl. **Admin Panel como Fase 5 legítima**).
- [definition-of-done.md](./process/definition-of-done.md) — Definition of Done (checklist).
- [risks-and-open-questions.md](./process/risks-and-open-questions.md) — Riscos, perguntas em aberto, próximos passos.

### Arquivos de referência (raiz de `docs/`)
- [archive-PLAN-v1.1.md](./archive-PLAN-v1.1.md) — Arquivo histórico monolítico v1.1 (referência).
- [admin-architecture-addendum.md](./admin-architecture-addendum.md) — Adendo prévio (v1.0) com design profundo do Admin Panel; recommenda app separado `apps/admin`. Conteúdo consolidado em `admin/`, mantido como referência.

---

## Roadmap de fases (resumo)

| Fase | Nome | Duração estimada | Detalhe |
|---|---|---|---|
| Fase 0 | Scaffolding (monorepo, packages, landing) | 1 dia | [phasing.md](./process/phasing.md) |
| Fase 1 | Infra + Auth + Shell | 3-4 dias | [phasing.md](./process/phasing.md) |
| Fase 2 | Submissão + Correção IA | 5-6 dias | [phasing.md](./process/phasing.md) |
| Fase 3 | Pagamentos AbacatePay | 4-5 dias | [phasing.md](./process/phasing.md) |
| Fase 4 | Polimento + Integração | 3-4 dias | [phasing.md](./process/phasing.md) |
| **Fase 5** | **Admin Panel + Correção Humana** | **5-7 dias** | **[phasing.md](./process/phasing.md) + [admin/](./admin/overview.md)** |
| Fase 6 | MercadoPago + PIX avulso | 2-3 dias (pós-MVP) | [phasing.md](./process/phasing.md) |

**Total MVP (Fases 0-4): ~4-6 semanas.** Total com Admin (Fases 0-5): ~5-7 semanas.

> **Mudança da v1.1 → v2.0:** A Fase 5 (Admin Panel) deixou de ser "pós-MVP deferred" e passou a ser uma fase legítima, totalmente documentada. Ver [`admin/`](./admin/overview.md).

---

## Histórico de versões

| Versão | Data | Resumo das mudanças |
|---|---|---|
| v1.0 | Junho 2026 | Plano inicial (NotaCerta) — sem AbacatePay. |
| v1.1 | Julho 2026 | Revisão técnica: trocou Stripe por AbacatePay, adicionou trimestral, "meu plano" no lugar do customer portal, limites de custo LLM. Mantido como `archive-PLAN-v1.1.md`. |
| addendum v1.0 | Julho 2026 | Adendo de arquitetura do Admin Panel (preexistente), recommenda app separado `apps/admin`. Mantido como `admin-architecture-addendum.md`. |
| **v2.0** | **Julho 2026** | **Reestruturação: split do PLAN.md monolítico em `docs/` por tema. Admin Panel promovido a fase legitima (Fase 5) com documentação completa em `admin/`. Novos docs: migrations, rate-limiting, ui-components, landing, essay-submission, subscriptions, emails.** |

---

## Glossário

| Termo | Significado |
|---|---|
| **ENEM** | Exame Nacional do Ensino Médio — vestibular brasileiro anual. |
| **Competências C1-C5** | As 5 competências oficiais avaliadas na redação do ENEM pelo INEP. Cada uma vale 0-200 pontos; soma = 0-1000. |
| **Rubrica** | Critérios objetivos de avaliação de uma redação (a rubrica oficial do INEP para C1-C5). |
| **Correção** | Resultado da avaliação de uma redação: notas C1-C5, feedback por competência, próximos passos. |
| **Free tier** | Plano gratuito com 5 correções vitalícios (não reseta). |
| **Plano pago** | Mensal R$ 9,90 / Trimestral R$ 25,20 / Anual R$ 82,80. Até 30 correções por ciclo. |
| **Ciclo** | Período de cobrança: mensal (30d), trimestral (90d), anual (365d). |
| **Admin** | Usuário com `role = 'admin'` que acessa o painel administrativo. |
| **Correção humana** | Correção manual feita pelo admin que sobrescreve a IA (`corrected_by = 'human'`). |
| **Reconciliação** | Job diário que sincroniza estado de assinaturas com a API do AbacatePay. |
| **RBAC** | Role-Based Access Control — modelo de autorização por papel (user vs admin). |
| **Audit log** | Registro imutável de todas as ações administrativas (tabela `admin_audit_log`). |
| **Feature flag** | Toggle de configuração que liga/desliga funcionalidades em runtime. |
| **MRR** | Monthly Recurring Revenue — receita recorrente mensal. |
| **LGPD** | Lei Geral de Proteção de Dados (Lei 13.709/2018, Brasil). |
| **DPO** | Encarregado de Proteção de Dados (Data Protection Officer). |
| **PKCE** | Proof Key for Code Exchange — extensão do OAuth para SSR seguro. |
| **SSE** | Server-Sent Events — streaming de progresso usado na correção. |
| **RLS** | Row Level Security do Postgres/Supabase — isolação de dados por usuário. |

---

**Notas de uso:** Navegue pelos links acima conforme o tema. Cada subdoc é standalone — pode ser lido isoladamente. Diagramas Mermaid são suportados no GitHub.