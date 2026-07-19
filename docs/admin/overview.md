# Admin Panel — Visão geral

> Documento detalhado do painel administrativo do Redana. Entrada de alto nível: [features/admin-panel.md](../features/admin-panel.md).

---

## O que é

O Admin Panel é o centro de controle do Redana, usado pelo **owner/operador** (role `admin`) para gerenciar todos os aspectos do sistema sem precisar tocar no banco de dados diretamente.

Antes (v1.1): listado como "pós-MVP deferrido". A partir da **v2.0**: fase legítima do produto (Fase 5), totalmente documentada.

---

## Quem usa

| Quem | Acesso |
|---|---|
| **Owner/operador** (role=admin) | Acesso total a `/admin/*` |
| User (role=user) | Sem acesso. Rota redireciona para `/dashboard` |
| User apply impersonation pelo admin | Session temporária com flag `impersonating:true` |

Rotação de admins: promover/despromover é feito via SQL direto (sem self-promotion) ou via rota `POST /admin/users/{id}/role` (admin promovendo outro admin).

---

## Onde o admin vive

> **Decisão confirmada (Julho 2026):** app Next.js 14 separado **`apps/admin`**, domínio **`admin.redana.com.br`**, deploy **Vercel** (segundo projeto no mesmo time/org).
>
> Justificativa: isolamento de auth/bundle/CORS — código admin nunca entra no bundle do aluno. A API FastAPI pode restringir `Origin: admin.redana.com.br` em `/api/v1/admin/*`. Cookies separados por domínio eliminam risco de XSS cross-app. Custo incremental baixo na escala Redana (1-2 dias para configurar o app no monorepo existente).
>
> Ver [admin-architecture-addendum.md](../admin-architecture-addendum.md) para a arquitetura completa.

---

## RBAC — Role-Based Access Control

### Schema

```sql
alter table users add column role text not null default 'user' check (role in ('user','admin'));
```

### Gates

| Rota/recursos | Role necessário |
|---|---|
| `/admin/*` (frontend) | `admin` + `status='active'` |
| `/api/v1/admin/*` | `admin` + `status='active'` |
| Audit log leitura | `admin` |
| Mutations admin (`ban`, `refund`, `prompt-edit`, ...) | `admin` + auditado |

### Flow

```mermaid
sequenceDiagram
  participant ADM as Admin
  participant W as Next.js
  participant A as FastAPI
  participant D as Postgres

  ADM->>W: POST /admin/login
  W->>W: signInWithPassword (Supabase)
  W->>W: getSession() → busca role → role='admin'
  W->>W: redirect /admin
  ADM->>A: GET /admin/users (Bearer JWT)
  A->>A: verify_jwt → get_current_admin (role check)
  A->>D: SELECT users (admin policy bypass via role check)
  D-->>A: rows
  A->>D: audit_log (READ não logado; apenas mutations são)
  A-->>ADM: 200 + paginated
```

---

## Audit log

### Tabela `admin_audit_log`

| Coluna | Conteúdo |
|---|---|
| `id` | UUID PK |
| `admin_id` | FK users.id (admin que executou) |
| `action` | Namespace e.g. `user.suspend`, `correction.human_override`, `subscription.refund` |
| `target_type` | `user` \| `essay` \| `subscription` \| `webhook_event` \| `prompt_template` \| `feature_flag` \| `plan` \| `email` |
| `target_id` | UUID do alvo |
| `payload` | JSONB com diff antes/depois + motivo |
| `ip_address` | inet do request |
| `user_agent` | string |
| `created_at` | timestamptz |

**Característica:** Append-only. RLS:
- Insert: API service role key (não via JWT)
- Select: admin only
- Update/Delete: none (RLS bloqueia)

### Decorator na API

```python
@router.post("/users/{user_id}/ban")
@audit_action("user.ban", "user")
async def ban_user(user_id: UUID, body: BanSchema,
                   admin: User = Depends(get_current_admin),
                   request: Request, db: AsyncSession = Depends(get_db)):
    user = await get_user(db, user_id)
    before = {"status": user.status, "banned_reason": user.banned_reason}
    user.status = "banned"
    user.banned_reason = body.reason
    await db.commit()
    after = {"status": "banned", "banned_reason": body.reason}
    # audit_action decorador já persiste diff antes/depois em payload + ip + ua
    return {"ok": True, "audit_id": getattr(request.state, "audit_id", None)}

# audit.py — pseudo
def audit_action(action, target_type):
    def deco(fn):
        @wraps(fn)
        async def wrapper(*args, admin, request, **kw):
            before = snapshot_target(target_type, kw)   # opcional
            result = await fn(*args, admin=admin, request=request, **kw)
            await persist_audit(db, admin, request, action, target_type, kw, before, result)
            return result
        return wrapper
    return deco
```

### Ações auditadas (não exaustivo)

| Action | Target | Quando |
|---|---|---|
| `user.suspend` | user | Suspende |
| `user.unsuspend` | user | Remove suspensão |
| `user.ban` | user | Bane |
| `user.unban` | user | Desbane |
| `user.impersonate.start` | user | Começa impersonation |
| `user.impersonate.end` | user | Encerra impersonation |
| `user.role.change` | user | Promove/rebaixa role |
| `essay.reprocess` | essay | Re-executa IA |
| `correction.human_override` | essay | Salva correção humana |
| `correction.assign` | essay | Atribui correção a admin |
| `subscription.status.force` | subscription | Força status |
| `subscription.grant.corrections` | subscription | Grant N corrections |
| `subscription.grant.plan` | subscription | Grant plano por X dias |
| `subscription.extend` | subscription | Estende period_end |
| `subscription.refund` | subscription | Refund integral |
| `webhook.replay` | webhook_event | Re-processa webhook |
| `prompt.create` | prompt_template | Novo template |
| `prompt.update` | prompt_template | Edita template |
| `prompt.activate` | prompt_template | Ativa template |
| `feature_flag.set` | feature_flag | Toggle |
| `feature_flag.override` | feature_flag + user | Override per-user |
| `plan.update` | plans_config | Edita preço/limites |
| `email.broadcast` | segment | Envia broadcast |
| `subscription.reconcile.drift` | subscription | Drift detectado pelo job |
| `user.exportdata` | user | Admin expõe dados de um user (LGPD facilitado) |

---

## Layout do painel

```mermaid
flowchart TD
  Login[/admin/login] --> Dashboard[/admin dashboard]

  Dashboard --> Users[/admin/usuarios]
  Dashboard --> Essays[/admin/redacoes]
  Dashboard --> Corrections[/admin/correcoes]
  Dashboard --> Subs[/admin/assinaturas]
  Dashboard --> Payments[/admin/pagamentos]
  Dashboard --> Analytics[/admin/metricas]
  Dashboard --> System[/admin/sistema]

  Users --> UserDetails[/admin/usuarios/id]
  Essays --> EssayDetails[/admin/redacoes/id]
  Corrections --> CorrectionForm[HumanCorrectionForm]
  Subs --> SubDetails[/admin/assinaturas/id]
  System --> Prompts[templates]
  System --> Flags[feature flags]
  System --> Pricing[plans_config]
  System --> Emails[broadcast + history]
  System --> Logs[API logs + health]
  System --> Audit[audit_log]
```

---

## Documentação detalhada

| Tema | Documento |
|---|---|
| Usuários e redações | [users-and-essays.md](./users-and-essays.md) |
| Correção humana | [corrections-admin.md](./corrections-admin.md) |
| Assinaturas, grants, refunds | [subscriptions-admin.md](./subscriptions-admin.md) |
| Analytics, charts | [analytics.md](./analytics.md) |
| Pagamentos, webhooks admin | [payments-and-webhooks.md](./payments-and-webhooks.md) |
| Prompt templates, flags, pricing, emails, logs | [system-config.md](./system-config.md) |