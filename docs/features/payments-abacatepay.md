# Pagamentos — AbacatePay

> Seção original: PLAN.md §11. Integração completa: setup, fluxo de upgrade, env vars, tela "meu plano".

---

## 11.1 Setup inicial

1. Criar conta AbacatePay → obter `ABACATE_API_KEY` + `ABACATE_WEBHOOK_SECRET`
2. Criar produtos via API ou painel:

| Produto | externalId | cycle | price (centavos) |
|---|---|---|---|
| Plano Mensal | `plan_monthly` | `MONTHLY` | 990 |
| Plano Anual | `plan_annual` | `ANNUALLY` | 8280 |

3. **Trimestral**: criado como produto `plan_quarterly` com `cycle: MONTHLY` e `price: 840`; no DB marcamos `plan = 'quarterly'` e `period_end = now() + 3 months`. **Solução mais limpa sem arremedo.**
4. Registrar webhook: `POST /webhooks/create` com endpoint `https://api.redana.com.br/api/v1/webhooks/abacatepay` e eventos:
   - `subscription.completed, subscription.renewed, subscription.cancelled, checkout.completed`

Mais detalhes do handler: [webhooks-abacatepay.md](../api/webhooks-abacatepay.md).

---

## 11.2 Fluxo de upgrade (Free → Pago)

```
Usuário clica "Assinar Anual" no dashboard
↓
POST /subscriptions/checkout { plan: 'annual' }
↓
API cria checkout AbacatePay: POST /checkouts/create
  items: [{ id: product_annual.id, quantity: 1 }]
  frequency: SUBSCRIPTION
  returnUrl: https://app.redana.com.br/planos?status=success
  completionUrl: https://app.redana.com.br/dashboard
↓
Retorna { url } → frontend redirect
↓
Cliente paga no checkout AbacatePay (Cartão ou PIX)
↓
AbacatePay dispara webhook subscription.completed
↓
API: valida HMAC, encontra user, atualiza subscriptions:
  plan = 'annual', status = 'active',
  abacate_subscription_id = <id>,
  max_corrections = 30,
  period_start = now, period_end = now + 365d
↓
Envia email "Assinatura ativa" (Resend)
```

### Diagrama

```mermaid
sequenceDiagram
  participant U as Usuário
  participant W as Next.js
  participant A as FastAPI
  participant AB as AbacatePay
  participant DB as Postgres
  participant R as Resend

  U->>W: Clica "Assinar Anual"
  W->>A: POST /subscriptions/checkout {plan:annual}
  A->>AB: POST /checkouts/create
  AB-->>A: {checkout_url}
  A-->>W: {url}
  W-->>U: Redirect checkout
  U->>AB: Paga (PIX/Cartão)
  AB->>A: webhook subscription.completed (HMAC)
  A->>A: Verifica HMAC
  A->>DB: UPDATE subscriptions SET plan=annual, max_corrections=30, period_end=now()+365d
  A->>R: send welcome-paid email
  U-​-​>>W: Redirect returnUrl ?status=success
  W->>W: Shows CurrentPlanBadge
```

---

## 11.3 Handlers webhook

Ver [webhooks-abacatepay.md](../api/webhooks-abacatepay.md) para o deep-dive.

| Evento | Ação |
|---|---|
| `subscription.completed` | Ativa plano, define max_corrections, period_start/end |
| `subscription.renewed` | Renova período (`period_end` += 1 ciclo), resetar `corrections_used = 0` |
| `subscription.cancelled` | `status = 'canceled'`, mantém ativos até `period_end`; depois free |
| `checkout.completed` (ONE_TIME fallback) | Marca pagamento avulso (se usarmos trimestral via checkout avulso) |
| `subscription.refunded` | Downgrade imediato para free |

---

## 11.4 Variáveis de ambiente

```
ABACATE_API_KEY=abk_live_xxx
ABACATE_WEBHOOK_SECRET=whsec_xxx
ABACATE_PRODUCT_MONTHLY_ID=prod_xxx
ABACATE_PRODUCT_ANNUAL_ID=prod_xxx
ABACATE_PRODUCT_QUARTERLY_ID=prod_xxx
ABACATE_DEV_MODE=true  # sandbox
```

Todas via `.env` Railway ou `vercel env` para front (somente `ABACATE_DEV_MODE` é público; o secret só no Railway).

---

## 11.5 Tela "Meu plano" (substituto do portal)

Como AbacatePay não tem customer portal de auto-serviço, implementamos:

- **Card de plano atual**: nome, preço, próxima cobrança (`period_end`)
- **Botão Cancelar assinatura** → `POST /subscriptions/cancel` → API chama AbacatePay `POST /subscriptions/cancel`
  - Após cancelar, `subscriptions.status = 'canceled'`, `max_corrections` mantém até `period_end`, depois cai para 5 (free)
- **Botão Trocar plano** → abre modal com `PricingCards` → novo checkout
  - Troca IMEDIATA não é suportada no MVP (evita pro-rata complexo): o novo plano começa no fim do ciclo atual, e o usuário paga hoje? Recomenda-se: cancelar e deixar o plano expirar para assinar novo.
  - Atalho v2.0: o admin pode fazer a troca manual via [admin/subscriptions-admin.md](../../admin/subscriptions-admin.md) fora do cliente.
- **Botão Atualizar cartão** → instrutivo: "Cancele e reative com novo cartão" (limitação da AbacatePay)

### Layout sugerido

```
┌─────────────────────────────────────┐
│ Plano Atual                          │
│ ┌─────────────────────────────────┐  │
│ │ ANUAL — R$ 82,80 / ano          │  │
│ │ Próx. cobrança: 18/07/2027      │  │
│ │ Correções usadas: 12 / 30       │  │
│ └─────────────────────────────────┘  │
│ [Trocar plano] [Cancelar assinatura] │
│                                       │
│ Atualizar cartão →                    │
│  Para mudar o cartão, cancele e       │
│  reative sua assinatura com o novo    │
│  cartão. (limitação AbacatePay)       │
└─────────────────────────────────────┘
```

---

## 11.6 Resposta do checkout (exemplo)

`POST /api/v1/subscriptions/checkout` → 200:

```json
{
  "url": "https://api.abacatepay.com/checkout/abc123...",
  "subscription_preview": {
    "plan": "annual",
    "max_corrections": 30,
    "price_cents": 8280
  }
}
```

Frontend:

```ts
const { url } = await api.post("/subscriptions/checkout", { plan: "annual" });
window.location.href = url;
```

---

## Plug removível

`abacatepay_service.py` é a única superfície de contato com AbacatePay. Toda lógica de cobrança está atrás dessa classe. Para trocar gateway:

1. Implementar interface `PaymentGateway` em `payment_service_new.py`
2. Trocar DI em `dependencies.py`
3. Reutilizar webhook handlers com novo secret

> Cobertura de testes de webhook é essencial para garantir que troca futura não quebre estado. Ver [webhooks-abacatepay.md](../api/webhooks-abacatepay.md).