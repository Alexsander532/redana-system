# Admin — Modo Correção Humana

> Documento detalhado do modo de correção manual pelo admin (override da IA).

---

## O que é

Quando o admin discord da correção da IA (ou a IA falha), o admin pode abrir a redação e **manualmente**:
- Marcar notas C1 a C5 (0-200 cada)
- Escrever feedback por competência
- Escrever overall_feedback
- Definir next_steps

A correction humana **substitui** a IA (não há histórico no MVP — ver decisão abaixo). `corrected_by` passa a `'human'`, `llm_model = null`, `llm_cost_brl = 0`.

> Decisão pendente: manter versões anteriores da correction? Possível tabela `correction_versions` no futuro. Para v2.0, sobrescreve em-place com diff salvo no `admin_audit_log`.

---

## Quando usar

| Cenário | Ação |
|---|---|
| LLM retorna scores absurdo (ex: overall != soma) | Reprocessar IA primeiro; se persistir, override humano |
| Aluno contesta a correção | Admin revisa e corrige manualmente |
| Plano "correção premium" futuro | TBD — futura feature onde aluno paga extra e recebe correção humana prioritária |
| LLM fica indisponível por > 30 min | Admin pode corrigir manualmente as essays em fila para não bloquear service |

---

## Fila de correções (`/admin/correcoes`)

Tabela de essays que estão prontas para correção humana:
- Filtrável por status (`pending`/`processing`/`failed`/`completed`)
- Por default mostra essays `failed` + essays marcadas como "needs human review" (ver abaixo)
- Por assignee (unassigned / atribuída ao admin atual)

### Marcação "needs human review"

Coluna opcional em essays (v2.0):

```sql
alter table essays add column needs_human_review boolean not null default false;
alter table essays add column assigned_admin_id uuid references users(id);
```

Setada:
- Automaticamente quando LLM falha 2x seguidas
- Manualmente pelo admin ao ver uma essay

→ Aparece na fila como vermelha/dourada.

---

## Tela `/admin/correcoes/[essay_id]`

### Layout

Flat: 1 única tela de correção, scrollável.

#### Bloco 1: Cabeçalho
- Essay title, user (link), status, word_count, source, created_at
- Diff da correction atual se existir (highlight: antes | depois)

#### Bloco 2: Texto da redação (left, sticky)
- Read-only display em coluna esquerda (40% width)
- Linhas numeradas
- Highlight on hover

#### Bloco 3: Form de correção (right)
- Overall score preview dinâmico (soma C1-C5)
- 5 seções C1-C5
- Without NextSteps inline edit

#### Bloco 4: Botões
- Salvar como humano (`corrected_by='human'`)
- Cancelar (volta pra lista)
- Comparar com IA atual (modal diff)

---

## `HumanCorrectionForm` (componente)

```tsx
function HumanCorrectionForm({ essayId, initial }: Props) {
  const [c1Score, setC1Score] = useState(initial?.c1_score ?? 0);
  // ... c2..c5
  const overall = c1+c2+c3+c4+c5;

  return (
    <Form action={async (fd) => { await saveHumanCorrection(fd); }}>
      <div className="grid grid-cols-5 gap-4">
        {["c1","c2","c3","c4","c5"].map((c) => (
          <CompetencyScoreInput key={c} name={c} score={score} setScore={...} />
        ))}
      </div>
      <OverallScorePreview value={overall} />
      {["c1","c2","c3","c4","c5"].map((c) => (
        <Textarea required name={`${c}_feedback`} placeholder={`Feedback ${c.toUpperCase()}...`} />
      ))}
      <Textarea required name="overall_feedback" placeholder="Resumo executivo" />
      <NextStepsEditor name="next_steps" />
      <Button type="submit">Salvar correção humana</Button>
    </Form>
  );
}
```

### Validation client-side

- Cada score 0-200
- Overall = soma C1-C5
- Cada feedback 1-2000 chars
- next_steps 0-5 items, cada com `area`, `action`, `priority`

### Validation server-side (Pydantic)

```python
class HumanCorrectionSchema(BaseModel):
    c1_score: int = Field(ge=0, le=200)
    c2_score: int = Field(ge=0, le=200)
    c3_score: int = Field(ge=0, le=200)
    c4_score: int = Field(ge=0, le=200)
    c5_score: int = Field(ge=0, le=200)
    overall_score: int = Field(ge=0, le=1000)
    c1_feedback: str = Field(min_length=1, max_length=2000)
    # ... c2..c5
    overall_feedback: str = Field(min_length=1, max_length=4000)
    next_steps: list[NextStep] = Field(max_length=5)

    @model_validator(mode="after")
    def check_sum(self):
        if self.overall_score != self.c1_score + self.c2_score + self.c3_score + self.c4_score + self.c5_score:
            raise ValueError("overall must equal sum of C1-C5")
        return self
```

---

## Endpoint API

```
PUT /api/v1/admin/corrections/{essay_id}
```

Body: `HumanCorrectionSchema` (acima)

Efeito:
1. Verifica `essay.status in ('completed', 'failed')` (must exists)
2. Busca correction atual → `before` snapshot (para audit diff)
3. UPSERT correction:
   - `corrected_by = 'human'`
   - `llm_model = null`
   - `llm_cost_brl = 0`
   - scores + feedbacks do body
4. Marca `essay.status='completed'`, `completed_at=now` (se ainda não)
5. Marca `essay.needs_human_review = false`
6. Audit `correction.human_override` com payload diff
7. Envia email "Sua redação foi reavaliada" (opcional, com flag `send_email`)
8. Retorna `{ok: true, audit_id, correction_id}`

### Assign

```
POST /api/v1/admin/corrections/{essay_id}/assign
body: { "admin_id": "uuid or null (unassign)" }
```

Efeito: `essays.assigned_admin_id = admin_id`. Se `admin_id == null`, desatribui.
Audit: `correction.assign`.

→ Aparece na "minha fila" do admin (`/admin/correcoes?assigned_to=me`).

---

## Diff em UI

Após salvar, mostra modal/bloco:

```
Diff da correction:
┌─────┬─────────┬──────────┬──────────┐
│     │ IA      │ Humano    │ Δ        │
├─────┼─────────┼──────────┼──────────┤
│ C1  │ 160     │ 140       │ -20      │
│ C2  │ 200     │ 200       │ —        │
│ C3  │ 160     │ 180       │ +20      │
│ C4  │ 200     │ 200       │ —        │
│ C5  │ 160     │ 160       │ —        │
│ Sum │ 880     │ 880       │ 0        │
└─────┴─────────┴──────────┴──────────┘
```

(Stored in `admin_audit_log.payload` JSONB after-save.)

---

## UX states

| Estado | Comportamento |
|---|---|
| Loading correction atual | Skeleton form |
| Sem correction atual | Form starts zeros |
| Invalid overall != sum | Inline error vermelho no preview |
| Saving | Botão desabilita + spinner |
| Saved | Toast "Correção salva — novo overall 880", exibe diff |
| Network error | Toast "Falha ao salvar", mantém form |

---

## Considerações

- A submissão humana **não** chama o LLM, portanto sem custo LLM
- `corrected_by='human'` é exibido no dashboard do aluno como badge "Avaliação revisada pela equipe Redana" (ou similar — UX decision)
- Não há rate limit para correção humana (admin actions são ilimitadas)
- Não há versionamento no MVP. Possível melhoria: `correction_versions` (ver [risks-and-open-questions.md](../process/risks-and-open-questions.md))