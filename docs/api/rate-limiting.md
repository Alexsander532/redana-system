# Rate limiting

> Documento NOVO (v2.0). Extraído e expandido da seção de segurança do PLAN.md §13.1.

---

## Regras

| Escopo | Limite | Implementação |
|---|---|---|
| Por IP, rotas autenticadas | 60 req/min | slowapi middleware (token bucket) |
| Por usuário, `POST /essays` | 5 essays/hora | Check em `rate_limiter.py` que consulta tabela (ou Redis pós-MVP) |
| Por usuário, submissão total | bloqueado por `subscriptions.corrections_used < max_corrections` | Lógica de negócio (não é rate limit puro) |
| Webhook AbacatePay | 30 req/min por IP | slowapi separado (sem auth, com HMAC verify) |
| Routes admin (`/admin/*`) | 120 req/min por admin | slowapi bucket |

---

## Implementação

```python
# apps/api/app/main.py
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# routers/essays.py
@router.post("/essays")
@limiter.limit("60/minute")
async def create_essay(request: Request, body: EssayCreate, user: User = Depends(get_current_user)):
    await check_essay_hourly_cap(user.id)        # 5/hora
    await check_subscription_cap(user.id)        # corrections_used < max_corrections
    ...
```

### Caps de submissão (não são rate limit HTTP)

```python
# apps/api/app/services/rate_limiter.py
async def check_essay_hourly_cap(db: AsyncSession, user_id: UUID) -> None:
    one_hour_ago = datetime.utcnow() - timedelta(hours=1)
    count = await db.scalar(
        select(func.count()).where(
            essays.c.user_id == user_id,
            essays.c.created_at >= one_hour_ago,
        )
    )
    if count >= 5:
        raise HTTPException(429, "Too many essay submissions in the last hour (max 5)")

async def check_subscription_cap(db: AsyncSession, user_id: UUID) -> None:
    sub = await get_subscription(db, user_id)
    if sub.max_corrections is not None and sub.corrections_used >= sub.max_corrections:
        raise HTTPException(402, "Subscription limit reached. Upgrade to continue.")  # 402 Payment Required
```

---

## Respostas 429

```json
{
  "error": {
    "code": "rate_limited",
    "message": "Too many requests. Try again in 42 seconds.",
    "retry_after": 42
  }
}
```

Headers (slowapi):
- `X-RateLimit-Limit`
- `X-RateLimit-Remaining`
- `X-RateLimit-Reset`

---

## Política de torpedeio (abuso)

| Cenário | Limite | Ação |
|---|---|---|
| Tentativas de login falhadas | 10/login/5min por IP | Após 10, IP bloqueado por 15 min |
| Cadastros por IP | 5/dia | Após 5, novos emails de anderenhoo exigem CAPTCHA (pós-MVP) |
| Submissões de webhook inválidas (HMAC fail) | 5/min | IP banido no firewall por 1h |

---

## Migração futura

→ Redis-backed rate limiter quando houver múltiplos workers Railway ou migração para Celery. Os limites definidos ficam idênticos — só muda o storage backend.