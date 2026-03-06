# Operations: Webhooks Over Polling

Agents need subscription state changes delivered to them — not discovered through polling. RC webhooks are the mechanism.

---

## Setup (via MCP or REST)

```python
# Via MCP
call("mcp_RC_create_webhook_integration", {
    "project_id": PROJECT_ID,
    "url": "https://your-app.example.com/webhook/revenuecat",
    "name": "Production Webhook",
    "event_types": [
        "INITIAL_PURCHASE", "RENEWAL", "CANCELLATION",
        "EXPIRATION", "BILLING_ISSUE", "PRODUCT_CHANGE",
        "TRIAL_STARTED", "TRIAL_CONVERTED", "TRIAL_CANCELLED"
    ]
})
```

---

## The Complete Handler

```python
from fastapi import FastAPI, Request, HTTPException
import hmac, hashlib

app = FastAPI()

# RC signs webhook payloads — verify before processing
def verify_rc_signature(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), payload, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)

@app.post("/webhook/revenuecat")
async def handle_rc_webhook(request: Request):
    body = await request.body()
    sig = request.headers.get("X-RevenueCat-Signature", "")
    
    if not verify_rc_signature(body, sig, RC_WEBHOOK_SECRET):
        raise HTTPException(status_code=401, detail="Invalid signature")
    
    payload = await request.json()
    event = payload.get("event", {})
    event_type = event.get("type")
    user_id = event.get("app_user_id")
    
    await log_subscription_event(event_type, user_id, event)
    await dispatch_event(event_type, user_id, event)
    
    return {"status": "ok"}


async def dispatch_event(event_type: str, user_id: str, event: dict):
    match event_type:
        case "INITIAL_PURCHASE":
            await on_initial_purchase(user_id, event)
        
        case "RENEWAL":
            await on_renewal(user_id, event)
        
        case "CANCELLATION":
            # User canceled — subscription still active until period end
            # Log intent, schedule win-back, don't remove access
            await on_cancellation(user_id, event)
        
        case "EXPIRATION":
            # Subscription period ended — remove access
            await on_expiration(user_id, event)
        
        case "BILLING_ISSUE":
            # Payment failed, grace period started
            # User thinks they're subscribed — gentle notification only
            await on_billing_issue(user_id, event)
        
        case "TRIAL_STARTED":
            await on_trial_started(user_id, event)
        
        case "TRIAL_CONVERTED":
            # Trial → paid — cancel any win-back sequences
            await on_trial_converted(user_id, event)
        
        case "TRIAL_CANCELLED":
            # Trial canceled — immediate win-back attempt
            await on_trial_cancelled(user_id, event)
        
        case "PRODUCT_CHANGE":
            # Upgrade or downgrade
            await on_product_change(user_id, event)
```

---

## The Events That Matter Most

### `BILLING_ISSUE` — Most Mishandled

```python
async def on_billing_issue(user_id: str, event: dict):
    # ⚠️ The user THINKS they're still subscribed. Don't gate them.
    # Grace period: typically 6-16 days (varies by store).
    # RC retries payment automatically during this period.
    
    # DO: soft notification
    await send_notification(user_id, 
        subject="Quick note about your subscription",
        body="We had trouble processing your payment. We're retrying automatically. "
             "If you'd like to update your payment method: [link]"
    )
    
    # DON'T: revoke access, show a hard paywall, send urgent/scary language
```

### `CANCELLATION` vs `EXPIRATION` — Often Confused

```python
async def on_cancellation(user_id: str, event: dict):
    # Fires when: user taps "Cancel subscription" in App Store
    # User still has access until: event["expiration_at_ms"]
    # This is intent to churn, not churn itself
    
    expiry = event.get("expiration_at_ms", 0) / 1000
    days_remaining = (expiry - time.time()) / 86400
    
    # Schedule win-back for after expiry, not immediately
    await schedule_winback(user_id, delay_days=max(1, days_remaining))


async def on_expiration(user_id: str, event: dict):
    # Fires when: subscription period actually ends with no renewal
    # User has NO access now
    
    await revoke_premium_access(user_id)
    
    # Immediate win-back: user just lost access, motivation is highest NOW
    await send_winback_email(user_id, template="just_expired")
    
    # Cancel any scheduled win-backs (this is the real one)
    await cancel_scheduled_winbacks(user_id)
```

### `RENEWAL` — Grant Credits

```python
async def on_renewal(user_id: str, event: dict):
    # Subscription renewed — grant this cycle's credits if using hybrid model
    product_id = event.get("product_id")
    credits_to_grant = PRODUCT_CREDIT_MAP.get(product_id, 100)
    
    await grant_credits(user_id, amount=credits_to_grant)
    
    # Cancel any win-back sequences (they came back / stayed)
    await cancel_scheduled_winbacks(user_id)
```

---

## Idempotency

RC may deliver the same webhook more than once. Your handler must be idempotent:

```python
async def on_initial_purchase(user_id: str, event: dict):
    transaction_id = event.get("transaction_id")
    
    # Check if already processed
    existing = await db.fetch_one(
        "SELECT id FROM processed_events WHERE transaction_id = ?",
        [transaction_id]
    )
    if existing:
        return  # Already handled — idempotent return
    
    await grant_premium_access(user_id)
    await send_welcome_email(user_id)
    
    await db.execute(
        "INSERT INTO processed_events (transaction_id, processed_at) VALUES (?, ?)",
        [transaction_id, datetime.utcnow()]
    )
```

---

## Testing Webhooks Locally

```bash
# RC can send test events from the dashboard
# Or use their webhook simulator:
# Dashboard → Project → Integrations → Webhooks → Send Test Event

# For local dev, ngrok or similar:
ngrok http 8000
# Set webhook URL to https://xxx.ngrok.io/webhook/revenuecat in RC dashboard
```
