# Landing Page — Pagina de marketing (Astro)

## Requirements

### Requirement: Pagina estatica em Astro
A landing page SHALL ser servida como site Astro estatico em `redana.com.br`, com conteudo: hero, proof strip, beneficios, como funciona, depoimentos, precos, FAQ, CTA final.

#### Scenario: Visitante acessa landing
- **WHEN** usuario digita redana.com.br
- **THEN** pagina carregada com todas as secoes, responsiva 320px-1440px

### Requirement: SEO e GEO
A landing SHALL incluir meta tags (title ~59 chars, description ~153 chars), hreflang pt-BR, JSON-LD (Organization, WebSite, SoftwareApplication, FAQPage, AggregateOffer), sitemap.xml, llms.txt, robots.txt com regras para GPTBot/ClaudeBot/Google-Extended.

#### Scenario: Bot de busca indexa pagina
- **WHEN** Googlebot ou ClaudeBot acessa a pagina
- **THEN** meta tags, JSON-LD e conteudo semanticamente estruturado disponiveis

### Requirement: Carrossel de depoimentos
O carrossel SHALL exibir 8 depoimentos com auto-scroll (2s) + navegacao por drag + dots + setas.

#### Scenario: Visitante ve depoimentos
- **WHEN** secao de depoimentos visivel no viewport
- **THEN** carrossel inicia auto-scroll, usuario pode pausar ao interagir

### Requirement: Secao de precos com ancoragem
A secao de precos SHALL exibir 4 planos (Free, Mensal, Trimestral, Anual) com preco mensal equivalente, destaque visual no plano Anual (ancoragem), e mental accounting (ex: "R$0,23/dia").

#### Scenario: Visitante compara planos
- **WHEN** secao de precos visivel
- **THEN** 4 cards lado a lado, Anual destacado como "Mais popular", precos diarios exibidos

### Requirement: CTA para cadastro
Botoes CTA SHALL direcionar para `app.redana.com.br/signup`.

#### Scenario: Visitante clica no CTA do hero
- **WHEN** usuario clica "Comecar gratis"
- **THEN** navegador redireciona para app.redana.com.br/signup
