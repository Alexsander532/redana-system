# Escopo do MVP (in/out)

> Seção original: PLAN.md §3. Lista clara do que entra e do que não entra no MVP.

---

## ✅ Dentro do MVP

1. Cadastro/login (email + Google OAuth via Supabase)
2. Envio de redação por texto colado **OU** upload `.txt`/`.docx`
3. Correção por IA: notas C1-C5, nota geral, feedback por competência, plano de próximos passos
4. Dashboard: histórico de redações, gráfico de evolução, barra de uso (Free)
5. Assinaturas via AbacatePay (Mensal, Trimestral, Anual) — chega de Free é só upgrade manual
6. Webhook AbacatePay para sincronizar status de assinatura
7. Cancelamento pelo usuário via portal/botão → downgrade para Free ao fim do ciclo
8. Emails transacionais (Resend): welcome, redação corrigida, redefinição de senha
9. LGPD básico: política, exportar meus dados, excluir conta
10. Responsivo 320px–1440px

> **Atualização v2.0:** A Fase 5 (Admin Panel + Correção Humana) não é mais "fora do MVP" puro — passou a fase legítima (não mais deferrida). Ver [phasing.md](../process/phasing.md). O MVP "core" (Fases 0-4) continua o escopo mínimo lançável.

---

## ❌ Fora do MVP (core)

| Item | Para quando |
|---|---|
| Painel admin + correção humana | ~~Pós-MVP~~ → **Fase 5** (legítima, faz parte do produto) |
| MercadoPago paralelo | Fase 6 (pós-MVP) |
| Upload de PDF com OCR | Pós-MVP, sem data |
| PWA offline | Pós-MVP, sem data |
| Tema-deteccção automática | Aberto — ver [risks-and-open-questions.md](../process/risks-and-open-questions.md) |
| Notificações push | Pós-MVP, sem data |
| Programmatic SEO (camadas/CM campanhas) | Pós-MVP, sem data |
| Exportação PDF do resultado | Pendente — ver pergunta aberta #6 |
| Magic link login | Pós-MVP |
| Antivírus escaneamento de upload | Pós-MVP |
| Dark mode | Aberto (pergunta #4) |

---

## Onde cada item está documentado

| Item | Documentação |
|---|---|
| Cadastro/login | [auth-and-authorization.md](../api/auth-and-authorization.md), [dashboard-routes.md](../frontend/dashboard-routes.md) |
| Envio de redação | [essay-submission.md](../features/essay-submission.md) |
| Correção por IA | [correction-engine.md](../features/correction-engine.md) |
| Dashboard + evolução | [dashboard-routes.md](../frontend/dashboard-routes.md), [ui-components.md](../frontend/ui-components.md) |
| Assinaturas AbacatePay | [payments-abacatepay.md](../features/payments-abacatepay.md), [subscriptions.md](../features/subscriptions.md) |
| Webhook AbacatePay | [webhooks-abacatepay.md](../api/webhooks-abacatepay.md) |
| Cancelamento | [subscriptions.md](../features/subscriptions.md) |
| Emails | [emails.md](../features/emails.md) |
| LGPD | [lgpd-compliance.md](../features/lgpd-compliance.md) |
| Responsividade | [ui-components.md](../frontend/ui-components.md), [definition-of-done.md](../process/definition-of-done.md) |
| **Admin panel** | [**admin-panel.md**](../features/admin-panel.md) + [`admin/`](../admin/overview.md) |