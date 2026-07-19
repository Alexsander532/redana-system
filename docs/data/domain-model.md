# Modelo de domínio

> Seção original: PLAN.md §7.1. Entidades principais e diagrama ER (expandido com entidades admin).

---

## Entidades principais (MVP core)

```mermaid
erDiagram
  users ||--|| subscriptions : "1:1"
  users ||--o{ essays : "1:N"
  essays ||--o| corrections : "1:1"
  users ||--o{ abacate_customers : "1:1 (opcional)"

  users {
    uuid id PK
    text email UNIQUE
    text name
    text avatar_url
    text auth_provider
    timestamptz created_at
  }
  subscriptions {
    uuid id PK
    uuid user_id FK
    text plan "free|monthly|quarterly|annual"
    text status "active|past_due|canceled|expired"
    text abacate_subscription_id
    int corrections_used
    int max_corrections "null = unlimited"
    timestamptz period_start
    timestamptz period_end
  }
  essays {
    uuid id PK
    uuid user_id FK
    text title
    text content
    int word_count
    text status "pending|processing|completed|failed"
    text source "text|file"
    timestamptz created_at
    timestamptz completed_at
  }
  corrections {
    uuid id PK
    uuid essay_id FK UNIQUE
    uuid user_id FK
    int overall_score "0-1000"
    int c1_score "0-200"
    int c2_score
    int c3_score
    int c4_score
    int c5_score
    text c1_feedback
    text c2_feedback
    text c3_feedback
    text c4_feedback
    text c5_feedback
    text overall_feedback
    jsonb next_steps
    text corrected_by "ai|human"
    text llm_model
    numeric llm_cost_brl
    timestamptz created_at
  }
```

---

## Entidades adicionadas para Admin Panel (v2.0)

```mermaid
erDiagram
  users ||--o{ admin_audit_log : "1:N (admin acting)"
  users ||--o{ prompt_templates : "created_by"
  users ||--o{ email_logs : "sent_to / sent_by"
  feature_flags ||--o{ feature_flag_overrides : "user-specific"

  users {
    uuid id PK
    text email UNIQUE
    text name
    text role "user|admin"        -- NOVO v2.0
    timestamptz created_at
  }
  admin_audit_log {
    uuid id PK
    uuid admin_id FK            -- users.id onde role='admin'
    text action                  -- e.g. 'user.suspend', 'correction.human_override'
    jsonb payload
    text ip_address
    text user_agent
    timestamptz created_at
  }
  prompt_templates {
    uuid id PK
    text name UNIQUE
    text content
    bool active
    uuid created_by FK
    timestamptz updated_at
  }
  feature_flags {
    uuid id PK
    text key UNIQUE
    bool enabled
    text description
    timestamptz updated_at
  }
  feature_flag_overrides {
    uuid id PK
    uuid flag_id FK
    uuid user_id FK
    bool enabled
  }
  webhook_events {
    uuid id PK
    text abacate_event_id UNIQUE
    text event_type
    jsonb raw_payload
    text status "received|processed|failed"
    timestamptz received_at
  }
  email_logs {
    uuid id PK
    uuid user_id FK               -- destinatário
    text template_name
    text subject
    text status "sent|failed|opened"
    jsonb metadata
    uuid sent_by FK               -- admin se broadcast; null se sistema
    timestamptz sent_at
  }
  plans_config {
    uuid id PK
    text plan UNIQUE             -- free|monthly|quarterly|annual
    int price_cents
    int max_corrections
    int cycle_days
    bool active
    timestamptz updated_at
  }
```

> O SQL completo (incl. constraints, RLS, índices, invariáveis) está em [schema.md](./schema.md).

---

## Cardinalidades e regras

| Relação | Card | Regra |
|---|---|---|
| `users` → `subscriptions` | 1:1 | Cada user tem exatamente 1 subscription (criada no signup como free). |
| `users` → `essays` | 1:N | User pode ter N essays. |
| `essays` → `corrections` | 1:1 | Essa tem NO MÁXIMO 1 correction. `corrections.essay_id` é UNIQUE. |
| `users` → `admin_audit_log` | 1:N | Quando `users.role='admin'`, registra ações como `admin_id`. |
| `prompt_templates` → `users` | N:1 | `created_by` referencia admin que criou. |
| `webhook_events` | — | Append-only; pode haver duplicados defendidos por `abacate_event_id` UNIQUE. |

---

## Entidades futuras (não implementadas no MVP)

| Entidade | Quando | Obs. |
|---|---|---|
| `abacate_customers` | Fase 3 | Cache de customer_id AbacatePay; opcional no MVP. |
| `human_correction_queue` | ~~Fase 5~~ já incluída | Implementado como filtro em `essays` (`status='processing'` + `corrected_by='human'` ou coluna `assigned_admin_id`). |
| `notifications` | Pós-MVP | Para push/in-app notifications. |