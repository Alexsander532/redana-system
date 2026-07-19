# Fase 0 — Scaffolding do monorepo

## Proposal
Inicializar o monorepo Redana com Turborepo + pnpm, migrar a landing page Astro existente e criar os packages compartilhados.

## Why
O projeto comeca como um monorepo para compartilhar tipos, design tokens, schema do banco e templates de email entre todos os apps (landing, web, admin, api). A landing page ja existe e precisa ser integrada.

## Tasks
- [ ] 0.1 Iniciar monorepo: pnpm-workspace.yaml, turbo.json, root package.json, .gitignore, .env.example
- [ ] 0.2 Criar packages/shared com types + constants (competencias, planos) + design tokens
- [ ] 0.3 Criar packages/db/schema.sql + RLS + seed
- [ ] 0.4 Criar packages/email com templates React Email
- [ ] 0.5 Migrar landing Astro para apps/landing
- [ ] 0.6 Git init, commit inicial, push para GitHub

## Dependencies
Nenhuma — fase inicial.

## Effort
~1 dia
