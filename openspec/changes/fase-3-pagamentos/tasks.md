# Fase 3 — Pagamentos AbacatePay

## Proposal
Integrar AbacatePay como gateway de pagamentos: criar produtos (Monthly, Annual, Quarterly), implementar checkout, webhook handler com HMAC, tela "Meu plano" no dashboard e upgrade flow.

## Why
Monetizacao e essencial para o negocio. Sem pagamentos, o produto nao gera receita. AbacatePay foi escolhido por ser gateway brasileiro com PIX nativo e API de assinaturas.

## Tasks
- [ ] 3.1 Criar conta AbacatePay + produtos (Monthly, Annual, Quarterly)
- [ ] 3.2 abacatepay_service.py (SDK Python: criar checkout, cancel assinatura, list)
- [ ] 3.3 subscriptions router: checkout/cancel/change-plan
- [ ] 3.4 Webhook handler com HMAC verify + handlers de eventos
- [ ] 3.5 Job de reconciliacao diario (sync subscription status via API list)
- [ ] 3.6 Front /planos com PricingCards + integracao checkout
- [ ] 3.7 UpgradeBanner quando free tier esgota
- [ ] 3.8 Tela /conta com status, cancelar, trocar, exportar (LGPD)

## Dependencies
- Fase 2 (correcao IA — rate limiter integra com subscriptions)
- Conta AbacatePay criada

## Effort
~4-5 dias
