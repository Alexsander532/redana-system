# Inventário de componentes UI

> Documento NOVO (v2.0). Componentes principais do dashboard e do admin, agrupados por função.

Stack de UI: **Tailwind CSS v4** + **shadcn/ui** (parcial) + **Recharts** (charts) + **@tanstack/react-table** (tabelas admin).

Design tokens ficam em `packages/shared/src/tokens.ts` para alinhar com a landing Astro.

---

## Layout

| Componente | Local | Função |
|---|---|---|
| `Sidebar.tsx` | `apps/web/src/components/layout/` | Navegação do dashboard do aluno |
| `Topbar.tsx` | `.../layout/` | Topbar com user menu, notificações |
| `UserMenu.tsx` | `.../layout/` | Dropdown com Perfil / Sair / Exportar dados |
| `AdminSidebar.tsx` | `.../admin/` | Navegação do painel admin (routes `/admin/*`) |
| `AdminTopbar.tsx` | `.../admin/` | Topbar admin (Pesquisa global, user count, MRR quick, UserMenu) |

---

## Auth

| Componente | Função |
|---|---|
| `LoginForm.tsx` | Email+Senha + `GoogleButton` (Supabase OAuth). Server Action `signin`. |
| `SignupForm.tsx` | Email+Senha+name. Chama `POST /auth/signup`. |
| `GoogleButton.tsx` | SignInWithOAuth('google') via `supabase-js`. |
| `AdminLoginForm.tsx` | Login admin extra `/admin/login`. |

---

## Redação (envio)

| Componente | Função |
|---|---|
| `EssayForm.tsx` | Textarea (150-1500 palavras) ou upload `.txt`/`.docx`. Botão "Corrigir agora" desabilita quando `corrections_used >= max_corrections`. |
| `FreeTierCounter.tsx` | Barra de progresso "3/5 correções gratuitas". |
| `SubmissionProgress.tsx` | Skeleton + steps "Analisando / Avaliando competências / Gerando feedback". SSE/polling. |
| `UpgradeBanner.tsx` | CTA "Esgotou suas 5 correções grátis. Assine Anual e economize 30%." |

---

## Correção (visualização)

| Componente | Função |
|---|---|
| `ScoreHeader.tsx` | Big badge 0-1000 com cor gradiente por faixa. |
| `CompetencyBars.tsx` | Barras C1-C5 (mesmo design da landing). 0-200 cada. Cores semânticas. |
| `CompetencyFeedback.tsx` | Accordion por competência com o feedback textual. |
| `NextSteps.tsx` | Cardsinhos action+priority (high/medium/baixa). |
| `OverallScoreBadge.tsx` | Badge compacto para histórico. |

---

## Dashboard

| Componente | Função |
|---|---|
| `EvolutionChart.tsx` | Line chart Recharts com overall_score por redação ao longo do tempo. |
| `EssayHistory.tsx` | Lista `EssayRow[]` paginada. |
| `EssayRow.tsx` | Linha: título, data, score, status badge. |
| `PlanStatusCard.tsx` | Card "Plano Anual ativo — 12/30 correções usadas". |
| `UpgradeBanner.tsx` | Reaproveitado na home. |

---

## Planos

| Componente | Função |
|---|---|
| `PricingCards.tsx` | 3 cards (Mensal/Trimestral/Anual) com badges "15% off / 30% off". Reaproveitado da landing. |
| `CurrentPlanBadge.tsx` | Badge "Anual — Renovação 12/08/2026". |

---

## UI primitives (ui/)

shadcn-style: `Button`, `Card`, `Input`, `Label`, `Textarea`, `Modal/Dialog`, `Accordion`, `Badge`, `Toast`, `Tabs`, `Dropdown`, `Skeleton`, `Tooltip`, `Switch`, `Select`, `Pagination`.

---

## Admin (NOVO v2.0)

| Componente | Função | Detalhe doc |
|---|---|---|
| `UserTable.tsx` | Tabela paginada de users (`@tanstack/react-table`) com search, status filter, bulk action | [admin/users-and-essays.md](../../admin/users-and-essays.md) |
| `EssayTable.tsx` | Tabela paginada de essays com filtro por status/user/data | [admin/users-and-essays.md](../../admin/users-and-essays.md) |
| `SubscriptionTable.tsx` | Tabela de subscriptions + ações inline (suspend/grant/refund) | [admin/subscriptions-admin.md](../../admin/subscriptions-admin.md) |
| `WebhookEventTable.tsx` | Log de webhooks com replay button | [admin/payments-and-webhooks.md](../../admin/payments-and-webhooks.md) |
| `PromptTemplateEditor.tsx` | Editor Monaco + placeholders `{{essay_text}}`, `{{rubric}}` | [admin/system-config.md](../../admin/system-config.md) |
| `FeatureFlagToggle.tsx` | Switch list com busca + override per-user | [admin/system-config.md](../../admin/system-config.md) |
| `PlanEditor.tsx` | Form para editar `plans_config` (price, max_corrections, cycle_days) | [admin/system-config.md](../../admin/system-config.md) |
| `BroadcastEmailForm.tsx` | Form de email broadcast: segmento + template + preview + schedule | [admin/system-config.md](../../admin/system-config.md) |
| `AuditLogTable.tsx` | Tabela `admin_audit_log` com filtros avançados | [admin/overview.md](../../admin/overview.md) |
| `MetricCard.tsx` | KPI card (número + delta % + sparkline) | [admin/analytics.md](../../admin/analytics.md) |
| `MrrChart.tsx` | Area chart MRR 30d/90d | [admin/analytics.md](../../admin/analytics.md) |
| `LlmCostChart.tsx` | Bar chart custo LLM/dia | [admin/analytics.md](../../admin/analytics.md) |
| `BreakdownChart.tsx` | Pie/Bar por plano/competência | [admin/analytics.md](../../admin/analytics.md) |
| `HumanCorrectionForm.tsx` | Form com 5 inputs (0-200) para C1-C5 + textareas feedback + preview overall | [admin/corrections-admin.md](../../admin/corrections-admin.md) |

---

## Storybook / docs de uso (recomendado pós-MVP)

Para Fase 4: configurar Storybook em `apps/web` com stories para `CompetencyBars`, `ScoreHeader`, `PricingCards`, `EvolutionChart` — permitindo testar visualmente estados (empty/error/load) isoladamente. Não é bloqueador para o MVP.