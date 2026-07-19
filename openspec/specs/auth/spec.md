# Auth — Autenticacao e autorizacao

## Purpose
Define os requisitos de autenticacao e autorizacao do Redana, incluindo cadastro, login, protecao de rotas e controle de acesso baseado em roles (user/admin).

## Requirements

### Requirement: Cadastro de usuario
O sistema SHALL permitir que usuarios se cadastrem com email + senha (min 8 chars) ou Google OAuth.

#### Scenario: Cadastro com email
- **WHEN** usuario submete email + senha valida via `POST /auth/signup`
- **THEN** conta criada no Supabase Auth, subscription padrao `free` criada, email de welcome enviado

#### Scenario: Cadastro com Google OAuth
- **WHEN** usuario clica "Continuar com Google"
- **THEN** login OAuth via Supabase, conta criada se novo usuario

### Requirement: Login
O sistema SHALL autenticar usuarios via Supabase Auth com PKCE, retornando JWT de acesso (1h) + refresh token (httpOnly cookie).

#### Scenario: Login com email e senha
- **WHEN** usuario submete credenciais validas
- **THEN** JWT retornado, redirect para /dashboard

#### Scenario: Login com credenciais invalidas
- **WHEN** usuario submete senha incorreta
- **THEN** erro 401, sem revelar se email existe

### Requirement: Protecao de rotas no frontend
O middleware Next.js SHALL redirecionar usuarios nao autenticados de `/(dashboard)/*` para `/login`, e usuarios logados de `/(auth)/*` para `/dashboard`.

#### Scenario: Acesso a rota protegida sem sessao
- **WHEN** usuario nao autenticado acessa /dashboard
- **THEN** redirect para /login

### Requirement: Autorizacao na API
A API FastAPI SHALL validar JWT Supabase em todas as rotas protegidas via dependencia `get_current_user`, injetando `user_id` no request.

#### Scenario: API chamada sem token
- **WHEN** request sem header Authorization chega a /essays
- **THEN** 401 Unauthorized

### Requirement: Row Level Security
Todas as tabelas com `user_id` SHALL ter RLS habilitado no Postgres, garantindo que usuarios so acessem seus proprios dados mesmo se o backend falhar na autorizacao.

#### Scenario: SELECT em essays sem user_id proprio
- **WHEN** query SQL tenta acessar essays de outro usuario
- **THEN** RLS bloqueia, retorna 0 rows

### Requirement: Admin role (v2.0)
O sistema SHALL suportar coluna `users.role` com valores `user` (default) e `admin`. Rotas `/api/v1/admin/*` exigem `role='admin'` + `status='active'`. Admin acessa em dominio separado `admin.redana.com.br`.

#### Scenario: Admin acessa painel
- **WHEN** usuario com role=admin faz login em admin.redana.com.br
- **THEN** middleware Next.js verifica role, libera acesso ao dashboard admin

#### Scenario: Usuario comum tenta acessar admin
- **WHEN** usuario com role=user acessa admin.redana.com.br
- **THEN** redirect para app.redana.com.br/dashboard

### Requirement: 2FA para admin
O sistema SHALL exigir autenticacao de dois fatores (TOTP) para usuarios com role=admin.

#### Scenario: Login admin
- **WHEN** admin submete credenciais validas
- **THEN** solicitado codigo TOTP; so avanca apos verificacao
