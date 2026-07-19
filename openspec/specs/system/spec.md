# System — Arquitetura, seguranca e LGPD

## Purpose
Define os requisitos de arquitetura do sistema, stack tecnologica, seguranca, LGPD e observabilidade.

## Requirements

### Requirement: Monorepo Turborepo + pnpm
O projeto SHALL ser organizado como monorepo com Turborepo + pnpm, contendo: apps/landing (Astro), apps/web (Next.js aluno), apps/admin (Next.js admin, v2.0), apps/api (FastAPI), packages/shared, packages/db, packages/email.

#### Scenario: Build do monorepo
- **WHEN** `pnpm run build` executado na raiz
- **THEN** Turborepo orquestra build paralelo de todos os apps e packages

### Requirement: Subdominios
O sistema SHALL usar subdominios: redana.com.br (landing), app.redana.com.br (dashboard aluno), api.redana.com.br (FastAPI), admin.redana.com.br (painel admin, v2.0).

#### Scenario: Trafego roteado por subdominio
- **WHEN** request chega em api.redana.com.br
- **THEN** FastAPI processa, CORS permite Origins: app.redana.com.br, admin.redana.com.br

### Requirement: Stack tecnologica
O sistema SHALL utilizar a seguinte stack tecnologica em todas as camadas do monorepo.

| Camada | Tecnologia |
|---|---|
| Landing | Astro 5 |
| Dashboard aluno | Next.js 14 (App Router) |
| Dashboard admin | Next.js 14 (App Router, dominio separado) |
| API | Python FastAPI 0.110+ |
| Auth | Supabase Auth (PKCE, email + Google OAuth) |
| Database | Supabase PostgreSQL + RLS |
| ORM | SQLAlchemy 2.0 async + Alembic |
| LLM | Claude 3.5 Sonnet (primario) + GPT-4o (fallback) |
| Pagamentos | AbacatePay (PIX + Cartao, MONTHLY/ANNUALLY, webhooks HMAC) |
| Email | Resend + React Email |
| Deploy Front | Vercel |
| Deploy API | Railway |
| Monorepo | Turborepo + pnpm |

#### Scenario: Novo desenvolvedor onboard
- **WHEN** novo dev consulta a documentacao de arquitetura
- **THEN** stack completa listada no spec.md para referencia

### Requirement: Seguranca
O sistema SHALL: todas secrets em env vars (nunca em codigo), CORS restrito por origem, rate limiting (60 req/min por IP, 5 essays/h por user), validacao Pydantic em todos inputs, HMAC verification em webhooks, RLS em todas tabelas com user_id.

#### Scenario: Ataque de forca bruta
- **WHEN** IP excede 60 requisicoes/minuto
- **THEN** HTTP 429 retornado por slowapi middleware

### Requirement: LGPD
O sistema SHALL: consentimento do titular como base legal, Politica de Privacidade + Termos no rodape, retencao de redacao por 12 meses (depois anonimizada), exportacao de dados via POST /account/export, exclusao de conta via POST /account/delete com grace period 30 dias.

#### Scenario: Usuario solicita exportacao
- **WHEN** usuario clica "Exportar meus dados" na tela Conta
- **THEN** JSON gerado com essays, corrections, subscription; download iniciado

#### Scenario: Usuario solicita exclusao
- **WHEN** usuario clica "Excluir minha conta"
- **THEN** conta marcada para delecao, grace period 30 dias, hard delete apos (cascade)

### Requirement: Observabilidade
O sistema SHALL registrar: erros via Sentry (API + Web), logs de runtime via Logflare (Supabase), metricas de negocio em tabela events (MRR, conversao, custo LLM), health check em GET /health.

#### Scenario: Erro na API
- **WHEN** excecao nao tratada ocorre no FastAPI
- **THEN** erro capturado pelo Sentry, stack trace disponivel

### Requirement: Migrations versionadas
O schema do banco SHALL ser versionado via Alembic, com migrations aplicadas no Supabase PostgreSQL. Ver docs/data/schema.md para DDL completo.

#### Scenario: Nova migration aplicada
- **WHEN** `alembic upgrade head` executado
- **THEN** schema atualizado, tabelas/colunas novas criadas, RLS policies atualizadas
