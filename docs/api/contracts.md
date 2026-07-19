# Contratos de API

> Seção original: PLAN.md §8. Todas as rotas da API, **+ rotas admin (seção nova v2.0)**.

Base: `https://api.redana.com.br/api/v1`

Todas as rotas autenticadas exigem header `Authorization: Bearer <JWT Supabase>`. Rotas admin exigem adicionalmente `users.role = 'admin'` (ver [auth-and-authorization.md](./auth-and-authorization.md)).

---

## Auth (Supabase client + API verify)

| Método | Rota | Descrição |
|---|---|---|
| POST | `/auth/signup` | Cria user no Supabase + cria `subscriptions` default free |
| POST | `/auth/login` | Login via Supabase, retorna JWT |
| POST | `/auth/logout` | Refresh token invalidate |
| GET | `/auth/me` | Perfil + subscription summary |

---

## Essays

| Método | Rota | Descrição |
|---|---|---|
| GET | `/essays?limit=20&offset=0` | Lista paginada (dono) |
| POST | `/essays` | Cria redação → dispara correção async |
| GET | `/essays/{id}` | Redação + correction (se houver) |
| DELETE | `/essays/{id}` | Soft delete |
| GET | `/essays/{id}/status` | SSE/streaming de progresso (opcional) |

### `POST /essays` — body

```json
{
  "title": "Redação de treino",
  "content": "Texto da redação...",
  "source": "text"
}
```

ou multipart/form-data com `file: <.txt|.docx>`.

### Resposta 201

```json
{
  "id": "uuid",
  "status": "pending",
  "word_count": 320,
  "created_at": "2026-07-18T10:00:00Z"
}
```

---

## Corrections

| Método | Rota | Descrição |
|---|---|---|
| GET | `/corrections/{essay_id}` | Detalhe da correção |
| POST | `/corrections/{essay_id}/retry` | Refaz IA (não reseta `corrections_used`) |

---

## Subscriptions

| Método | Rota | Descrição |
|---|---|---|
| GET | `/subscriptions/me` | Estado atual: plano, status, usado/restante |
| POST | `/subscriptions/checkout` | Cria checkout AbacatePay → retorna URL |
| POST | `/subscriptions/cancel` | Cancela assinatura ativa |
| POST | `/subscriptions/change-plan` | Troca de plano (novo checkout) |

---

## Webhooks (público, com assinatura HMAC)

| Método | Rota | Descrição |
|---|---|---|
| POST | `/webhooks/abacatepay` | Notificações: subscription.completed, .renewed, .cancelled, checkout.completed |

→ Detalhe em [webhooks-abacatepay.md](./webhooks-abacatepay.md).

---

## Account (LGPD)

| Método | Rota | Descrição |
|---|---|---|
| POST | `/account/export` | Gera JSON com todos os dados do usuário |
| POST | `/account/delete` | Agenda deleção (grace period 30 dias) |

---

## Admin (NOVO v2.0)

Prefixo: `/api/v1/admin/*`

Todas as rotas admin exigem `role = 'admin'` e são logadas em `admin_audit_log` (ver [auth-and-authorization.md](./auth-and-authorization.md)).

### Users

| Método | Rota | Descrição |
|---|---|---|
| GET | `/admin/users?limit=20&offset=0&q=...&status=...` | Lista paginada de users (filtro por status/email/name) |
| GET | `/admin/users/{id}` | Detalhe: profile, subscription, essays, audit trail |
| POST | `/admin/users/{id}/suspend` | Suspende até `until` (opcional) |
| POST | `/admin/users/{id}/unsuspend` | Remove suspensão |
| POST | `/admin/users/{id}/ban` | Bane com `banned_reason` |
| POST | `/admin/users/{id}/unban` | Desbane |
| POST | `/admin/users/{id}/impersonate` | Emite token de impersonação (curta duração) para diagnóstico |
| POST | `/admin/users/{id}/role` | Promove/rebaixa role (`user` ↔ `admin`) |

### Essays

| Método | Rota | Descrição |
|---|---|---|
| GET | `/admin/essays?status=...&user_id=...&from=...&to=...` | Lista paginada com filtros |
| GET | `/admin/essays/{id}` | Detalhe (content + correction) |
| POST | `/admin/essays/{id}/reprocess` | Re-executa correção IA |
| PUT | `/admin/essays/{id}/correction` | Cria/atualiza correction manual (override humano) |

### Corrections

| Método | Rota | Descrição |
|---|---|---|
| GET | `/admin/corrections?corrected_by=...&assigned_admin=...` | Lista paginada |
| GET | `/admin/corrections/queue?status=...` | Fila de redações pendentes de correção humana |
| POST | `/admin/corrections/{essay_id}/assign` | Auto-assinalar ou assignar a outro admin |
| PUT | `/admin/corrections/{essay_id}` | Salvar correção humana (C1-C5 + feedback) → `corrected_by='human'` |

### Subscriptions

| Método | Rota | Descrição |
|---|---|---|
| GET | `/admin/subscriptions?status=...&plan=...` | Lista paginada |
| GET | `/admin/subscriptions/{id}` | Detalhe + webhook events relacionados |
| POST | `/admin/subscriptions/{id}/status` | Força mudança de status (active, past_due, canceled, expired) |
| POST | `/admin/subscriptions/{id}/grant-corrections` | Concede N correções avulsas (não altera plano) |
| POST | `/admin/subscriptions/{id}/grant-plan` | Grant manual de plano (free→monthly/annual) por X dias |
| POST | `/admin/subscriptions/{id}/extend` | Estende `period_end` por N dias |
| POST | `/admin/subscriptions/{id}/refund` | Dispara refund integral no AbacatePay + downgrade imediato |

### Payments & webhooks

| Método | Rota | Descrição |
|---|---|---|
| GET | `/admin/payments/failed` | Dashboard de pagamentos falhados (subs `past_due`) |
| GET | `/admin/webhooks?event_type=...&from=...&to=...` | Lista de eventos webhook (`webhook_events`) |
| GET | `/admin/webhooks/{id}` | Detalhe (raw payload, status processamento) |
| POST | `/admin/webhooks/{id}/replay` | Re-processa evento webhook manualmente |

### Analytics

| Método | Rota | Descrição |
|---|---|---|
| GET | `/admin/analytics/overview` | KPIs: MRR, ativos, conversão, latência, custo, falhas |
| GET | `/admin/analytics/mrr?range=30d` | Série temporal MRR |
| GET | `/admin/analytics/llm-cost?range=30d` | Série custo LLM/dia |
| GET | `/admin/analytics/corrections?range=30d` | Volume correções/dia |
| GET | `/admin/analytics/conversion?range=30d` | Conversão Free → Pago |

### System config

| Método | Rota | Descrição |
|---|---|---|
| GET | `/admin/prompts` | Lista prompt templates |
| POST | `/admin/prompts` | Cria novo template |
| PUT | `/admin/prompts/{id}` | Edita (se não active, edita sem quebrar produção) |
| POST | `/admin/prompts/{id}/activate` | Ativa (desativa os demais do mesmo name) |
| GET | `/admin/feature-flags` | Lista flags |
| PUT | `/admin/feature-flags/{id}` | Toggle |
| POST | `/admin/feature-flags/{id}/override` | Override por user |
| GET | `/admin/plans` | Lista planos config |
| PUT | `/admin/plans/{id}` | Edita preço/limites/cycle |
| POST | `/admin/emails/broadcast` | Envia email broadcast para um segmento |
| GET | `/admin/emails?template=...&status=...` | History de emails |
| GET | `/admin/logs?level=...&service=...&limit=100` | API logs (via Logflare proxy ou tabela `events`) |
| GET | `/admin/health` | Healthcheck estendido (DB, LLM, AbacatePay) |
| GET | `/admin/audit-log?admin_id=...&action=...` | Filtra audit log |

### Exemplos

#### `POST /admin/users/{id}/suspend`

```json
{ "until": "2026-08-18T00:00:00Z", "reason": "Comportamento inadequado" }
```

Resposta 200:
```json
{ "ok": true, "audit_id": "uuid" }
```

#### `PUT /admin/corrections/{essay_id}` (humana)

```json
{
  "c1_score": 160,
  "c2_score": 200,
  "c3_score": 160,
  "c4_score": 200,
  "c5_score": 160,
  "overall_score": 880,
  "c1_feedback": "...",
  "c2_feedback": "...",
  "c3_feedback": "...",
  "c4_feedback": "...",
  "c5_feedback": "...",
  "overall_feedback": "...",
  "next_steps": [{ "area": "c5", "action": "Detalhar agente, ação, meio e finalidade", "priority": "alta" }]
}
```

> A validação server-side exige `overall_score = soma C1-C5`. `corrected_by = 'human'`, `llm_model = null`, `llm_cost_brl = 0`.

#### `POST /admin/subscriptions/{id}/grant-corrections`

```json
{ "amount": 10, "reason": "Compensação por lentidão" }
```

→ Incrementa `max_corrections` em `amount` (se era null, fica null + log).

---

## Esquemas comuns (Pydantic)

```python
class Paginated[T](BaseModel, Generic[T]):
    items: list[T]
    total: int
    limit: int
    offset: int

class AdminListMeta(BaseModel):
    items: ...
    total: ...
```

---

## Erros padrão

```json
{
  "error": {
    "code": "forbidden",
    "message": "Admin role required",
    "audit_id": "uuid"   // presente quando logado
  }
}
```

| HTTP | code | Quando |
|---|---|---|
| 400 | `bad_request` | Pydantic validation failed |
| 401 | `unauthorized` | JWT missing/invalid |
| 403 | `forbidden` | role insufficient (e.g. user calling /admin) |
| 404 | `not_found` | Recurso não existe ou não pertence ao user (RLS também vetorunamente) |
| 409 | `conflict` | Estado inválido (e.g. já banido, plano já ativo) |
| 422 | `validation_error` | LLM JSON inválido após retry |
| 429 | `rate_limited` | Rate limit excedido — ver [rate-limiting.md](./rate-limiting.md) |
| 500 | `internal_error` | Erro não tratado (Sentry captura) |