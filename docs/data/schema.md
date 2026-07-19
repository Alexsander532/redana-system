# Schema SQL, RLS, invariáveis e adições admin

> Seção original: PLAN.md §7.2 (SQL) + §7.3 (invariáveis). Inclui adições do Admin Panel (v2.0).

---

## Schema MVP core

```sql
-- packages/db/schema.sql

create table users (
  id uuid primary key default gen_random_uuid(),
  email text unique not null,
  name text,
  avatar_url text,
  auth_provider text not null default 'email',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

create table subscriptions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references users(id) on delete cascade,
  plan text not null default 'free' check (plan in ('free','monthly','quarterly','annual')),
  status text not null default 'active' check (status in ('active','past_due','canceled','expired')),
  abacate_subscription_id text,
  abacate_customer_id text,
  corrections_used int not null default 0,
  max_corrections int,  -- null = unlimited
  period_start timestamptz,
  period_end timestamptz,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

create table essays (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references users(id) on delete cascade,
  title text,
  content text not null,
  word_count int not null,
  status text not null default 'pending' check (status in ('pending','processing','completed','failed')),
  source text check (source in ('text','file')),
  file_url text,
  created_at timestamptz default now(),
  completed_at timestamptz
);

create table corrections (
  id uuid primary key default gen_random_uuid(),
  essay_id uuid not null references essays(id) on delete cascade,
  user_id uuid not null references users(id) on delete cascade,
  overall_score int not null check (overall_score between 0 and 1000),
  c1_score int not null check (c1_score between 0 and 200),
  c2_score int not null check (c2_score between 0 and 200),
  c3_score int not null check (c3_score between 0 and 200),
  c4_score int not null check (c4_score between 0 and 200),
  c5_score int not null check (c5_score between 0 and 200),
  c1_feedback text, c2_feedback text, c3_feedback text, c4_feedback text, c5_feedback text,
  overall_feedback text,
  next_steps jsonb,
  corrected_by text not null default 'ai',
  llm_model text,
  llm_cost_brl numeric(10,4),
  created_at timestamptz default now()
);

-- RLS:
alter table essays enable row level security;
create policy "user_sees_own_essays" on essays for select using (auth.uid() = user_id);
create policy "user_inserts_own_essays" on essays for insert with check (auth.uid() = user_id);

alter table corrections enable row level security;
create policy "user_reads_own_corrections" on corrections for select using (auth.uid() = user_id);

alter table subscriptions enable row level security;
create policy "user_reads_own_sub" on subscriptions for select using (auth.uid() = user_id);

-- Indexes
create index idx_essays_user_created on essays (user_id, created_at desc);
create index idx_corrections_essay on corrections (essay_id);
create index idx_subscriptions_user on subscriptions (user_id);
```

---

## Adições Admin Panel (v2.0)

Vacinas de role + RBAC + auditlog + configurações.

```sql
-- Adições à tabela users:
alter table users
  add column role text not null default 'user' check (role in ('user','admin')),
  add column status text not null default 'active' check (status in ('active','suspended','banned')),
  add column suspended_until timestamptz,
  add column banned_reason text,
  add column last_admin_action_at timestamptz;

-- RLS para role/status: admins podem ler todos os users; user só lê a si mesmo
create policy "admin_reads_all_users" on users for select
  using (auth.uid() is not null and exists (
    select 1 from users u where u.id = auth.uid() and u.role = 'admin'
  ));
create policy "user_reads_self_only" on users for select
  using (auth.uid() = id);

-- Audit log (imutável)
create table admin_audit_log (
  id uuid primary key default gen_random_uuid(),
  admin_id uuid not null references users(id) on delete restrict,
  action text not null,                                  -- namespace.action, ex: 'user.suspend'
  target_type text,                                       -- 'user' | 'essay' | 'subscription' | ...
  target_id uuid,
  payload jsonb,                                          -- antes/depois, motivo, etc.
  ip_address inet,
  user_agent text,
  created_at timestamptz not null default now()
);
-- RLS: admin-only leitura; inserts só via service role
alter table admin_audit_log enable row level security;
create policy "admin_reads_audit_log" on admin_audit_log for select
  using (exists (select 1 from users u where u.id = auth.uid() and u.role = 'admin'));
-- (inserts via API usando service role, não aut.uid())

create index idx_audit_admin_created on admin_audit_log (admin_id, created_at desc);
create index idx_audit_target on admin_audit_log (target_type, target_id);

-- Feature flags
create table feature_flags (
  id uuid primary key default gen_random_uuid(),
  key text unique not null,        -- ex: 'broadcast_emails', 'human_correction'
  enabled boolean not null default false,
  description text,
  updated_by uuid references users(id),
  updated_at timestamptz not null default now()
);
alter table feature_flags enable row level security;
create policy "admin_reads_flags" on feature_flags for select using (true);  -- readable (checked by API anyway)
create policy "admin_writes_flags" on feature_flags for all
  using (exists (select 1 from users u where u.id = auth.uid() and u.role = 'admin'));

create table feature_flag_overrides (
  id uuid primary key default gen_random_uuid(),
  flag_id uuid not null references feature_flags(id) on delete cascade,
  user_id uuid not null references users(id) on delete cascade,
  enabled boolean not null,
  unique (flag_id, user_id)
);

-- Prompt templates
create table prompt_templates (
  id uuid primary key default gen_random_uuid(),
  name text unique not null,        -- 'enem_default', 'enem_v2_strict', ...
  content text not null,            -- template com placeholders: {{essay_text}}
  active boolean not null default false,  -- apenas 1 active por vez
  created_by uuid references users(id),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
-- Garante só um active
create unique index uniq_active_prompt on prompt_templates (name) where (active = true);

alter table prompt_templates enable row level security;
create policy "admin_all_prompt_templates" on prompt_templates for all
  using (exists (select 1 from users u where u.id = auth.uid() and u.role = 'admin'));

-- Webhook events (log bruto)
create table webhook_events (
  id uuid primary key default gen_random_uuid(),
  abacate_event_id text unique not null,    -- idempotência
  event_type text not null,
  raw_payload jsonb not null,
  status text not null default 'received' check (status in ('received','processed','failed')),
  error text,
  received_at timestamptz not null default now(),
  processed_at timestamptz
);
create index idx_webhook_type_received on webhook_events (event_type, received_at desc);
-- Sem RLS (apenas API/Service role tem acesso)

-- Email logs
create table email_logs (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references users(id) on delete set null,
  template_name text not null,
  subject text,
  status text not null default 'sent' check (status in ('queued','sent','failed','opened','bounced')),
  metadata jsonb,
  sent_by uuid references users(id) on delete set null,    -- admin se broadcast; null se sistema
  sent_at timestamptz not null default now()
);
create index idx_email_logs_sent_at on email_logs (sent_at desc);
create index idx_email_logs_user on email_logs (user_id, sent_at desc);
alter table email_logs enable row level security;
create policy "admin_reads_email_logs" on email_logs for select
  using (exists (select 1 from users u where u.id = auth.uid() and u.role = 'admin'));

-- Plans config (configuração de preços persistida)
create table plans_config (
  id uuid primary key default gen_random_uuid(),
  plan text unique not null check (plan in ('free','monthly','quarterly','annual')),
  price_cents int not null default 0,
  max_corrections int not null default 0,        -- 5 para free, 30 para pagos
  cycle_days int not null default 0,              -- 0 para free, 30/90/365 para pagos
  active boolean not null default true,
  updated_by uuid references users(id),
  updated_at timestamptz not null default now()
);
alter table plans_config enable row level security;
create policy "admin_writes_plans" on plans_config for all
  using (exists (select 1 from users u where u.id = auth.uid() and u.role = 'admin'));

-- Eventos de analytics
create table events (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references users(id) on delete set null,
  event_type text not null,
  properties jsonb,
  created_at timestamptz default now()
);
create index idx_events_type_created on events (event_type, created_at desc);
create index idx_events_user_created on events (user_id, created_at desc);
```

---

## Invariáveis

- `corrections.overall_score = c1+c2+c3+c4+c5` (validado server-side; check constraint pode não cobrir soma mas tem trigger)
- `subscriptions.corrections_used <= max_corrections` (ou `max_corrections` é `null`)
- **Free tier:** `max_corrections = 5`, **vitalício** (não reseta)
- **Planos pagos:** `max_corrections = 30` por ciclo (não "unlimited"; documenta como "até 30/mês")
- `essays.status` só vai `completed` quando há `corrections` row vinculada
- `users.role` defaults to `'user'`; admin é atribuído manualmente (não há self-promotion)
- `admin_audit_log` é **append-only** — nenhuma rota faz UPDATE/DELETE (RLS de delete ausente + API não expõe)
- `prompt_templates` tem no máximo 1 row `active=true` (índice unique partial)
- `webhook_events.abacate_event_id` é UNIQUE → idempotência
- `essays.status='failed'` exige `corrections IS NULL` (correção não persistida)

### Trigger recomendado: soma overall_score

```sql
create or replace function validate_overall_score()
returns trigger as $$
begin
  if new.overall_score <> new.c1_score + new.c2_score + New.c3_score + new.c4_score + new.c5_score then
    raise exception 'overall_score must equal sum of C1-C5';
  end if;
  return new;
end;
$$ language plpgsql;

create trigger trg_validate_overall
  before insert or update on corrections
  for each row execute function validate_overall_score();
```

---

## RLS — Resumo

| Tabela | RLS | Policy principal |
|---|---|---|
| `users` | ✅ | user reads self; admin reads all |
| `essays` | ✅ | user owns; admin reads all (mesma regra admin) |
| `corrections` | ✅ | user reads own; admin reads all |
| `subscriptions` | ✅ | user reads own; admin reads/writes all |
| `admin_audit_log` | ✅ | admin reads only; inserts via service role |
| `feature_flags` | ✅ | readable public; admin writes |
| `prompt_templates` | ✅ | admin writes/reads; API reads (service role) |
| `webhook_events` | ✅ | adminOnly visible |
| `email_logs` | ✅ | admin reads |
| `plans_config` | ✅ | admin writes |
| `events` | ❌ | service role only (inserido pela API) |

> Para todas tabelas admin-needed (essays/corrections), permita leitura por admin. Use `exists(select 1 from users where id = auth.uid() and role='admin')` na policy.