# Admin — Configuração do Sistema

> Documento detalhado das configurações administráveis: prompt templates, feature flags, pricing config, emails broadcast, logs, health.

---

## Tela `/admin/sistema`

Hub com sidebar de sub-seções:

- **Prompts** — Gerenciar prompt templates do LLM
- **Flags** — Feature flags on/off + override por user
- **Preços** — Editar `plans_config`
- **Emails** — Broadcast + history
- **Logs** — API logs + Sentryevents proxy
- **Health** — Status de DB / LLM / AbacatePay / Resend
- **Audit log** — Filtros sobre `admin_audit_log`
- **Variáveis / Versão** — env info + versão do deploy

---

## 1. Prompt templates

### Visão
Permite editar prompt do LLM sem redeploy. Apenas 1 template ativo por name (ver [schema.md](../data/schema.md)).

### Tela
- Lista templates: name, active (badge), created_by, updated_at
- Botões: New / Edit / Activate / Delete (soft — não delete se active)

### Editor
Componente `PromptTemplateEditor`:
- Monaco editor (lightweight via `@monaco-editor/react`)
- Placeholders suportados: `{{essay_text}}`, `{{rubric}}`, `{{competencies}}`, `{{user_language}}`
- Preview com sample essay (pós-MVP)

### API
```
GET    /api/v1/admin/prompts
POST   /api/v1/admin/prompts                                # Create
PUT    /api/v1/admin/prompts/{id}                            # Edit
POST   /api/v1/admin/prompts/{id}/activate                  # Activate (desativa outros do mesmo name)
DELETE /api/v1/admin/prompts/{id}                            # Não permitido se active
```

### Workflow
1. Admin edita prompt (cria novo ou versão) → salva como `active=false`
2. Faz test evaluation manual: botão "Testar" chama `POST /api/v1/admin/prompts/{id}/test` que roda LLM com prompt pendurado numa essay de test
3. Se satisfeito, ativa → anterior desativado (índice unique partial garante)
4. Audit `prompt.create`/`prompt.update`/`prompt.activate`

> Não há default built-in via DB: o default está hard-coded em `apps/api/app/services/prompt_templates.py` (fallback se nenhum template ativo). O primeiro template admin-create DEVE ser baseado no default.

### Constraints
- `name` é unique (mais de um template pode ter o mesmo name desde que só um active)
- `content` máx 50.000 chars
- `created_by` setado automaticamente a partir do admin autenticado
- Não pode deletar template active (API retorna 409)

---

## 2. Feature flags

### Tabela `feature_flags`

```sql
key text unique            -- 'broadcast_emails', 'human_correction', 'mp_gateway'
enabled boolean
description text
```

### Tela
Lista de flags com switch `FeatureFlagToggle` + search.

### Override per-user

`feature_flag_overrides (flag_id, user_id, enabled)`.

UX: na tela do usuário detail (`/admin/usuarios/[id]`) há um bloco "Feature flags overrides" que permite ligar/desligar flags específicas para aquele user.

### API
```
GET  /api/v1/admin/feature-flags
PUT  /api/v1/admin/feature-flags/{id}                       # Toggle global
POST /api/v1/admin/feature-flags/{id}/override              # { user_id, enabled }
```

### Flags sugeridas (v2.0)

| key | Default | Descrição |
|---|---|---|
| `broadcast_emails` | `false` | Liga/desliga envio de broadcast admin (kill switch) |
| `human_correction` | `true` | Habilita modo correção humana |
| `passwordless_login` | `false` | Magic link login (pós-MVP) |
| `mp_gateway` | `false` | Switch AbacatePay → MercadoPago (Fase 6) |
| `essay_pdf_export` | `true` | Botão "baixar PDF" no resultado |
| `maintenance_mode` | `false` | Mostra banner de manutenção no app |

### Check na API/Next.js
- API: `await is_feature_enabled(db, "human_correction")` antes de habilitar correção humana
- Next.js: server fetch `/admin/feature-flags/public` expõe flags booleans (cached em memory)

```python
async def is_feature_enabled(db, key: str, user_id: UUID | None = None) -> bool:
    flag = await db.scalar(select(FeatureFlag).where(FeatureFlag.key == key))
    if not flag:
        return False
    if user_id:
        override = await db.scalar(select(FeatureFlagOverride)
            .where(FeatureFlagOverride.flag_id == flag.id, FeatureFlagOverride.user_id == user_id))
        if override is not None:
            return override.enabled
    return flag.enabled
```

Audit `feature_flag.set` / `feature_flag.override`.

---

## 3. Pricing config (plans_config)

### Tela
Editor `PlanEditor` para cada plano (free/monthly/quarterly/annual):

| Campo | Tipo |
|---|---|
| `plan` | read-only |
| `price_cents` | int |
| `max_corrections` | int |
| `cycle_days` | int |
| `active` | boolean |

### API
```
GET  /api/v1/admin/plans
PUT  /api/v1/admin/plans/{id}
```

### Constraints
- Não pode desativar free (geraria drift global)
- Não pode setar `price_cents < 0`
- Após PUT, modificações refletem imediatamente no checkout via `abacatepay_service` (que lê plan) — atualização sync

> Mudar `price_cents` NÃO afeta assinaturas já existentes (somente novos checkouts). Para mudar preço de active sub, AbacatePay tem sua própria administering.

### Audit `plan.update`.

---

## 4. Emails — Broadcast + History

### Tela broadcast
Componente `BroadcastEmailForm`:

| Campo | Tipo |
|---|---|
| Subject | text |
| Template | select (`broadcast.tsx` ou custom HTML inline) |
| Body | rich text / HTML (Monaco) |
| Segment | select (`all_users` / `free_only` / `paid_active` / `lapsed_30d` / custom SQL via DSL pós-MVP) |
| Schedule | now or datetime |
| Unsubscribe URL | auto `https://redana.com.br/unsub?token={user_token}` |

Botão "Preview" → mostra HTML renderizado.
Botão "Send to test (admin email)" → envia só para admin.
Botão "Schedule / Send agora" → submete.

### API
```
POST /api/v1/admin/emails/broadcast
body: {
  "subject": "...",
  "template": "broadcast",
  "template_vars": {...},
  "segment": "all_users",
  "schedule_at": "ISO ou null"
}
→ 202 Accepted
{ "queued_count": 4200, "broadcast_id": "uuid" }
```

Implementação: enqueue via Background Task → percorre segment → para cada user chama `email_service.send_email`. Cada envio registrado em `email_logs` com `sent_by=<admin_id>`.

### Kill-switch
Antes de processar fila, check `is_feature_enabled('broadcast_emails')`. Se off, abortar fila com audit `email.broadcast.aborted`.

### History
Tabela paginada de `email_logs` (todos, incluindo transacionais):

| Coluna | Conteúdo |
|---|---|
| user | Email destinatário |
| template | `welcome` / `correction-ready` / `broadcast` / ... |
| subject | String |
| status | `queued`/`sent`/`failed`/`opened`/`bounce` |
| Sent by | `system` / email admin |
| Sent at | ts |

### API
```
GET /api/v1/admin/emails?template=...&status=...&from=...&to=...&limit=50&offset=0
```

### Audit `email.broadcast.send` com `payload.broadcast_id`.

---

## 5. Logs

### Visão
Proxy para Logflare (logs Supabase) ou tabela `events` filtrada.

### API
```
GET /api/v1/admin/logs?level=info,warn,error&service=api&limit=100&offset=0&q=...
```

> Implementação MVP: query em `events` table (filtrado por `event_type`LIKE 'log.%'— pendente). Para logs técnicos reais, integrar com Logflare API em pós-MVP.

### Sentry events
Sugestão: embedar link externo para Sentry dashboard (período em URL). Não tem API no MVP.

---

## 6. Health

### Tela
Cards de status por componente:

| Componente | Check |
|---|---|
| Database | `select 1` |
| Supabase Auth | ping `/health` |
| LLM Claude | ping Anthropic API `/v1/models` (ou degradado para call no fallback) |
| LLM GPT-4o | ping OpenAI API `/v1/models` |
| AbacatePay | ping AbacatePay `/v1/balance` |
| Resend | ping `/domains` |
| Better Stack | link externo |

### API
```
GET /api/v1/admin/health
→ { "db": "ok", "auth": "ok", "llm_claude": "ok", "llm_gpt4o": "degraded", "abacatepay": "ok", "resend": "ok" }
```

Refresh manual (botão) ou a cada 60s (auto-poll).

---

## 7. Audit log

### Tela `/admin/sistema/audit`

`AuditLogTable` com filtros:
- admin_id (select with search)
- action (multi-select prefix: `user.*`, `subscription.*`)
- target_type
- target_id (uuid)
- date range

### Columns
- admin_id (link para user detail)
- action
- target_type + target_id (link)
- payload (JSONB viewer)
- ip_address, user_agent
- created_at

### API
```
GET /api/v1/admin/audit-log?admin_id=...&action_prefix=user.&target_type=subscription&from=...&to=...&limit=50&offset=0
```

> Imutável: sem edit/delete. Exports to CSV (pós-MVP).

---

## 8. Variáveis / Versão (info)

Display read-only:
- API version (git SHA via `app.version` em `main.py`)
- Build time
- Lista de env vars públicas (sem secret values): `ABACATE_DEV_MODE`, `LOG_LEVEL`, `SENTRY_ENABLED`, `DASHBOARD_URL`, ...
- Link para `docs/plan.md` (versão da doc)

### API
```
GET /api/v1/admin/system/info
→ {
    "api_version": "1.0.0",
    "git_sha": "abc123",
    "build_time": "...",
    "env": "production",
    "env_vars_public": {"ABACATE_DEV_MODE": "false", ...},
    "doc_version": "v2.0"
  }
```

---

## Considerações gerais

- Todo `PUT`/`POST` nessa tela requer role `admin` (RLS + dependency) e é auditado
- Frontend faz optimistic update + revert on error
- Pricing config afeta checkout imediatamente → cuidado com edits
- Prompt template `content` é a única fonte lida pelo correction engine quando `active=true` (default fallback no código)
- Feature flags são cached em memória na API por 60s (use `lru_cache` ou simples cache dict)
- Logs API em formato structured JSON para fácil filter (ver [observability.md](../architecture/observability.md))