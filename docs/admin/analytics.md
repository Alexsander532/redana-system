# Admin — Métricas e charts

> Documento detalhado do dashboard de analytics do admin. KPIs, queries, charts.

Lib de charts: **Recharts** (mesma do `EvolutionChart` do dashboard do aluno). Componentes admin em `apps/web/src/components/admin/`.

---

## Tela `/admin/metricas`

Layout: KPI cards no topo + grid 2x2 de charts + tabela dinâmica abaixo.

### KPI cards

| Card | Definição | Cálculo SQL |
|---|---|---|
| **MRR** | Monthly recurring revenue (somatório mensalizado) | `SELECT sum(price_cents)/100/12 * CASE cycle_days WHEN 365 THEN 12 WHEN 90 THEN 3 WHEN 30 THEN 1 END FROM subscriptions s JOIN plans_config p ON s.plan=p.plan WHERE s.status='active'` (simplificação) |
| **Assinantes ativos** | Count subs `status='active'` e `plan != 'free'` | `SELECT count(*) FROM subscriptions WHERE status='active' AND plan != 'free'` |
| **Usuários totais** | Count | `SELECT count(*) FROM users` |
| **Conversão Free→Pago (30d)** | `subs ativos criados nos últimos 30d / users criados nos últimos 30d` | Calculado |
| **Correções hoje** | `SELECT count(*) FROM corrections WHERE created_at::date = today()` |
| **Correções last 7d** | idem |
| **Custo LLM hoje** | `SELECT sum(llm_cost_brl) FROM corrections WHERE created_at::date = today()` |
| **Latência média hoje** | `SELECT avg(completed_at - created_at) FROM essays WHERE status='completed' AND completed_at::date = today()` |
| **Taxa de falha hoje** | `SELECT count(*) FILTER (WHERE status='failed') * 100 / count(*) FROM essays WHERE created_at::date = today()` |
| **Cortesias ativas** | Subs criadas via admin grant (`abacate_subscription_id IS NULL`) | `WHERE abacate_subscription_id IS NULL AND status='active' AND plan != 'free'` |

`MetricCard` componente: número grande + delta % vs período anterior (sparkline opcional).

---

### Charts

#### `MrrChart` (Area chart 30d/90d)

```sql
SELECT date_trunc('day', period_start) as day, sum(price_cents)/100 as mrr_brl
FROM subscriptions s JOIN plans_config p ON s.plan=p.plan
WHERE s.status='active' AND s.period_start >= now() - interval '90 days'
GROUP BY 1 ORDER BY 1;
```

> `MRR` é uma snapshot por dia (cumulativo). Para média corrida, calc normalization: cada plano contribui com `price / cycle_days` por dia.

#### `LlmCostChart` (Bar chart 30d)

```sql
SELECT date_trunc('day', created_at) as day, sum(llm_cost_brl) as cost_brl
FROM corrections
WHERE created_at >= now() - interval '30 days'
GROUP BY 1 ORDER BY 1;
```

#### `CorrectionsChart` (Bar chart 30d)

```sql
SELECT date_trunc('day', created_at) as day,
       count(*) as total,
       count(*) FILTER (WHERE corrected_by='ai') as ai,
       count(*) FILTER (WHERE corrected_by='human') as human
FROM corrections
WHERE created_at >= now() - interval '30 days'
GROUP BY 1 ORDER BY 1;
```

Stacked: ai vs human.

#### `ConversionChart` (Line chart 30d)

Free → Pago conversão por dia:
```sql
SELECT date_trunc('day', u.created_at) as day,
       count(*) as users,
       count(*) FILTER (WHERE s.status='active' AND s.plan != 'free') as paid
FROM users u LEFT JOIN subscriptions s ON s.user_id = u.id
WHERE u.created_at >= now() - interval '30 days'
GROUP BY 1 ORDER BY 1;
```

Conversão = `paid / users` por dia.

#### `PlanBreakdownChart` (Donut)

```sql
SELECT plan, count(*) FROM subscriptions WHERE status='active' GROUP BY plan;
```

#### `LatencyChart` (Line 30d)

```sql
SELECT date_trunc('day', completed_at) as day,
       extract(epoch FROM avg(completed_at - created_at)) as avg_latency_sec
FROM essays WHERE status='completed' AND completed_at >= now() - interval '30 days'
GROUP BY 1 ORDER BY 1;
```

---

## APIs

```
GET /api/v1/admin/analytics/overview                # todos os KPIs em uma chamada
GET /api/v1/admin/analytics/mrr?range=30d          # timeseries 30/90d
GET /api/v1/admin/analytics/llm-cost?range=30d
GET /api/v1/admin/analytics/corrections?range=30d
GET /api/v1/admin/analytics/conversion?range=30d
GET /api/v1/admin/analytics/latency?range=30d
GET /api/v1/admin/analytics/plans-breakdown
```

Resposta `overview` exemplo:

```json
{
  "snapshot_at": "...",
  "mrr_brl": 4250.50,
  "active_subscribers": 510,
  "total_users": 8200,
  "conversion_30d_pct": 6.2,
  "corrections_today": 142,
  "corrections_7d": 920,
  "llm_cost_today_brl": 4.30,
  "llm_cost_7d_brl": 27.10,
  "latency_today_sec": 28.5,
  "failure_rate_today_pct": 1.4,
  "cortesia_active_subs": 8
}
```

Resposta `mrr`:
```json
{
  "range": "30d",
  "series": [
    {"day": "2026-06-18", "mrr": 3950.00},
    {"day": "2026-06-19", "mrr": 4120.00},
    ...
  ]
}
```

---

## Período custom

Todas as rotas aceitam `?from=ISO&to=ISO` além de `?range=30d` ou `?range=90d`.

---

## Custo/cálculo servidor

Queries são agregadas no Postgres (não na API). Indices recomendados:

```sql
create index idx_corrections_created on corrections (created_at);
create index idx_essays_completed on essays (completed_at) where status='completed';
create index idx_subscriptions_status_plan on subscriptions (status, plan);
```

Para volumes > 100k corrections, considerar materialized view ou tabela pré-agregada (`daily_metrics`):
```sql
create table daily_metrics (
  day date primary key,
  corrections_total int, corrections_ai int, corrections_human int,
  llm_cost_brl numeric,
  avg_latency_sec numeric,
  failed_essays int
);
```

Job Madrugada atualiza (ver [admin/system-config.md](./system-config.md)). Não necessário no MVP.

---

## UX states

| Estado | Comportamento |
|---|---|
| Loading | Spinner sobre o KPI card; charts com skeleton |
| Sem dados no range | "Sem dados neste período" |
| Erro de query | Toast "Falha ao carregar métricas" |
| Range invalid | Erro inline |

---

## Export

Botão "Exportar CSV" (pós-MVP) baixa qualquer series temporais. Não bloqueador v2.0.