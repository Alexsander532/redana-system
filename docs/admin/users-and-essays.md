# Admin — Usuários e Redações

> Documento detalhado da gestão de usuários e redações pelo admin.

---

## Usuários

### Tela `/admin/usuarios`

Lista paginada de todos os usuários. `UserTable` (`@tanstack/react-table`).

| Coluna | Conteúdo |
|---|---|
| Email | String com link para detalhe |
| Nome | String |
| Role | Badge `user`/`admin` |
| Status | Badge `active`/`suspended`/`banned` |
| Plano | `free`/`monthly`/`quarterly`/`annual` |
| Correções usadas | `12/30` ou `5/5 (vitalício)` |
| Criado em | timestamptz short |
| Ações | Menu: Ver / Suspend / Ban / Impersonate |

### Filtros

- Search by email/name (LIKE `%query%`)
- Status filter (`active`/`suspended`/`banned`)
- Role filter (`user`/`admin`)
- Plan filter (`free`/`monthly`/`quarterly`/`annual`)
- Subscription status filter

### API

```
GET /api/v1/admin/users?limit=20&offset=0&q=...&status=...&role=...&plan=...
```

Response:
```json
{
  "items": [
    {"id":"uuid","email":"...","name":"...","role":"user","status":"active","plan":"annual","corrections_used":12,"max_corrections":30,"created_at":"..."}
  ],
  "total": 1234,
  "limit": 20,
  "offset": 0
}
```

---

### Tela `/admin/usuarios/[id]`

Detalhe de 1 usuário.

#### Seção 1: Perfil

- Avatar, nome, email (read-only), role, status, auth_provider
- Created at, last login (de auth.users), last admin action
- Botões: Suspend / Ban / Unban / Impersonate / Promover admin

#### Seção 2: Assinatura

- Plano atual, status, period_start/end, abacate_subscription_id
- Corrections usadas/max
- Link para editar assinatura (`/admin/assinaturas/[sub_id]`)

#### Seção 3: Redações (essays)

Tabela das essays do user (status, word_count, score, created_at, completed_at) — link para `/admin/redacoes/[id]`

#### Seção 4: Audit log sobre o usuário

Lista paginada das entries de `admin_audit_log WHERE target_type='user' AND target_id=uuid`.
Coluna com `payload` diff visual.

---

### Ações

#### Suspender

```
POST /api/v1/admin/users/{id}/suspend
body: { "until": "ISO8860 ou null", "reason": "texto" }
```

Efeito:
- `users.status = 'suspended'`
- `users.suspended_until = until` (null = indefinido)
- Login bloqueado (RLS/API get_current_user verifica status)
- Email "Sua conta foi suspensa" (template `subscription-suspended`-like — criar `account-suspended`)

Audit: `user.suspend`

#### Desbanir / Remover suspensão

```
POST /api/v1/admin/users/{id}/unsuspend
POST /api/v1/admin/users/{id}/unban
```

Limpa `status='active'`, `suspended_until=null`, `banned_reason=null`. Audit `user.unsuspend`/`user.unban`.

#### Banir

```
POST /api/v1/admin/users/{id}/ban
body: { "reason": "motivo" }
```

Efeito:
- `users.status = 'banned'`
- `users.banned_reason = reason`
- Refresh tokens revogados (Supabase admin API)
- Login bloqueado

#### Impersonar

```
POST /api/v1/admin/users/{id}/impersonate
→ { "token": "uuid-impersonation-token", "expires_at": "..." } (15 min)
```

Efeito:
- API emite token JWT Supabase custom signed com custom claim `impersonating:true` + `impersonated_by:admin_id` + `real_user_id:user_id` (tempo 15 min)
- Admin frontend usa `setSession(impersonationToken)` → redirect para `/dashboard` (mas com banner "Você está impersonando [user] / Sair" no topo)
- Tudo que o admin fizer nesse estado é logado em audit com `admin_id=original_admin` e `impersonating:user_id`
- Não pode mutar settings críticas (não pode deletar a conta, não pode ver danger zone), apenas visualizar e simular

> Implementação: usar **Supabase Auth Admin API** (`auth.admin.generateLink`) OU custom JWT. Detalhes pendentes de implementação — confirmar com suporte Supabase.

#### Promover/despromover role

```
POST /api/v1/admin/users/{id}/role
body: { "role": "admin" | "user" }
```

Efeito:
- `users.role = role`
- Email notificando
- Audit `user.role.change`
- Risco: se despromover role do próprio admin, fica trancado — UI bloqueia self-promotion error e despromoção self-service (`if user_id == admin.id: 400`).

---

## Redações

### Tela `/admin/redacoes`

Lista paginada de todos os essays do sistema.

| Coluna | Conteúdo |
|---|---|
| ID | Truncated uuid + tooltip |
| Aluno | Email (link para o user detail) |
| Título | String |
| Word count | N |
| Status | `pending`/`processing`/`completed`/`failed` (badge color) |
| Score overall | Se completed, int 0-1000 |
| Corrigido por | `ai`/`human` badge |
| Criado em | ts |
| Concluído em | ts |
| Ações | Ver / Reprocessar IA |

### Filtros

- status (`pending`/`processing`/`completed`/`failed`)
- user_id (via search)
- date range (created_at)
- corrected_by (`ai`/`human`)

### API

```
GET /api/v1/admin/essays?status=...&user_id=...&from=...&to=...&limit=20&offset=0
```

---

### Tela `/admin/redacoes/[id]`

Detalhe da redação.

#### Seção 1: Cabeçalho

- Title, aluno, status, created_at, completed_at, source, word_count
- Link para o usuário

#### Seção 2: Conteúdo

- Texto da redação (read-only display) - NOTA: sublime / textarea readonly. Não permite edit por LGPD.

#### Seção 3: Correção atual

- Se existe correction: mostra ScoreHeader + CompetencyBars + feedback + next_steps
- Indica `corrected_by` (ai/human) + `llm_model` + `llm_cost_brl`

#### Seção 4: Ações

- **Reprocessar IA**: re-executa correction engine. Em geral usado quando (`status='failed'`) ou para retry de erro de parse. Cria IMPLÍCITO uma nova linha correction com versão=v2? → Não. Atualiza a existente in-place, mas guarda diff no audit log.
- **Criar/Editar correção humana**: botão leva para `/admin/correcoes/[essay_id]` (ver [corrections-admin.md](./corrections-admin.md))

#### Seção 5: Audit log da essay

Entries de `admin_audit_log WHERE target_type='essay' AND target_id=essay_id`.

---

### Reprocessar IA

```
POST /api/v1/admin/essays/{id}/reprocess
```

Efeito:
1. Atualiza essay.status='processing'
2. Snapshot `before` da correction atual (para audit diff)
3. Enfileira nova execução do correction engine (LLM)
4. NOT incrementa `corrections_used` (admin action — não consome cota do user)
5. Email não enviado para o user automaticamente (admin decide se avisa)

Audit: `essay.reprocess` payload:
```json
{ "before": {"overall_score":800, "corrected_by":"ai", "model":"claude-3-5-sonnet"}, "after": "pending" }
```

Resposta:
```json
{ "status": "processing", "audit_id": "uuid" }
```

> A reprocessação substitui a correction existente em-place. Para manter históricos completos de IA attempts, considerar tabela `correction_versions` (pós-MVP).

---

## UX / Components

| Componente | Detalhe |
|---|---|
| `UserTable` | `@tanstack/react-table` server-side sorting/filter/pagination |
| `EssayTable` | idem |
| `UserDetailHeader` | Avatar + status badge + role badge + quick actions |
| `EssayDetailHeader` | Title + status badge + actions |
| `AuditLogTable` | `target_type`, `target_id`, `action`, `admin_id`, `created_at` filterable |
| `ConfirmModal` | Confirm destructive (ban, refund) |

---

## Considerações

- Impersonation deve impossibilitar que o admin, enquanto impersonando, faça mutações destrutivas. Implementar check middleware API: se `impersonating:true` é setado no JWT, qualquer `DELETE /account`, `POST /account/delete`, `POST /subscriptions/cancel` retorna 403.
- Banir mantém a conta e dados (LGPD: titular pode solicitar deleção). Hard delete só viafluxo `/account/delete` ou force pelo admin via "force-delete" (alta fricção — exige duplo modal).
- Suspend vs Ban: suspende é temporal / revogável; ban é permanente até desbanir manual.
- Não há export "todos usuários para csv" no MVP — admin usa o filtro + download de cada user via `/account/export` indireto (precisa implementar rota admin-only equivalente). Ver [admin/system-config.md](./system-config.md).