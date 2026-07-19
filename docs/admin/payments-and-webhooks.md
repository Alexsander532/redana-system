# Admin — Pagamentos e Webhooks AbacatePay

> Documento detalhado da visão admin do AbacatePay. Para o handler técnico do webhook, ver [api/webhooks-abacatepay.md](../api/webhooks-abacatepay.md).

---

## Tela `/admin/pagamentos`

Foco em diagnóstico: pagamentos falhados, drifts, reprocessáveis.

### Sub-tela 1: Pagamentos falhados (default)

Tabela de subscriptions com `status='past_due'`:

| Coluna | Conteúdo |
|---|---|
| User | Email |
| Plano | `monthly`/`annual`/... |
| Status | Badge red `past_due` |
| Última cobrança (estimada) | Date |
| Dias em past_due | `period_end - now()` negativo count |
| Tries webhook | Count `webhook_events` recentes |
| Ações | Send "atualize cartão" email / Force cancel / Ver detalhe |

### Sub-tela 2: Reconciliação

Apresenta output do último job de reconciliação:

- Total subs comparadas
- Drifts detectados (com detail)
- Last drift timestamp

Botão "Force reconcile now" → chama `POST /api/v1/internal/reconcile` (sistema API key).

---

## Tela `/admin/webhooks` (sub-aba)

Tabela `webhook_events` paginada:

| Coluna | Conteúdo |
|---|---|
| Abacate event id | Truncated |
| Type | `subscription.completed`/`renewed`/`cancelled`/`refunded`/`checkout.completed` |
| Status | `received`/`processing`/`processed`/`failed` badge |
| Received at | ts |
| Processed at | ts |
| Ações | Ver payload / Replay |

### Filtros

- `event_type`
- `status`
- date range (`received_at`)
- Free text search in raw_payload JSONB (LIKE)

### API

```
GET /api/v1/admin/webhooks?event_type=subscription.completed&status=failed&from=...&to=...&limit=50&offset=0
GET /api/v1/admin/webhooks/{id}     # raw payload view
POST /api/v1/admin/webhooks/{id}/replay
```

---

## Detalhe do evento `/admin/webhooks/[id]`

### Seção 1: Cabeçalho
- event_type, abacate_event_id, received_at, processed_at, status
- Error message (se failed)

### Seção 2: Raw payload JSON
- Highlight JSON viewer
- Botão "Copy"

### Seção 3: Action aplicada
- Identifica qual handler (`TODO_MAP[event_type]`) foi invocado
- Effect side: e.g. "Updated subscription <id> status → active" (parseado de logs)
- Audit log relacionado (se houve mutation automática)

### Seção 4: Replay
- Botão "Replay este evento"
- Confirma modal: "Reprocessa o handler. Idempotência aplicada — se já aplicado, é no-op"
- Após replay: shows new audit_id

---

## Replay endpoint

```
POST /api/v1/admin/webhooks/{id}/replay
```

Efeito:
1. Busca `webhook_events` row → pega `raw_payload`
2. Verify idempotência: se handler já aplicado totalmente, return 200 with note "no-op (idempotency)"
3. Re-invoca handler (`TODO_MAP[event_type]`) com `raw_payload`
4. Atualiza `webhook_events.status` para `processed` (ou mantém `failed` se ainda falhar)
5. Audit `webhook.replay` with `target_id=webhook_event.id`

> Idempotência é crítica: handler deve ser deterministic re-executable. Cada handler deve checkar estado atual antes de mutar. E.g. `subscription.renewed`: só reseta `corrections_used` se periódico atual já expirou. Ver [webhooks-abacatepay.md](../api/webhooks-abacatepay.md).

---

## Drift detection (reconciliação)

Job `reconcile_subscriptions()` roda diariamente:

1. `GET /subscriptions` AbacatePay paginado
2. Para cada Abacate sub, match com `subscriptions.abacate_subscription_id`
3. Comparar campos: `status`, `period_end`
4. Se divergente:
   - Atualiza DB para match AbacatePay (fonte de verdade para cobrança)
   - Cria `webhook_event` sintético com `event_type='reconciliation.drift'` + metadata diff
   - Audit `subscription.reconcile.drift`
5. Cancelados (`AbacatePay status=canceled`) com period_end ≤ now → downgrade para free

→ Resultado visível na sub-tela "Reconciliação" acima.

---

## "Force reconcile now"

```
POST /api/v1/internal/reconcile
Header: X-Internal-Api-Key: <API_KEY>
```

> Rota protegida por system API key (não por user JWT), pois é invocável do cron Railway OU do botão admin via proxy. Implementação: admin frontend chama `POST /api/v1/admin/reconcile/run` que internamente invoca o job.

---

## Tabela failures

Sugerido: derivado de `webhook_events WHERE status='failed'` + `subscriptions WHERE status='past_due'`. Mostrar em banner no topo do /admin/pagamentos.

| Evento | Count last 7d | Trend |
|---|---|---|
| `subscription.cancelled` (unexpected) | 3 | ↑ |
| `checkout.completed` failed to process | 1 | → |
| HMAC mismatch | 0 | → |

---

## Alertas configuráveis (futuro)

Via feature flags + cron:
- "Se webhook failed > 5 em 24h → email admin@redana.com.br"
- "Se drift > 10 subs → telegram/Sentry alert"

Pós-MVP; infrastructure já está pronta (events table + webhooks integration).