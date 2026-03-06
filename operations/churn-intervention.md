# Operations: Churn Intervention

Churn intervention is the highest-ROI work in subscription ops. These users already know your product. The question is whether they remember why they valued it.

---

## The Three Windows

RC's subscription state machine gives you three distinct intervention windows:

```
grace_period    → Payment failed, store retrying. User thinks they're subscribed.
                  Window: 1–16 days (varies by store).
                  Action: SOFT notification. Do NOT show a paywall. Do NOT revoke access.

billing_retry   → Grace period ended, RC is still retrying.
                  Window: variable.
                  Action: Direct payment update ask. Clear, not urgent.

expired         → Gone. No access.
                  Window: first 30 days post-expiry is highest-conversion win-back window.
                  Action: Win-back sequence. Best offer goes out within 24 hours.
```

**The most important distinction:** `CANCELLATION` ≠ `EXPIRATION`.

- `CANCELLATION` fires when the user cancels. They still have access. This is *intent* to churn.
- `EXPIRATION` fires when the period ends. Access is gone. This is actual churn.

Don't treat these the same. Panicking at cancellation annoys users who changed their mind. Missing the expiration window loses them.

---

## The Grace Period: Most Mishandled

```python
async def on_billing_issue(user_id: str, event: dict):
    # The user THINKS they're subscribed.
    # RC is retrying payment automatically.
    # Your job: gentle heads-up, not alarm.
    
    await send_notification(user_id,
        subject="A note about your subscription",
        body=(
            "We had a small issue processing your latest payment — "
            "we're retrying automatically and you won't lose access. "
            "If you'd like to update your payment method in the meantime: {link}"
        )
    )
    
    # DO NOT:
    # - Show a hard paywall
    # - Revoke any access
    # - Send urgent/scary language ("YOUR SUBSCRIPTION IS AT RISK")
    # - Send more than one notification per grace period
```

One gentle email. That's it. RC handles the retries. Your job is to make sure the user knows there's a path to update their payment if they want to, without creating anxiety about something that might resolve itself.

---

## Win-Back: The Sequence

```python
async def on_expiration(user_id: str, event: dict):
    await revoke_premium_access(user_id)
    
    # Cancel any in-flight winback sequences from cancellation intent
    await cancel_scheduled_sequences(user_id)
    
    # Start fresh winback sequence from actual expiry
    await schedule_winback_sequence(user_id, [
        {"delay_hours": 2,   "template": "just_expired"},      # First touch: what they're missing
        {"delay_hours": 168, "template": "week_later"},         # One week: soft offer
        {"delay_hours": 720, "template": "month_later_final"},  # One month: final offer, then stop
    ])


WINBACK_TEMPLATES = {
    "just_expired": {
        "subject": "Your access has ended — here's what you had",
        "angle": "Loss framing — show what they built, what they used, what they're missing",
        "offer": None,  # No discount yet — just remind them of value
    },
    "week_later": {
        "subject": "Still thinking about it?",
        "angle": "Soft — 'we're here when you're ready', maybe 10% off",
        "offer": "10% off first month back",
    },
    "month_later_final": {
        "subject": "Last chance — 30% off to come back",
        "angle": "Clear final offer. After this, stop. Don't spam.",
        "offer": "30% off first month back",
    },
}
```

**The rule:** After the final win-back email, stop. Repeated emails after 30 days post-expiry have near-zero conversion and real cost (spam complaints, brand damage).

---

## On Cancellation: Don't Panic

```python
async def on_cancellation(user_id: str, event: dict):
    # Access still active until expiration_at
    expiry_ms = event.get("expiration_at_ms", 0)
    days_remaining = max(0, (expiry_ms / 1000 - time.time()) / 86400)
    
    # Log the intent — useful for churn analysis
    await log_churn_intent(user_id, days_remaining=days_remaining)
    
    # Schedule a single check-in AFTER access ends, not before
    # (Contacting them before expiry is intrusive — they made a decision)
    await schedule_task(
        task=lambda: start_winback_sequence(user_id),
        delay_hours=(days_remaining * 24) + 2  # 2h buffer after actual expiry
    )
    
    # DO NOT send anything now. Respect the decision.
```

One exception: if you have a genuine product update or new feature launching within their remaining access window, a single "before you go, we just shipped X" message is acceptable. Not a discount plea — actual news.

---

## Instrumentation

```python
async def churn_health_report() -> dict:
    thirty_days_ago = datetime.utcnow() - timedelta(days=30)
    
    return {
        # What % of billing issues resolved without intervention?
        "grace_period_recovery_rate": ...,
        
        # What % of expired users came back within 30d?
        "winback_conversion_rate": ...,
        
        # Which win-back email converts best?
        "winback_by_template": {
            "just_expired": ...,
            "week_later": ...,
            "month_later_final": ...,
        },
        
        # How many users did we lose to billing (preventable)?
        "billing_churn_count": ...,
    }
```

If `grace_period_recovery_rate` is below 60%, your payment retry settings or notification timing may be off. If `winback_conversion_rate` is below 5%, either the offer isn't compelling or you're reaching out too late.

---

## The Honest Rule

Most churn is a product problem, not a pricing problem. Win-back offers and intervention sequences can recover 5-15% of churned users. The other 85-95% churned because the product didn't deliver enough value.

Invest in win-back sequences because the math is good on recovered users (high LTV, low CAC). But don't let win-back work become a substitute for understanding why people left.

---

*See [paywall-patterns/08-churn-intervention.md](https://github.com/zarpa-cat/paywall-patterns/blob/main/patterns/08-churn-intervention.md) for SDK implementation details.*
