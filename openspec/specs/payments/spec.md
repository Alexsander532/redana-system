# Payments — Assinaturas e pagamentos via AbacatePay

## Purpose
Define os requisitos de monetizacao via AbacatePay, incluindo planos de assinatura, checkout, webhooks e gestao de assinaturas.

## Requirements

### Requirement: Planos de assinatura
O sistema SHALL oferecer 4 planos: Free (5 corr/vitalicias), Mensal (R$9,90, 30 corr/ciclo), Trimestral (R$8,40/mes, 15% off, 30 corr/ciclo), Anual (R$6,90/mes, 30% off, 30 corr/ciclo).

#### Scenario: Usuario visualiza precos
- **WHEN** usuario acessa /planos no dashboard ou ve landing page
- **THEN** 4 cards exibidos com preco mensal equivalente, beneficios e CTA

### Requirement: Checkout AbacatePay
O sistema SHALL criar checkout via AbacatePay para upgrades de plano, suportando ciclos MONTHLY e ANNUALLY. Trimestral usa `cycle: MONTHLY` com metadata `plan=quarterly` e `period_end: now + 90d`.

#### Scenario: Upgrade Free para Anual
- **WHEN** usuario clica "Assinar Anual"
- **THEN** POST /subscriptions/checkout → AbacatePay cria checkout → usuario redirecionado para pagina de pagamento AbacatePay

#### Scenario: Pagamento aprovado
- **WHEN** AbacatePay processa pagamento com sucesso
- **THEN** webhook subscription.completed → API atualiza subscription: plan=annual, status=active, max_corrections=30, period_start=now, period_end=now+365d

### Requirement: Webhooks AbacatePay
O sistema SHALL processar webhooks AbacatePay com verificacao HMAC de assinatura, tratando eventos: subscription.completed, subscription.renewed, subscription.cancelled, checkout.completed.

#### Scenario: Webhook com assinatura invalida
- **WHEN** webhook chega sem HMAC ou com assinatura incorreta
- **THEN** 401, evento nao processado

#### Scenario: Renovacao de assinatura
- **WHEN** webhook subscription.renewed chega
- **THEN** period_end += 1 ciclo, corrections_used = 0

#### Scenario: Cancelamento de assinatura
- **WHEN** webhook subscription.cancelled chega
- **THEN** status = canceled, acesso mantido ate period_end, depois downgrade para free

### Requirement: Tela "Meu plano"
O sistema SHALL exibir status da assinatura no dashboard do usuario: plano atual, correcoes usadas/restantes, proxima cobranca, botoes Cancelar e Trocar plano.

#### Scenario: Usuario cancela assinatura
- **WHEN** usuario clica "Cancelar assinatura" na tela Conta
- **THEN** POST /subscriptions/cancel → AbacatePay cancela → status atualizado

### Requirement: Reconciliacao de webhooks
O sistema SHALL executar job diario de reconciliacao comparando status de assinaturas no DB com estado real no AbacatePay via API.

#### Scenario: Webhook perdido detectado
- **WHEN** job diario encontra subscription paid no AbacatePay mas pending no DB
- **THEN** status corrigido, audit log registrado

### Requirement: Grants manuais (v2.0 admin)
O sistema SHALL permitir que admin conceda correcoes extras, estenda periodos de assinatura ou conceda planos manualmente, com audit log obrigatorio.

#### Scenario: Admin concede 10 correcoes extras
- **WHEN** admin acessa assinatura do usuario e adiciona +10 max_corrections
- **THEN** contador atualizado, audit log registra grant
