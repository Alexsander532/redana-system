# Admin — Assinaturas

> Documento detalhado da gestão de assinaturas pelo admin: mudança de status, grants, extensões, refunds. Para o modelo de planos em si, ver [features/subscriptions.md](../features/subscriptions.md).

---

## Visão geral

Tela `/admin/assinaturas` lista todas as subscriptions do sistema. Operações:
- Forçar status (`active`/`past_due`/`canceled`/`expired`)
- Grant N correções avulsas
- Grant plano por X dias (sem AbacatePay)
- Estender `period_end`
- Disparar refund integral (AbacatePay + downgrade imediato)

Todas as mutações são auditadas (`admin_audit_log`).

---

## Tela `/admin/assinaturas`

`SubscriptionTable` paginada:

| Coluna | Conteúdo |
|---|---|
| User | Email (link para user detail) |
| Plano | `free`/`monthly`/`quarterly`/`annual` |
| Status | `active`/`past_due`/`canceled`/`expired` |
| Endpoint Abacate | `abacate_subscription_id` (truncated) |
| Usage | `12/30` |
| Period | `start - end` (format short) |
| Ações | Ver / Edit / ajustes rápidos menu (Grant corrections, Extend, Refund) |

### Filtros

- `status`
- `plan`
- `abacate_linked` (yes/no — `abacate_subscription_id` not null)
- date range em `created_at`

### API

```
GET /api/v1/admin/subscriptions?status=...&plan=...&limit=20&offset=0
```

---

## Tela `/admin/assinaturas/[id]`

### Seção 1: Cabeçalho
- User (link), plano, status badges
- abacate_subscription_id (link externo para painel AbacatePay se possível — Deep link)
- abacate_customer_id
- Corrections usadas/max
- Period start, period_end

### Seção 2: Endpoints rápidos (ações)

Cards de ação:
- Forçar status (combo + reason)
- Grant N correções
- Grant plano por X dias
- Estender period_end
- Disparar refund

Cada ação abre um modal com confirmação e campo `reason`.

### Seção 3: Webhook events relacionados

Se `abacate_subscription_id` setado: query `webhook_events WHERE raw_payload->>'subscription_id' = ?`. Mostra eventos recentes.

### Seção 4: Audit log sobre a subscription

`admin_audit_log WHERE target_type='subscription' AND target_id=uuid`.

---

## Ações — APIs

### Forçar status

```
POST /api/v1/admin/subscriptions/{id}/status
body: { "status": "active"|"past_due"|"canceled"|"expired", "reason": "texto" }
```

Efeito:
- `subscriptions.status = status`
- `subscriptions.updated_at = now`
- Se `status='canceled'` e `period_end > now()` mantém ativo até `period_end`
- Se `status='expired'` ou downgrade para free → `plan='free'`, `max_corrections=5`, `status='active'`
- NOT chama AbacatePay (é override local)
- Audit `subscription.status.force` with before/after

### Grant N correções

```
POST /api/v1/admin/subscriptions/{id}/grant-corrections
body: { "amount": 10, "reason": "Compensação por lentidão" }
```

Efeito:
- Se `max_corrections IS NULL`: noop (mantém unlimited) + audit nota
- Else `max_corrections += amount`
- Audit `subscription.grant.corrections`

### Grant plano por X dias

```
POST /api/v1/admin/subscriptions/{id}/grant-plan
body: { "plan": "monthly"|"quarterly"|"annual", "days": 30, "reason": "Cortesia" }
```

Efeito:
- `plan = plan`
- `status = 'active'`
- `max_corrections = 30`
- `period_start = now()`
- `period_end = now() + X days`
- `abacate_subscription_id = null` (admin grant não vem de AbacatePay — marcado)
- Job ao expirar mantém lógica de downgrade para free

Audit `subscription.grant.plan`.

### Estender period_end

```
POST /api/v1/admin/subscriptions/{id}/extend
body: { "days": 7, "reason": "Prorrogado por bug no app" }
```

Efeito:
- `period_end += days` (no timestamp atual)
- Audit `subscription.extend`

### Refund integral

```
POST /api/v1/admin/subscriptions/{id}/refund
body: { "reason": "Solicitado pelo user via email" }
```

Efeito:
1. Chama AbacatePay `POST /subscriptions/{abacate_id}/refund` (ou equivalente)
2. Se AbacatePay confirmar refund → `subscriptions.status = 'canceled'`, plano downgrade imediato para free (`plan='free'`, `max_corrections=5`, `corrections_used` mantém)
3. Email "Seu reembolso foi processado" (template `subscription-refunded`, pós-MVP se não existir)
4. Audit `subscription.refund` with `abacate_refund_id`

Política: 7 dias para cancelar e reembolsar integral dentro do período do trial, conforme LGPD/CDC.

> AbacatePay só permite **refund integral** — ver [tech-stack.md](../architecture/tech-stack.md) § Limitações. Sem partial refund.

---

## Bulk actions (pós-MVP)

| Ação | UX |
|---|---|
| "Selecionar N users e grant corrections" | Checkbox + bulk action menu (pós Fase 5) |
| "Estender todos plans past_due por 3 dias" | Ação do banner de debt (pós Fase 5) |

Não bloqueador para v2.0.

---

## UX

| Estado | Comportamento |
|---|---|
| Modal "Confirmar refund" | Texto forte: "Esta ação dispara refund no AbacatePay e downgrada a assinatura IMEDIATAMENTE. Não pode ser desfeita." + check "Estou ciente". |
| Success | Toast `Sub ref performed + audit_id` |
| AbacatePay erro | Toast `Falha no AbacatePay: <message>`, status HTTP 502, não muta estado local |
| Subscription já cancelada | 409 conflict com mensagem |

---

## Considerações

- Grants manuais **não** aparecem no painel AbacatePay (não são cobranças). Têm efeito local apenas. Marcado claramente no audit log.
- Após grant plano por X dias, ao expirar `period_end`, job desce para free — mesmo comportamento que assinatura AbacatePay cancelada.
- Admin não pode forçar status para `active` se AbacatePay tem status canceled — isso criaria divergência. API valida abacate state antes de aceitar force-active (recomendação).
- Admin pode usar grant-plano como "plano cortesia" para usuários VIP. Esses usuários não aparecem em MRR (são não-pagantes). [admin/analytics.md](./analytics.md) — KPI separado "Cortesias ativas".