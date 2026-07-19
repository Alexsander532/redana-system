# Essays — Submissao e correcao de redacoes

## Requirements

### Requirement: Envio de redacao
O sistema SHALL permitir que usuarios submetam redacoes via texto colado (150-1500 palavras) ou upload de arquivo `.txt`/`.docx` (max 100KB).

#### Scenario: Envio por texto
- **WHEN** usuario cola texto no EssayForm e clica "Corrigir agora"
- **THEN** redacao validada (min 150 palavras), status=pending, correcao async enfileirada

#### Scenario: Envio por upload
- **WHEN** usuario faz upload de arquivo .txt/.docx valido
- **THEN** conteudo extraido pelo text_parser, validado, enfileirado para correcao

#### Scenario: Texto muito curto
- **WHEN** usuario submete texto com menos de 150 palavras
- **THEN** erro de validacao, mensagem explicativa

### Requirement: Correcao por IA
O sistema SHALL corrigir redacoes usando Anthropic Claude 3.5 Sonnet (primario) com fallback para GPT-4o, retornando notas de 0-200 por competencia (C1-C5), nota geral (soma das 5, 0-1000), feedback por competencia e proximos passos em JSON estruturado.

#### Scenario: Correcao bem-sucedida
- **WHEN** correction_engine processa redacao
- **THEN** JSON valido retornado com scores C1-C5, feedback textual, next_steps; essay.status=completed; email enviado ao usuario

#### Scenario: LLM retorna JSON invalido
- **WHEN** resposta do Claude nao valida contra schema esperado
- **THEN** retry 1x com prompt de correcao; se falhar novamente, essay.status=failed

#### Scenario: Claude indisponivel
- **WHEN** chamada ao Claude falha (timeout/erro)
- **THEN** fallback para GPT-4o automaticamente

### Requirement: Controle de custo LLM
O plano Free SHALL ter 5 correcoes vitalicias. Planos pagos SHALL ter 30 correcoes por ciclo de cobranca. Custo LLM registrado por correcao na coluna `llm_cost_brl`.

#### Scenario: Free tier esgotado
- **WHEN** usuario free com 5 correcoes usadas tenta enviar nova redacao
- **THEN** bloqueado, UpgradeBanner exibido com CTAs para planos pagos

#### Scenario: Plano pago atinge limite mensal
- **WHEN** usuario pago atinge 30 correcoes no ciclo
- **THEN** bloqueado ate renovacao do ciclo

### Requirement: Prompt de correcao ENEM
O sistema SHALL usar rubrica oficial do INEP como system prompt, exigindo 3 citacoes do texto do aluno por feedback de competencia, saida em PT-BR, formato via tool use (Claude) ou json_schema (GPT-4o).

#### Scenario: Correcao com citacoes
- **WHEN** feedback de C1 e gerado
- **THEN** contem pelo menos 3 trechos citados do texto original do aluno

### Requirement: Correcao humana (v2.0 admin)
O sistema SHALL permitir que admin sobrescreva correcao de IA com notas e feedbacks manuais, marcando `corrected_by='human'` e `llm_model=NULL`.

#### Scenario: Admin corrige manualmente
- **WHEN** admin preenche scores C1-C5 e feedbacks via HumanCorrectionForm
- **THEN** correction atualizada, corrected_by=human, email "redacao revisada" enviado, audit log registrado

### Requirement: Dashboard de evolucao
O sistema SHALL exibir grafico de evolucao (Recharts) com scores por competencia ao longo das redacoes do usuario.

#### Scenario: Usuario com 3+ redacoes acessa dashboard
- **WHEN** dashboard home carrega
- **THEN** grafico de linhas mostra progresso C1-C5 nas ultimas redacoes
