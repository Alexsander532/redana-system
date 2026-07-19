# Admin Panel — Painel administrativo (v2.0)

## Requirements

### Requirement: App separado em dominio proprio
O Admin Panel SHALL rodar em app Next.js 14 separado (`apps/admin`) no dominio `admin.redana.com.br`, deployado na Vercel.

#### Scenario: Admin acessa painel
- **WHEN** admin digita admin.redana.com.br
- **THEN** login admin carregado (mesmo Supabase Auth, gate por role=admin + 2FA TOTP)

### Requirement: Gestao de usuarios
O admin SHALL poder listar, buscar, suspender, banir e visualizar dados de qualquer usuario (redacoes, assinatura, historico).

#### Scenario: Admin suspende usuario
- **WHEN** admin clica "Suspender" no perfil de um usuario
- **THEN** usuario bloqueado de login, audit log registrado

#### Scenario: Admin visualiza redacoes do usuario
- **WHEN** admin acessa detalhe do usuario e clica "Redacoes"
- **THEN** lista de redacoes do usuario exibida, com scores e status

### Requirement: Correcao humana
O admin SHALL poder corrigir redacao manualmente: preencher scores C1-C5 (0-200 cada), feedbacks por competencia, nota geral (validada como soma), sobrescrevendo correcao IA com `corrected_by='human'`.

#### Scenario: Admin corrige redacao manualmente
- **WHEN** admin preenche HumanCorrectionForm com scores e feedbacks e submete
- **THEN** PUT /admin/corrections/{essay_id} → correction atualizada, essay.status=completed, email "revisada" enviado, audit log registrado

### Requirement: Gestao de assinaturas
O admin SHALL poder forcar status de assinatura, conceder correcoes extras, conceder/estender planos e disparar reembolso integral.

#### Scenario: Admin concede plano anual manualmente
- **WHEN** admin seleciona usuario, escolhe "Conceder Anual", define dias
- **THEN** subscription atualizada (plan=annual, period_end ajustado), audit log registrado

### Requirement: Analytics dashboard
O admin SHALL visualizar metricas: MRR, assinantes ativos, conversao Free→Paid, correcoes/dia, custo LLM/dia, latencia media, taxa de falha, com graficos (Recharts).

#### Scenario: Admin acessa /metricas
- **WHEN** admin abre painel de metricas
- **THEN** cards KPI + graficos carregados com dados agregados do DB

### Requirement: Webhook event log
O admin SHALL poder visualizar todos os eventos de webhook AbacatePay recebidos, com status, timestamp e possibilidade de replay.

#### Scenario: Admin investiga webhook com falha
- **WHEN** admin filtra webhook events por status=failed
- **THEN** lista de eventos falhos exibida, botao "Replay" disponivel

### Requirement: Configuracao do sistema
O admin SHALL poder gerenciar prompt templates (criar/editar/ativar versoes), feature flags (toggle + override per-user), planos de preco e limites.

#### Scenario: Admin edita prompt de correcao
- **WHEN** admin abre prompt template ativo, edita e clica "Ativar nova versao"
- **THEN** nova versao ativada, correcoes subsequentes usam novo prompt, audit log registrado

### Requirement: Broadcast de emails
O admin SHALL poder enviar emails em massa por segmento (todos, free, pagos, admins) com preview obrigatorio, limite de 10k destinatarios/broadcast, 1 broadcast/segmento/dia.

#### Scenario: Admin envia broadcast para usuarios free
- **WHEN** admin compoe email, seleciona segmento "free", preview obrigatorio, confirma envio
- **THEN** emails enfileirados via Resend, broadcast_log registrado com recipient_count

### Requirement: Impersonation
O admin SHALL poder "logar como" um usuario (impersonation) com token scoped de 30min, banner visivel durante impersonation, sem acoes destrutivas permitidas, audit log obrigatorio.

#### Scenario: Admin impersona usuario
- **WHEN** admin clica "Logar como usuario" no perfil de um usuario
- **THEN** sessao de impersonation iniciada, banner "Visualizando como [nome]" exibido, sem botoes de delete/suspend visiveis

### Requirement: Audit log
O sistema SHALL registrar toda mutacao de admin em tabela `admin_audit_log` imutavel (sem UPDATE/DELETE), com admin_id, action, target_type, target_id, payload (diff antes/depois), ip_address, user_agent.

#### Scenario: Admin bane usuario
- **WHEN** admin executa ban
- **THEN** linha inserida em admin_audit_log com action=user.ban, payload com before/after

### Requirement: Seguranca do admin
O sistema SHALL exigir 2FA (TOTP) para admin, aplicar IP allowlist opcional via middleware, e restringir CORS em /api/v1/admin/* para Origin admin.redana.com.br.

#### Scenario: IP nao autorizado tenta acessar admin
- **WHEN** request para /api/v1/admin/* vem de IP fora da allowlist
- **THEN** 403 Forbidden antes de qualquer processamento
