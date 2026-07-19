# Definition of Done (MVP+Admin)

> Seção original: PLAN.md §16. Checklist estendido com items do Admin Panel (v2.0).

---

## MVP core (Fases 0-4)

- [ ] Cadastro/login funcional (email + Google)
- [ ] Free user envia redação (texto ou .txt/.docx) e recebe correção em < 5 min
- [ ] Correção tem nota geral + notas C1-C5 + feedback por competência + próximos passos
- [ ] Dashboard mostra evolução e histórico
- [ ] Free tier respeita 5 correções vitalícias; bloqueia e oferece upgrade
- [ ] Upgrade para Mensal/Trimestral/Anual via AbacatePay (checkout hospedado)
- [ ] Webhook AbacatePay sincroniza status da assinatura
- [ ] Planos pagos têm 30 correções/mês (não unlimited)
- [ ] Usuário pode cancelar via tela /conta
- [ ] Landing links para app signup
- [ ] Responsivo 320-1440, visual consistente com a landing
- [ ] Rotas protegidas redirecionam para /login
- [ ] Emails: welcome, correção-pronta, reset
- [ ] RLS em todas as tabelas com user_id
- [ ] LGPD: política + exportar + excluir conta + retenção 12m

---

## Admin Panel (Fase 5) — v2.0

### Auth e RBAC
- [ ] Coluna `users.role` presente, default `'user'`
- [ ] Login admin separado `/admin/login` (cookie isolado)
- [ ] Middleware protege `/admin/*` (role check)
- [ ] API `get_current_admin` valida role + status active
- [ ] RLS policies admin-all em todas tabelas relevantes

### Audit log
- [ ] Tabela `admin_audit_log` imutável (sem update/delete via RLS)
- [ ] Toda mutação admin (`suspend`, `ban`, `grant-corrections`, `refund`, `prompt-template-edit`, `flag-toggle`, `plan-edit`, `broadcast-send`) é registrada
- [ ] Audit log view com filtros (admin_id, action, target_type, date range)
- [ ] `payload` inclui diff antes/depois

### Users & essays
- [ ] `/admin/usuarios`: listagem paginada com search, filtro por status
- [ ] `/admin/usuarios/[id]`: perfil do user + subscription + essays recentes + audit actions sobre o user
- [ ] Suspend/unsuspend/ban/unban funcionais (com audit log)
- [ ] Impersonation funcional (token curto, auditado)
- [ ] `/admin/redacoes`: listagem paginada com filtros (status, user_id, data)
- [ ] `/admin/redacoes/[id]`: content + correction + `reprocess` IA button
- [ ] `reprocess` re-executa o LLM e atualiza correction

### Corrections humanas
- [ ] `/admin/correcoes` fila de redações para correção humana (status filtros)
- [ ] `HumanCorrectionForm` valida soma C1+C2+C3+C4+C5 = overall
- [ ] Salvar correction humana seta `corrected_by='human'`, `llm_model=null`, `llm_cost_brl=0`
- [ ] Assign de redação a admin funciona
- [ ] Diff vs AI previous é exibido após salvar

### Subscriptions admin
- [ ] `/admin/assinaturas`: listagem paginada (status, plan)
- [ ] Forçar mudança de status (active, past_due, canceled, expired)
- [ ] Grant N corrections (incrementa `max_corrections`)
- [ ] Grant plano por X dias (sem AbacatePay)
- [ ] Extend `period_end`
- [ ] Refund integral chama AbacatePay + downgrade imediato

### Payments & webhooks
- [ ] `/admin/pagamentos`: dashboard de pagamentos falhados (subs `past_due`)
- [ ] `/admin/webhooks`: lista de `webhook_events` com raw payload
- [ ] Replay de evento webhook (re-processar)
- [ ] Drift de reconciliação visível

### Analytics
- [ ] `/admin/metricas` mostra KPIs (MRR, ativos, conversão, latência, custo/dia, falhas)
- [ ] MRR chart 30/90 dias
- [ ] Custo LLM chart (sum e avg por dia)
- [ ] Correções/dia chart
- [ ] Conversão Free → Pago chart
- [ ] Latência média + taxa de falha

### System config
- [ ] `/admin/sistema/prompts`: editor de prompt templates, ativar/desativar
- [ ] `/admin/sistema/features`: feature flags toggle (on/off, override por user)
- [ ] `/admin/sistema/precos`: editar `plans_config` (price, max_corrections, cycle_days)
- [ ] `/admin/sistema/emails`: broadcast form + history (`email_logs`)
- [ ] `/admin/sistema/logs`: API logs com nível/serviço/limit
- [ ] `/admin/sistema/health`: healthcheck estendido (DB, LLM, AbacatePay)

### UX
- [ ] Layout responsivo 1280+ (admin é desktop-first)
- [ ] Confirmação modal paradestructive actions (ban, refund, broadcast)
- [ ] Toast em todas mutations
- [ ] Empty/loading states em todas tabelas
- [ ] Filtros persistem na URL (back/forward funciona)

### Testes
- [ ] E2E: admin login → suspend user → impersonate → reprocess essay → human correction → grant corrections → refund → broadcast email
- [ ] Unit tests audit decorator (cobertura actions)
- [ ] Unit tests RBAC (user não-admin chamando /admin → 403)
- [ ] Integration tests webhook replay (idempotência)