# Riscos, perguntas abertas e próximos passos

> Seção original: PLAN.md §17 (riscos) + §18 (perguntas em aberto + próximos passos imediatos).

---

## Riscos

| Risco | Severidade | Mitigação |
|---|---|---|
| **LLM inconsistente** — JSON inválido, notas fora da rubrica | Alto | Tool use (JSON forçado), validação server-side, retry 1x com prompt de correção. Trigger DB `validate_overall_score`. |
| **Custo LLM > margem no plano Anual** (R$6,90/mês com 30 corr) | Alto | Cap em 30 correções/mês. Custo médio R$0,03 × 30 = R$0,90/mês vs receita R$6,90 → 87% margem. Monitorar via [admin/analytics.md](../../admin/analytics.md). |
| **AbacatePay amadurecimento** — gateway mais novo que Stripe | Médio | Limitar lógica crítica de cobrança a `abacatepay_service.py` para troca futura. Cobertura de testes no webhook. |
| **Webhook perdido** → estado inconsistente | Médio | Job diário de reconciliação + monitorar `subscription.cancelled`. Drift visible admin. |
| **Trimestral não-cycle nativo** | Médio | Usar `cycle: MONTHLY` + metadata `plan=quarterly` no DB; reforça a regra via `period_end`. Ver [subscriptions.md](../features/subscriptions.md). |
| **LGPD não conformidade** | Alto | Política clara, exportar/excluir, retenção 12m, DPO email. Ver [lgpd-compliance.md](../features/lgpd-compliance.md). |
| **Ataques ao endpoint webhook** | Médio | HMAC verify, ratelimit, idempotência por `event_id`. |
| **Custo do Supabase Free tier** | Baixo | Pro tier $25/mês ao passar 500MB DB ou 50k MAU. |
| **Admin panel misuse** (v2.0) | Médio | Audit log imutável, RBAC, impersonation com curta duração + log, confirmação modals em ação destrutiva. |
| **Vazamento de PII via Sentry/logs** (v2.0) | Alto | `before_send` filter strips `essay.content`, tokens, passwords. Tabela `events` não loga content. |
| **Drift entre `schema.sql` e Alembic** (v2.0) | Médio | Workflow: mesma mudança em ambos (ver [migrations.md](../data/migrations.md)). CI pode validar com `alembic check`. |

---

## Perguntas abertas

1. **Trimestral via AbacatePay**: melhor como `cycle: MONTHLY` com `period_end: +90d` OU criar 3 checkouts `ONE_TIME`?
   - **Decisão atual (v1.1):** `cycle: MONTHLY` + metadata `plan=quarterly`. Confirmar.
2. **Reembolso parcial**: política "7 dias para cancelar e reembolsar integral" — confirmar legalmente.
   - AbacatePay só permite refund integral. Limitação documentada.
3. **Detecção de tema**: no MVP, repassamos o tema como texto junto da redação, ou deixamos o LLM inferir?
   - Recomendação v2.0: deixar LLM inferir (texto tem o tema naturalmente nas redações do ENEM).
4. **Front-end do app em dark/light**: a landing é clara; o dashboard segue claro ou assume dark?
   - Recomendação v2.0: claro, consistente com landing.
5. **Limites do Free tier**: 5 correções vitalícias OU 5 por mês? (Atualmente vitalício para reduzir abuso).
   - Recomendação: manter vitalício.
6. **Exportação PDF do resultado**: incluir link de download no email "correção-pronta"?
   - Pendente: só pós-MVP.
7. **Admin panel: app dedicado ou rota dentro de `apps/web`?** (v2.0)
   - Documentação atual assume rota `/admin/*` dentro do `apps/web`. App dedicado reduziria risco de bundle/admin auth contaminação. Pedir confirmação ao usuário.
8. **Como propagar `users.role` à sessão Next.js?** (v2.0)
   - (a) Consultar `users` em `getSession()` (extra query por request); **(atual)**<br/>
   (b) Custom claim Supabase via trigger `on_auth_user_created` para evitar query extra.
9. **Impersonation expiry** (v2.0): 15 minuots parece bom? Token rotulado `impersonating:true`?
10. **Audit log retention**: manter indefinidamente (imutável) OU automatizar archiving após 2 anos?
    - Recomendação: arquivar para S3/Storage_after 2 anos, manter summary no DB.

---

## Próximos passos imediatos (após aprovação do plano)

1. **Aprovar plano** com o time/usuário (v2.0 — especialmente mudança do Admin Panel para Fase 5 legítima)
2. **Confirmar perguntas abertas #7, #8, #9** (admin app vs rota, custom claim, impersonation)
3. **Sprint 1 (Fase 0)**: criar pasta `redana`, iniciar monorepo, migrar landing
4. **Sprint 2 (Fase 1)**: Supabase project + auth + shell Next.js
5. **Sprint 3 (Fase 2)**: primeira correção E2E funcionando
6. **Sprint 4 (Fase 3)**: AbacatePay checkout + webhook
7. **Sprint 5 (Fase 4)**: polimento + deploy produção → **MVP core em produção**
8. **Sprint 6 (Fase 5)**: Admin Panel + correção humana → MVP completo v2.0
9. **Sprint 7+ (Fase 6, pós-MVP)**: MercadoPago + PIX avulso

---

## Decisões pendentes — confirmar com o usuário

| Item | Default na doc | Alternativa |
|---|---|---|
| Admin app dedicado vs rota em `apps/web` | Rota `/admin/*` em `apps/web` | App dedicado `apps/admin` |
| Coluna `users.role` para RBAC | `users.role text check ('user','admin')` | Tabela `roles` separada (overkill p/ MVP) |
| Audit log retention | Imutável indefinidamente | Arquivamento pós 2 anos |
| Logo de prompt templates | Tabela `prompt_templates` (ativo único) | Variável env (pós-MVP simplificada) |
| Feature flags | Tabela `feature_flags` + overrides por user | Env vars (sem runtime toggle) |