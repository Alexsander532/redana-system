# Migrations

> Documento NOVO (v2.0). Como as migrations são geridas entre Alembic e Supabase.

---

## Estratégia

O projeto usa **dois canais** de migrations que precisam estar sincronizados:

1. **`packages/db/schema.sql`** — Schema canônico, declarativo, usado para:
   - Resetar o banco em dev (`drop + recreate`)
   - Aplicar a um Supabase project novo (cola no SQL Editor do Supabase Studio ou via `psql`)
   - Documentação viva do estado atual do DB

2. **`apps/api/alembic/versions/`** — Migrations versionadas via Alembic, usadas para:
   - Aplicar mudanças incrementais no ambiente de produção
   - Track no git com mensagem semântica (e.g. `add_role_to_users`)
   - Rollback via `alembic downgrade`

> **Regra:** Toda mudança de schema deve ser refletida em ambos. Primeiro escreva a migration Alembic, depois atualize `schema.sql` para refletir o estado final.

---

## Workflow padrão

### Adicionar uma coluna/tabela

```bash
# 1. Gerar a migration
cd apps/api
alembic revision -m "add role column to users"

# 2. Editar apps/api/alembic/versions/<timestamp>_add_role_column_to_users.py
#    implementar upgrade() e downgrade()

# 3. Testar localmente
alembic upgrade head
alembic downgrade -1     # verifica downgrade
alembic upgrade head

# 4. Commit
git add apps/api/alembic/versions/*.py packages/db/schema.sql
git commit -m "db: add users.role column for admin RBAC"
```

### Aplicar em produção (Supabase)

Duas opções equivalentes:

| Opção | Quando |
|---|---|
| **Alembic CI job** | Recomendado — job no CI roda `alembic upgrade head` após deploy da API |
| **Supabase Studio SQL Editor** | Para hotfix rápido de produção — cola SQL da migration; não versionaautomaticamente |

> Preferir Alembic. O SQL Editor do Supabase é fallback para emergência.

---

## Convenção de nomenclatura

- Files Alembic: `<timestamp>_<snake_case_slug>.py` (gerado automaticamente)
- Slug: uso a forma verbal + entidade: `add_role_to_users`, `create_admin_audit_log`, `add_status_to_users`
- Downgrade deve ser **sempre implementado** (mesmo que `pass` em casos irreversíveis, comentiono explicativo)

---

## Migrations iniciais (planejadas — v2.0)

| Revisão | Descrição |
|---|---|
| `0001_initial` | Schema MVP core: users, subscriptions, essays, corrections + RLS + índices |
| `0002_admin_role_and_audit` | Add `users.role`, `users.status`, `users.suspended_until`, `users.banned_reason`, `users.last_admin_action_at`; create `admin_audit_log` |
| `0003_webhook_events` | Create `webhook_events` |
| `0004_prompt_templates` | Create `prompt_templates` + unique index active |
| `0005_feature_flags` | Create `feature_flags`, `feature_flag_overrides` |
| `0006_plans_config` | Create `plans_config` e seed com free/monthly/quarterly/annual |
| `0007_email_logs` | Create `email_logs` |
| `0008_events_table` | Create `events` |
| `0009_audit_log_overall_trigger` | Trigger `validate_overall_score` em corrections |

---

## Seed (dev/staging)

`packages/db/seed.sql` deve criar:

- 1 admin user: `admin@redana.com.br` (role='admin') — senha via Supabase Studio
- 3 usuários de teste (role='user'):
  - free com 5/5 used
  - monthly ativo com 12/30 used
  - annual ativo com 30/30 used
- 8 essays distribuídas (mix status: pending, completed, failed)
- 6 corrections (mix `corrected_by = ai` e `human`)
- 2 planos da `plans_config` ativos no `prompt_templates` (1 `active=true`)

```sql
-- exemplo
insert into users (id, email, name, role) values
  ('11111111-1111-1111-1111-111111111111', 'admin@redana.com.br', 'Admin Redana', 'admin'),
  ('22222222-2222-2222-2222-222222222222', 'free.user@test.com', 'Free', 'user'),
  ('33333333-3333-3333-3333-333333333333', 'monthly@test.com', 'Monthly', 'user'),
  ('44444444-4444-4444-4444-444444444444', 'annual@test.com', 'Annual', 'user');
-- ... plano, essays, corrections
```

---

## Reset local

```bash
# Dev: drop + recreate
psql $DATABASE_URL -c "drop schema public cascade; create schema public;"
psql $DATABASE_URL -f packages/db/schema.sql
psql $DATABASE_URL -f packages/db/seed.sql
# Reset Alembic stamp
alembic stamp head
```

---

## Forward-compat

- Não usar `SELECT *` em código prod (aliasing evita quebrar com add column). SQLAlchemy usa explicit columns.
- Não renomear colunas sem migration de compat (deprecate + drop em 2 fases).
- Add nullable columns primeiro; popular via backfill; depois add NOT NULL em migration separada.