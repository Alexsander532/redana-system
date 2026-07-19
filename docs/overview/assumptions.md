# Premissas

> Seção original: PLAN.md §2. Decisões técnicas e de produto tomadas como ponto de partida, com a razão de cada uma.

As premissas abaixo são assumidas como verdades fundamentais do projeto. Mudar qualquer uma delas implica revisão arquitetural relevante.

---

| Premissa | Razão |
|---|---|
| **Monorepo com Turborepo + pnpm** | Landing + Dashboard + API compartilham tipos, design tokens e catálogo de planos. Evita drift entre apps. |
| **Subdomínios dedicados** | `redana.com.br` (landing), `app.redana.com.br` (dashboard), `api.redana.com.br` (FastAPI). Separa cache/CDN, SEO e preocupações de deploy. |
| **Landing em Astro (existente)** | Já no ar, nota alta em performance. Mantém no monorepo como `apps/landing`. |
| **Dashboard em Next.js 14 (App Router)** | SSR + RSC para busca de dados, middleware de auth, deploy no Vercel, ecossistema React. |
| **API em Python FastAPI** | Orquestração de LLM, prompts estruturados, ML libs — Python é o padrão para IA/LLM. |
| **Supabase Auth + Postgres** | Tudo-em-um gerenciado: auth (email + Google), DB, RLS, Storage, Real-time. Free tier generoso. |
| **LLM principal: Anthropic Claude (3.5 Sonnet+)** | Excelente em seguir rubrica estruturada, contexto longo, saída JSON. |
| **LLM fallback: OpenAI GPT-4o** | Redundância, disponibilidade. |
| **Pagamentos AbacatePay** | Gateway brasileiro, suporta Assinaturas (Mensal/Anual), Cartão e PIX. Webhooks HMAC. Mais simples que Stripe/MercadoPago para cobrança recorrente BR. Ver [tech-stack.md](../architecture/tech-stack.md). |
| **Sem "unlimited" ilimitado** | Planos pagos têm limite de correções por ciclo (30) para controlar custo de LLM — comunicado como "até 30 correções/mês". |
| **Política de retenção: 12 meses** | Escrito da redação mantido 12 meses, depois anonimizado (LGPD). Ver [lgpd-compliance.md](../features/lgpd-compliance.md). |
| **Admin é papel de primeira classe** |.usuario admin (`role = 'admin'`) acessa painel dedicado. Não é mais "pós-MVP deferrido". Ver [`admin/`](../admin/overview.md). |
| **Idioma do produto** | PT-BR em todo conteúdo (UI, emails, prompts, feedback). |
| **Dark/light** | A landing é clara; o dashboard segue claro (decisão pendente — ver [risks-and-open-questions.md](../process/risks-and-open-questions.md) #4). |

---

## Implicações das premissas

| Premissa | Implica em |
|---|---|
| Monorepo | Um único `pnpm install` roda tudo; CI tem pipeline paralelo por app. |
| Subdomínios | CORS na API: só `app.redana.com.br` e `redana.com.br`. |
| Supabase Auth | JWT verificado server-side (PKCE); RLS é segunda camada de defesa. |
| Python FastAPI | Linguagem de backend única; SDKs LLM e pagamentos têm suporte Python oficial. |
| AbacatePay | Sem customer portal nativo → tela "Meu plano" interna. Ver [payments-abacatepay.md](../features/payments-abacatepay.md). |
| Sem unlimited | Logic de rate limit nos serviços: `corrections_used < max_corrections`. |
| Admin RBAC | Nova coluna `users.role` + tabela `admin_audit_log`. Ver [schema.md](../data/schema.md). |

---

## Premissas que precisam confirmação

Ver [risks-and-open-questions.md](../process/risks-and-open-questions.md) para os itens ainda em aberto (trimestral como `cycle: MONTHLY` vs `ONE_TIME`, reembolso parcial, etc.).