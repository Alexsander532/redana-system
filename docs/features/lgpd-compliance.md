# Segurança & LGPD

> Seção original: PLAN.md §13 (§13.1 Segurança + §13.2 LGPD combinados).

---

## 13.1 Segurança

### Secrets

- **Secrets em env vars**, nunca em código. `.env.example` documenta todas as chaves sem valor real.
- `.env` está no `.gitignore`.
- No Railway/Vercel, secrets via variáveis de ambiente criptografadas.
- Rotação periódica: `ABACATE_WEBHOOK_SECRET`, `RESEND_API_KEY`.

### CORS

API permite só:

- `https://app.redana.com.br` (dashboard)
- `https://redana.com.br` (landing)

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.redana.com.br", "https://redana.com.br"],
    allow_credentials=False,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

### Rate limiting

Ver [rate-limiting.md](../api/rate-limiting.md):

- 60 req/min por IP
- 5 essays/hora por user
- 30 webhook/minute por IP
- 60 admin/minute por admin

### Input validation

- **Pydantic v2** em todos endpoints (`schemas/*`)
- Redação entre 150-1500 palavras (ver [essay-submission.md](./essay-submission.md))
- UUID validation em path params

### Upload

- Apenas `.txt` e `.docx`, máx 100KB
- Antivírus escaneamento (pós-MVP)
- Magic bytes check (não confiar só na extensão)

```python
def validate_file(file: UploadFile):
    if file.content_type not in ("text/plain", "application/vnd.openxmlformats-officedocument.wordprocessingml.document"):
        raise HTTPException(415, "Formato não suportado")
    # Magic bytes (opcional)
    head = await file.read(8); await file.seek(0)
    if file.filename.endswith(".docx") and not head.startswith(b"PK\x03\x04"):
        raise HTTPException(415, "Arquivo .docx inválido")
```

### Webhook HMAC

- Signature verification antes de processar evento AbacatePay (`/webhooks/abacatepay`) e Resend (`/webhooks/resend`, opcional pós-MVP)
- Idempotência por `abacate_event_id` UNIQUE → replay seguro
- Ver [webhooks-abacatepay.md](../api/webhooks-abacatepay.md)

### RLS — Row Level Security

- Todas tabelas com `user_id` têm RLS habilitado
- Admin policies adicionadas: `exists (select 1 from users where id = auth.uid() and role = 'admin')`
- Tabelas só-admin (`admin_audit_log`, `webhook_events`, `plans_config`, `prompt_templates`, `feature_flags`, `email_logs`) têm RLS admin-only
- Ver [schema.md](../data/schema.md)

### TLS

- TLS 1.2+ obrigatório (Railway/Vercel forçam)
- HSTS header on (`Strict-Transport-Security: max-age=31536000`)
- Cookies httpOnly para refresh token

### Outras proteções

| Camada | Implementação |
|---|---|
| CSRF | Cookie `SameSite=Lax` no refresh + Origin check em mutations server actions |
| XSS | React escape por padrão; Trust Level baixo em recovered HTML |
| SQL Injection | SQLAlchemy parametrizado (zero strings SQL) |
| SSRF | URLs do webhook só vêm do AbacatePay (whitelist de IPs se possível) |
| Mass assignment | Pydantic `model_config = ConfigDict(extra='forbid')` |

---

## 13.2 LGPD

### Base legal

- **Consentimento** do titular para tratar a redação (finalidade: correção educacional)
- Checkbox explícito no signup: "Concordo com os Termos e a Política de Privacidade."
- Log do consentimento (timestamp + IP) — armazenado em `users.created_at` + `events` (`event_type='signup_consent'`)

### Documentos

- **Política de Privacidade** + **Termos de Uso** acessíveis no rodapé (landing e dashboard)
- Email DPO: `privacidade@redana.com.br` (configurado no Resend + rodapé + Política)

### Retenção

- Conteúdo da redação (`essays.content`) mantido por **12 meses**
- Após 12 meses, campo `content` é sobrescrito por **hash anonimizado** (mantém notas/feedback para analytics)
- Job mensal de retenção executa: `UPDATE essays SET content = sha256(content) WHERE created_at < now() - interval '12 months' AND content ~ '^\w+$'` (heurística para não reanonimizar)
- Redações anônimas não podem mais ser exibidas para o user (content é hash incompreensível); o resultado da correção mantém-se visível sem o texto original

```sql
-- package/db/anonymize_old_essays.sql
update essays
  set content = 'anonymized:' || encode(digest(content, 'sha256'), 'hex')
where created_at < now() - interval '12 months'
  and content not like 'anonymized:%';
```

### Direitos do titular

| Direito | Endpoint | Fluxo |
|---|---|---|
| Acesso / portabilidade | `POST /account/export` | Gera JSON com `essays`, `corrections`, `subscription`, audit log do user → email com link presigned Supabase Storage (24h) |
| Eliminação | `POST /account/delete` | Grace period 30 dias → hard delete (cascade). User pode cancelar dentro de 30d via Conta |
| Retificação | Editar nome na tela `/conta` | Server Action `updateName` → API `PATCH /auth/me` |
| Revogação consentimento | `POST /account/delete` | Mesmo fluxo da eliminação |

### Exportação de dados

`POST /account/export`:

1. Gera JSON completo inline (não tem volume no MVP)
2. Upload para Supabase Storage privado com URL presigned de 24h
3. Enviar email "Seus dados estão prontos" com o link
4. Link logado em `events`

### Exclusão de conta

- Grace period: 30 dias
- Durante grace period, login bloqueado mas dados preservados
- Dia 30: job roda `DELETE FROM users WHERE id = ...` (cascade remove essays, corrections, subscription)
- User pode cancelar a exclusão nos 30 dias via link no email "Exclusão agendada"

### Criptografia

| Estado | Tecnologia |
|---|---|
| Em repouso | Supabase gerenciado AES-256 |
| Em trânsito | TLS 1.2+ |
| Backups | Supabase automated (encrypted) |

### Encarregado / DPO

- Email `privacidade@redana.com.br` na política
- Canal de exercício de direitos: endpoint `/account/export` + `/account/delete` + FAQ contato
- Resposta em até 15 dias (padrão LGPD)

### Logs / Analytics

- Tabela `events` não armazena conteúdo da redação (somento metadados)
- Sentry: PII stripped (configura `before_send` filter para remover `essay.content`, `user.email`)

```python
def sentry_before_send(event, hint):
    if "exception" in event:
        for frame in event["exception"]["values"][0]["stacktrace"]["frames"]:
            frame["vars"] = {k: v for k, v in frame.get("vars", {}).items()
                              if k not in ("content", "essay_text", "password", "token")}
    return event
```

---

## Compliance checklist

- [ ] Política de Privacidade publicada
- [ ] Termos de Uso publicados
- [ ] Checkbox de consentimento no signup
- [ ] Email de DPO visível no rodapé
- [ ] Job de retenção 12 meses mensal
- [ ] Endpoints `/account/export` e `/account/delete` funcionais
- [ ] Sentry PII filter ativo
- [ ] RLS em todas as tabelas com user_id
- [ ] Audit log de admin actions (v2.0 — ver [`admin/`](../../admin/overview.md))