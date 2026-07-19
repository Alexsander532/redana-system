# Fase 1 — Infra + Auth + Shell

## Proposal
Configurar Supabase (Auth + DB), aplicar o schema inicial, criar o backend FastAPI com rotas de auth e o shell do dashboard Next.js para o aluno.

## Why
A infraestrutura de auth e o shell do dashboard sao pre-requisitos para todas as features seguintes. Sem auth, nao ha como associar redacoes a usuarios.

## Tasks
- [ ] 1.1 Criar projeto Supabase, configurar auth providers (email+Google), obter URL + anon key
- [ ] 1.2 Aplicar schema.sql no Supabase + RLS policies
- [ ] 1.3 Scaffolding apps/api FastAPI: main, config, db, alembic, /health
- [ ] 1.4 Auth router: signup/login/me com verify Supabase JWT
- [ ] 1.5 Scaffolding apps/web Next.js 14 + Tailwind + configs de design tokens
- [ ] 1.6 Supabase clients (browser + server) + middleware de auth
- [ ] 1.7 Telas de login + signup (email + Google)
- [ ] 1.8 Shell do dashboard: Sidebar + Topbar + UserMenu + responsivo

## Dependencies
- Fase 0 (monorepo)

## Effort
~3-4 dias
