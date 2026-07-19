## Purpose
Adiciona os requisitos desta fase ao sistema.

## ADDED Requirements

### Requirement: Monorepo Turborepo + pnpm
O sistema SHALL ser organizado como monorepo com Turborepo + pnpm.
- Status: ADDS System spec / Arquitetura requirement

#### Scenario: Scaffolding concluido
- **WHEN** `pnpm run build` executado na raiz
- **THEN** Turborepo orquestra build paralelo de todos os apps e packages

### Requirement: Estrutura de subdominios
O sistema SHALL usar subdominios para cada app.
- Status: ADDS System spec / Subdominios requirement

#### Scenario: Roteamento por subdominio
- **WHEN** request chega em api.redana.com.br
- **THEN** CORS permite Origins dos apps autorizados
