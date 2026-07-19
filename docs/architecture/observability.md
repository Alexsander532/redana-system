# Observabilidade

> Seção original: PLAN.md §14. Sinais de observabilidade e métricas-chave (expandido com uso pelo admin panel).

---

## Sinais

| Sinal | Ferramenta | Onde | Detalhe |
|---|---|---|---|
| Erros de aplicação | Sentry | API + Web | DSN por ambiente; release tracking via git SHA. |
| Logs de runtime | Logflare (Supabase logs) | API | Structured logs JSON nivelado (`logger.info({"event":"essay.created", ...})`). |
| Métricas de negócio | Tabela `events` no DB | API insere | Append-only; agregações SQL no admin. |
| Up/Down | Better Stack (ping) | Endpoint `/health` | Ping a 1 min; alerta em falha de 3 consecutivos. |
| Custo LLM | Log por correction (coluna `llm_cost_brl`) + agregação SQL | API | Agregação no admin (`/admin/analytics`). |
| Webhooks AbacatePay | Tabela `webhook_events` | API | Cada webhook bruto persistido (antes do handler). |
| Ações admin | Tabela `admin_audit_log` | API (decorator `audit_action`) | Imutável. Ver [admin/overview.md](../../admin/overview.md). |
| Latência correção | Métrica calculada (`essays.completed_at - essays.created_at`) | DB | Exposta no admin. |
| Uptime API | `/health` + `/ready` | FastAPI | `/health` = liveness, `/ready` = readização (DB + LLM). |

### Endpoints de health

```python
@app.get("/health")
async def health():
    return {"status": "ok"}

@app.get("/ready")
async def ready(db: AsyncSession = Depends(get_db)):
    await db.execute(text("select 1"))
    # opcional: ping na API do Claude
    return {"status": "ready", "db": "ok"}
```

### Estrutura de log (apenas exemplo)

```json
{
  "timestamp": "2026-07-18T10:00:00Z",
  "level": "info",
  "event": "correction.completed",
  "essay_id": "uuid",
  "user_id": "uuid",
  "llm_model": "claude-3-5-sonnet",
  "llm_cost_brl": 0.034,
  "latency_ms": 24000,
  "overall_score": 880
}
```

---

## Métricas-chave

| Métrica | Definição | Exposta em |
|---|---|---|
| Conversão Free → Pago (mensal) | `subscriptions.status='active' AND plan != 'free'` / `total users last 30d` | [admin/analytics.md](../../admin/analytics.md) |
| MRR | Soma das assinaturas ativas mensalizadas | [admin/analytics.md](../../admin/analytics.md) |
| Correções por dia / custo médio BRL | `count(corrections)/day`, `avg(llm_cost_brl)` | [admin/analytics.md](../../admin/analytics.md) |
| Taxa de falha de LLM | `essays.status='failed'` / total | [admin/analytics.md](../../admin/analytics.md) |
| Tempo médio de correção (latência) | `avg(completed_at - created_at)` | [admin/analytics.md](../../admin/analytics.md) |
| Planos ativos por tipo | Count por `plan` | [admin/analytics.md](../../admin/analytics.md) |
| Fila de correções pendentes | `count(essays WHERE status='pending' OR 'processing')` | [admin/system-config.md](../../admin/system-config.md) (logs) |
| Eventos webhook AbacatePay | Count por `event_type` nos últimos 7d | [admin/payments-and-webhooks.md](../../admin/payments-and-webhooks.md) |
| Ações admin (audit) | Count/tipo nos últimos 7d | [admin/overview.md](../../admin/overview.md) |

### Tabela `events` (resumo)

```sql
create table events (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references users(id) on delete set null,
  event_type text not null,            -- 'signup', 'essay.created', 'correction.completed', 'checkout.initiated', etc.
  properties jsonb,
  created_at timestamptz default now()
);
create index idx_events_type_created on events (event_type, created_at desc);
create index idx_events_user_created on events (user_id, created_at desc);
```

> Sem RLS (a tabela é owner-only; inserts só pela API com service role key).

---

## Alertas (sugestão inicial)

| Alerta | Gatilho | Ação |
|---|---|---|
| LLM down | `essays.status='failed'` rate > 10% em 10 min | Sentry + email para admin |
| Webhook ausente | Sem evento `subscription.renewed` esperado em 24h | Job de reconciliação dispara manual fetch |
| custo LLM/day > R$ X | Agregação diária | Notificação admin |
| Fila > 50 pendentes | Query `count(essays.status='pending')` | Investigar Celery+Redis migration |