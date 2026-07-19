# Fase 2 — Submissao + Correcao IA

## Proposal
Implementar o fluxo completo de envio de redacao e correcao por IA: endpoints CRUD de essays, engine de correcao (Claude + GPT-4o fallback), rate limiter, UI de submissao e visualizacao de resultados no dashboard.

## Why
Este e o core do produto. A correcao por IA e o que entrega valor ao usuario. Sem esta fase, o Redana nao funciona como produto.

## Tasks
- [ ] 2.1 API: POST /essays, GET /essays, GET /essays/{id}, DEL /essays/{id}
- [ ] 2.2 Prompt templates ENEM (C1-C5) com schema JSON de saida
- [ ] 2.3 CorrectionEngine: chamada Claude + fallback GPT-4o + validacao + retry
- [ ] 2.4 Rate limiter: verificar corrections_used < max_corrections
- [ ] 2.5 Background task: enqueue + status pending→processing→completed
- [ ] 2.6 Front EssayForm (texto + upload .txt/.docx) + FreeTierCounter
- [ ] 2.7 Front EssayHistory na home do dashboard
- [ ] 2.8 Front redacoes/[id]: ScoreHeader + CompetencyBars + NextSteps
- [ ] 2.9 Front EvolutionChart (Recharts) na home

## Dependencies
- Fase 1 (auth + shell)
- Prompt templates prontos

## Effort
~5-6 dias
