# Engine de correção

> Seção original: PLAN.md §10. Design detalhado da engine de correção por LLM.

---

## 10.1 Fluxo

```
1. POST /essays com content
2. Rate limiter verifica subscription.corrections_used < max_corrections
3. Incrementa corrections_used + cria essay status=pending
4. Enfileira job (FastAPI BackgroundTasks; migrar a Celery+Redis depois)
5. CorrectionEngine.process(essay_id):
   a. Monta prompt com texto + rubrica ENEM
   b. Chama Claude (fallback GPT-4o) com tool/function para JSON estruturado
   c. Parse + valida scores (0-200 por competência, soma = overall)
   d. Se inválido → retry 1x com "/json-fix" injetado
   e. Persiste correction + essay.status=completed
   f. Envia email "correção pronta" (Resend)
```

### Diagrama

```mermaid
sequenceDiagram
  participant W as Next.js
  participant A as FastAPI
  participant R as Rate Limiter
  participant DB as Postgres
  participant Q as Background Task
  participant LLM as Claude/GPT-4o
  participant E as Resend

  W->>A: POST /essays
  A->>R: check caps
  R-->>A: OK (or 402/429)
  A->>DB: insert essay (status=pending), inc corrections_used
  A-->>W: 201 (essay_id)
  A->>Q: enqueue process(essay_id)
  W-​-​>>W: poll /essays/{id}/status (SSE ou 3s)
  Q->>LLM: prompt + tool use
  LLM-->>Q: JSON {overall_score, c1..c5, feedback, next_steps}
  Q->>Q: validate (soma = overall, range 0-200)
  alt inválido
    Q->>LLM: retry com "/json-fix" (1x)
  end
  Q->>DB: insert correction, essay.status=completed
  Q->>E: send correction-ready email
  Q-->>DB: insert events (correction.completed)
```

---

## 10.2 Schema de saída do LLM (JSON)

```json
{
  "overall_score": 880,
  "competencies": {
    "c1": { "score": 160, "feedback": "..." },
    "c2": { "score": 200, "feedback": "..." },
    "c3": { "score": 160, "feedback": "..." },
    "c4": { "score": 200, "feedback": "..." },
    "c5": { "score": 160, "feedback": "..." }
  },
  "overall_feedback": "Resumo exec...",
  "next_steps": [
    { "area": "c5", "action": "Detalhar agente, ação, meio e finalidade", "priority": "alta" }
  ]
}
```

### Validação server-side

| Campo | Regra |
|---|---|
| `overall_score` | int 0-1000 |
| `c1..c5.score` | int 0-200 cada |
| `overall_score` | obrigatoriamente `c1+c2+c3+c4+c5` (trigger DB também valida) |
| `c1..c5.feedback` | string não-vazia, máx 2000 chars, PT-BR |
| `overall_feedback` | string não-vazia |
| `next_steps[]` | 1-5 items; cada `{area, action, priority}`; priority ∈ {alta, media, baixa}; area ∈ {c1,c2,c3,c4,c5,geral} |

### Persistência no DB

A engine平面 o JSON para as colunas canônicas do schema `corrections`:

```python
correction = Correction(
    essay_id=essay_id, user_id=user_id,
    overall_score=data["overall_score"],
    c1_score=data["competencies"]["c1"]["score"],
    c1_feedback=data["competencies"]["c1"]["feedback"],
    # ... c2..c5
    overall_feedback=data["overall_feedback"],
    next_steps=data["next_steps"],
    corrected_by="ai",
    llm_model=model_name,
    llm_cost_brl=cost,
)
```

---

## 10.3 Estratégia de prompt

- **System**: "Você é um corretor oficial do ENEM. Use a rubrica atual do INEP."
- **User**: rubrica resumida C1-C5 + texto da redação
- **Mode**: `tool` para forçar JSON válido (Claude) ou `response_format: json_schema` (GPT-4o)
- **Idioma**: 100% PT-BR nas saídas
- **Antialucinação**: exigir 3 citações do texto do aluno por feedback de competência

### Template mínimo (resumo)

```python
SYSTEM = """Você é um corretor oficial do ENEM. Para cada competência C1 a C5:
- Atribua score 0-200 conforme a rubrica atual do INEP.
- Escreva feedback objetivo em PT-BR citando 3 trechos do texto do aluno.
- Sugira 1-5 acciones de melhoria (next_steps) com area, action, priority.
Sempre retorne JSON válido conforme schema."""

USER = f"""
Rubrica oficial resumida:
{RUBRIC_TEXT}

Redação do aluno:
\"\"\"
{essay.content}
\"\"\"

Devolver em JSON estruturado.
"""
```

> A rubrica oficial (`RUBRIC_TEXT`) é cached como variável fixa em `prompt_templates.py` e pode ser substituída por template DB em [admin/system-config.md](../../admin/system-config.md).

---

## 10.4 Controle de custo LLM

- Cache de prompts da rubrica (fixo) — só texto da redação é dinâmico
- Limite de 30 correções/mês nos planos pagos (não "unlimited")
- Estimativa de custo: **~R$0,03 por correção** (Claude Sonnet entrada+saída ~2k tokens)
- Dashboard de custo no admin (ver [admin/analytics.md](../../admin/analytics.md))
- `corrections.llm_cost_brl` por item → agregações SQL

### Cálculo do `llm_cost_brl`

```python
def calc_cost(model: str, in_tokens: int, out_tokens: int) -> Decimal:
    rates = {
        "claude-3-5-sonnet": {"in": 3.00e-6, "out": 15.00e-6},   # USD por token
        "claude-3-5-haiku":  {"in": 0.25e-6, "out": 1.25e-6},
        "gpt-4o":            {"in": 2.50e-6, "out": 10.00e-6},
    }
    rate = rates[model]
    usd = in_tokens * rate["in"] + out_tokens * rate["out"]
    return usd * BRL_RATE   # BRL_RATE atualizado por env/config
```

### Margem (cenário Anual)

- Receita mensalizada: R$ 82,80 / 12 = R$ 6,90/mês
- Limite: 30 correções/mês
- Custo médio: 30 × R$0,03 = **R$ 0,90/mês**
- **Margem: 87%** — compatível com a meta do produto (≥ 80%)

---

## 10.5 UX de latência

- Frontend mostra "Analisando..." com skeleton de `CompetencyBars`
- Polling `/essays/{id}/status` a cada 3s OU SSE via `/essays/{id}/stream`
- Tempo médio: **20-40s** → mostrar progresso com tempo estimado

### Estados de progresso

| Estado | UI |
|---|---|
| pending | "Enfileirando..." (rápido) |
| processing | "Lendo sua redação..." → "Avaliando C1, C2..." → "Gerando feedback" |
| completed | Redirect automático para visualizar |
| failed | Toast "Não foi possível corrigir. Tentar novamente?" |

---

## Resilência

| Falha | Tratamento |
|---|---|
| Claude API timeout | Fallback para GPT-4o |
| JSON inválido após 1 retry | Marca `essay.status='failed'`, não consome `corrections_used` (reverte increment), email "tente novamente" |
| Rate limit Anthropic | Backoff exponencial 1s, 2s, 4s; se esgotar, fallback |
| LLM cost > R$0,10 por correção | Alerta no admin + terroria (não bloqueia) |

→ Logs de erro em Sentry; agregação em `events`.

---

## Modo de correção humana (override)

Quando admin sobrescreve uma correção manualmente:
- `corrected_by = 'human'`, `llm_model = null`, `llm_cost_brl = 0`
- Não invoca LLM
- Ver [admin/corrections-admin.md](../../admin/corrections-admin.md)