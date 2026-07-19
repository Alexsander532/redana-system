# Visão geral e objetivos

> Seção original: PLAN.md §1. Documento de produto — o que é Redana, metas do MVP, fora de escopo e glossário de produto.

---

## O que é

Redana é uma plataforma brasileira onde estudantes enviam suas redações do ENEM e recebem, em minutos, uma correção detalhada organizada pelas **5 competências oficiais** (C1 a C5): nota estimada por competência, feedback qualitativo e um plano de melhoria para a próxima redação.

O produto atende a três públicos:

- **Estudantes** — dashboard Next.js (`app.redana.com.br`): submetem redações, consomem correções, acompanham evolução, assinam planos.
- **Administrador (owner/operador)** — painel admin (`app.redana.com.br/admin`): controla usuários, redações, correções, assinaturas, pagamentos, métricas e configurações do sistema. Documentação completa em [`admin/`](../admin/overview.md).
- **Visitantes** — landing Astro (`redana.com.br`): conhecer o produto e cadastrar-se.

---

## Metas do produto (MVP)

1. **Tempo-to-value:** recém-cadastrado consiga corrigir sua primeira redação em menos de 5 minutos (desde o signup até ver a correção).
2. **Free tier generoso:** 5 correções vitalícias (sem cartão) — prova de valor antes da venda.
3. **Planos pagos recorrentes:** Mensal R$ 9,90, Trimestral R$ 25,20 (15% off), Anual R$ 82,80 (30% off).
4. **Evolução visível:** Dashboard de evolução por competência ao longo das redações (gráfico Recharts).
5. **Margem saudável:** Mesmo no plano Anual, custo por correção abaixo do preço por mês dividido por uso médio.
6. **Painel admin operável:** O operador consegue gerir todo o sistema sem tocar no banco (ver [`admin/`](../admin/overview.md)).

---

## Métricas de sucesso

| Métrica | Meta MVP | Como medir |
|---|---|---|
| Tempo até primeira correção | < 5 min | Tabela `events` no DB (`signup` → `correction_completed`) |
| Conversão Free → Pago | > 5% mensal | Query em `subscriptions` (status `active` / `free`) |
| Latência média de correção | 20-40 s | Coluna `essays.completed_at - created_at` |
| Custo LRM por correção | ≤ R$ 0,05 | Coluna `corrections.llm_cost_brl` |
| Margem no plano Anual | ≥ 80% | Receita R$ 6,90/mês vs custo LLM |
| Retenção D30 | > 30% | Cohort em `essays.created_at` |

Ver [`admin/analytics.md`](../admin/analytics.md) para o dashboard de métricas.

---

## Fora do escopo (MVP)

### Pós-MVP / Fase 5+
- ~~Painel admin + correção humana~~ → **agora incluído como Fase 5 legítima** (`docs/admin/`).
- MercadoPago/Pix avulso além do que AbacatePay oferece → Fase 6.
- Tema-deteccão automática da redação → **aberto** (ver [risks-and-open-questions.md](../process/risks-and-open-questions.md)).

### Fora do MVP (provavelmente nunca)
- App mobile nativo — web responsivo cobre.
- PWA offline — correção depende de LLM cloud.
- Suporte a PDF/OCR de redações manuscritas.
- Notificações push.

---

## Glossário de produto

| Termo | Definição |
|---|---|
| **Redação** | Texto enviado pelo estudante para correção. |
| **Correção** | Avaliação com notas C1-C5 + feedback. |
| **Competências C1-C5** | As 5 competências oficiais do ENEM (max 200 pts cada). |
| **Free tier** | 5 correções vitalícias grátis. |
| **Ciclo** | Período da assinatura (30/90/365 dias). |
| **Limite por ciclo** | 30 correções por ciclo nos planos pagos. |
| **Admin** | Operador do painel administrativo (role distinta). |
| **Correção humana** | Override manual do admin sobre a IA. |
| **Upload** | Anexo `.txt`/`.docx` no lugar de texto colado. |

Glossário técnico completo em [`plan.md`](../plan.md).