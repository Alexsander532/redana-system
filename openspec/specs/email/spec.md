# Email — Emails transacionais e broadcast

## Requirements

### Requirement: Emails transacionais via Resend
O sistema SHALL enviar emails transacionais via Resend com templates React Email para: welcome (pos-cadastro), correcao-pronta (pos-correcao), reset-password, upgrade-reminder.

#### Scenario: Usuario se cadastra
- **WHEN** cadastro concluido com sucesso
- **THEN** email de welcome enviado com CTA para primeira redacao

#### Scenario: Correcao fica pronta
- **WHEN** status de redacao muda para completed
- **THEN** email "Sua redacao foi corrigida" enviado com link para resultado

### Requirement: Broadcast de emails (v2.0 admin)
O sistema SHALL permitir que admin envie emails em massa por segmento, com preview obrigatorio e persistencia de historico em tabela `broadcast_emails`.

#### Scenario: Admin envia broadcast
- **WHEN** admin compoe, preview, seleciona segmento e confirma
- **THEN** emails enfileirados, `broadcast_emails` registrado com recipient_count
