# Emails — Resend

> Documento NOVO (v2.0). Integração Resend, templates React Email, pontos de gatilho.

---

## Stack

- **Resend** — provedor de email transacional (entrega, webhooks de bounce/open)
- **React Email** — authoring de templates como componentes React
- Templates ficam em `packages/email/templates/`
- Envio feito por `apps/api/app/services/email_service.py` (Resend SDK Python)

> Resend permite escolher Python SDK ou Node. Optamos pelo Python API para centralizar na API.

---

## Variáveis de ambiente

```
RESEND_API_KEY=re_xxx
RESEND_FROM_ADDRESS="Redana <no-reply@redana.com.br>"
RESEND_REPLY_TO=contato@redana.com.br
RESEND_WEBHOOK_SECRET=whsec_xxx        # para webhook de bounce/open (opcional)
```

---

## Templates (MVP core)

| Template | Arquivo | Quando | Assunto |
|---|---|---|---|
| Welcome | `welcome.tsx` | Após `signup` | "Bem-vindo ao Redana!" |
| Correção pronta | `correction-ready.tsx` | Após `correction.completed` | "Sua redação foi corrigida!" |
| Reset senha | `reset-password.tsx` | `/auth/recuperar` | "Redefina sua senha" |
| Lembrete upgrade | `upgrade-reminder.tsx` | Sub atingiu limite OU trial expirou | "Você atingiu seu limite. Assine e continue evoluindo." |

### Templates adicionais (v2.0)

| Template | Quando |
|---|---|
| `subscription-active.tsx` | Webhook `subscription.completed` |
| `subscription-canceled.tsx` | Webhook `subscription.cancelled` |
| `subscription-renewed.tsx` | Webhook `subscription.renewed` |
| `payment-failed.tsx` | Sub vai para `past_due` |
| `broadcast.tsx` | Email broadcast do admin (ver [admin/system-config.md](../../admin/system-config.md)) |

---

## Pontos de gatilho

| Evento | Template | Em qual serviço |
|---|---|---|
| `auth.signup` | welcome | `auth_service.sign_up` |
| `auth.password_reset` | reset-password | `auth_service.request_reset` |
| `correction.completed` (AI) | correction-ready | `correction_engine.process` |
| `subscription.completed` (webhook) | subscription-active | `webhooks_abacatepay.handle_subscription_completed` |
| `subscription.cancelled` (webhook) | subscription-canceled | `webhooks_abacatepay.handle_subscription_cancelled` |
| `subscription.renewed` (webhook) | subscription-renewed | `webhooks_abacatepay.handle_subscription_renewed` |
| `subscription.past_due` (job) | payment-failed | `subscription_service.reconcile_subscriptions` |
| `subscription.expired` (job downgrade) | upgrade-reminder | `subscription_service.downgrade_to_free` |
| `essay.failed` (LLM error) | (nenhum email) | toast no app |

---

## Service (Python)

```python
# apps/api/app/services/email_service.py
import resend

resend.api_key = settings.RESEND_API_KEY

async def send_email(to: str, template: str, context: dict) -> str:
    html = render_template(template, context)
    resp = resend.Emails.send(
        from_=settings.RESEND_FROM_ADDRESS,
        to=to,
        subject=context["subject"],
        html=html,
        reply_to=settings.RESEND_REPLY_TO,
    )
    # Persist email_log
    await email_log_repo.create(template_name=template, subject=..., status="sent", metadata={"resend_id": resp["id"]})
    return resp["id"]
```

> Templates React Email são compilados em HTML uma vez (build step do `packages/email`) e importados na API via filesystem read, OU: a API recebe HTML pré-renderizado via admin broadcast. Alternativa: usar API do Resend com template ID criado no painel Resend.

### Alternativa simples (MVP)

Em vez de manter build-step de React Email → HTML, regenar HTML no Next.js (admin broadcast) ou usar HTML template string em Python. Decisão pendente (preferência por React Email para consistência).

---

## Persistence (`email_logs`)

Cada envio registra:

| Campo | Conteúdo |
|---|---|
| `user_id` | Destinatário |
| `template_name` | Nome do template enviado |
| `subject` | Assunto |
| `status` | `queued` \| `sent` \| `failed` \| `opened` \| `bounce` |
| `metadata` | `{resend_id, ...}` |
| `sent_by` | admin id (se broadcast) ou null (sistema) |
| `sent_at` | timestamp |

→ Visível no admin em [admin/system-config.md](../../admin/system-config.md).

---

## Webhook Resend (bounce/open)

`POST /api/v1/webhooks/resend` opcional:

- Eventos: `email.opened`, `email.bounced`, `email.delivery`
- Atualiza `email_logs.status` correspondente
- HMAC verify (via `RESEND_WEBHOOK_SECRET`)

Pós-MVP, mas a coluna `status` já está pronta para isso.

---

## Considerações

| Item | Detalhe |
|---|---|
| SPF/DKIM/DMARC | Configurar no DNS (vercel/Cloudflare) para `redana.com.br` |
| Reply-to | `contato@redana.com.br` configurado como DPO no rodapé (LGPD) |
| Unsubscribe | Para emails transacionais não é obrigatório, mas `broadcast` exige `unsubscribe_url` no rodapé |
| Limite de envio | Free tier Resend: 3.000/mês, 100/dia — suficiente no MVP |
| Supressão | Resend suprime bounces; admin tem vista de bounce em [admin/system-config.md](../../admin/system-config.md) |