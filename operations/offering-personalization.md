# Operations: Offering Personalization

Agents can assign different paywalls to different users based on behavior signals — without a formal A/B experiment. This is per-customer targeting, and it's fully automatable via RC's API.

---

## The Mechanism

```bash
# Assign a specific offering to a customer
POST /v2/projects/{project_id}/customers/{app_user_id}/actions/assign_offering
{ "offering_id": "ofrng_xxx" }
```

After this call, `Purchases.shared.offerings().current` on that user's device returns the assigned offering, not the project default. RC handles the routing.

---

## When to Use It (vs. Formal Experiments)

| Situation | Use |
|-----------|-----|
| Testing two prices on random users | RC Experiments (A/B) |
| Showing win-back offer to churned users | Per-customer assignment |
| Annual push for high-engagement users | Per-customer assignment |
| Segment pricing (B2B vs. consumer) | Per-customer assignment |
| Statistical significance needed | RC Experiments |
| You control the logic | Per-customer assignment |

Per-customer assignment is immediate, fully programmable, and requires no dashboard setup. The tradeoff: no built-in analytics. You track conversion yourself.

---

## Signal → Offering Map

Design your assignments around behavioral signals, not demographics:

```python
OFFERING_MAP = {
    "default":     "ofrng_default",      # Standard pricing
    "annual_push": "ofrng_annual_push",  # Monthly hidden, annual prominent
    "winback":     "ofrng_winback",      # 30% off first month, expires 7d
    "high_intent": "ofrng_high_intent",  # Higher price point ($7.99 vs $5.99)
}

async def assign_offering_for_user(user_id: str, signals: UserSignals) -> str:
    """Determine and assign the right offering based on behavioral signals."""
    
    if signals.is_recently_churned and signals.days_since_expiry < 30:
        offering_key = "winback"
    
    elif signals.sessions_last_30d > 20 and signals.items_created > 50:
        # Heavy user: push annual
        offering_key = "annual_push"
    
    elif signals.sessions_last_30d > 10 and not signals.has_seen_paywall:
        # Engaged but not yet shown paywall: try higher price
        offering_key = "high_intent"
    
    else:
        offering_key = "default"
    
    offering_id = OFFERING_MAP[offering_key]
    await assign_offering(user_id, offering_id)
    
    return offering_key


async def assign_offering(user_id: str, offering_id: str):
    async with httpx.AsyncClient() as client:
        await client.post(
            f"{RC_BASE_URL}/customers/{user_id}/actions/assign_offering",
            headers={"Authorization": f"Bearer {RC_SECRET_KEY}"},
            json={"offering_id": offering_id}
        )
```

---

## When to Re-Evaluate

Assignment isn't permanent — re-evaluate at lifecycle moments:

```python
# Triggers to re-run offering assignment:
REASSIGNMENT_TRIGGERS = [
    "user_login",           # On each login — signals may have changed
    "session_milestone",    # After N sessions (5, 10, 20)
    "subscription_expiry",  # User just churned → winback
    "trial_started",        # In trial → don't assign winback
    "trial_cancelled",      # Canceled trial → winback immediately
]
```

Don't set-and-forget. A user who was "low intent" six months ago may be a power user now. Stale assignments show the wrong paywall.

---

## Offering Metadata: Dynamic Copy

RC offerings carry arbitrary JSON metadata. Use this to drive paywall copy from the server side — no app update needed to change your headline, CTA, or feature list.

```bash
# Set metadata on an offering
PATCH /v2/projects/{project_id}/offerings/{offering_id}
{
  "metadata": {
    "headline": "You've been with us for 60 days. Here's a thank-you.",
    "cta_monthly": "Keep going",
    "cta_annual": "Lock in your rate",
    "highlight": "annual",
    "show_savings_badge": true
  }
}
```

An agent can update this metadata via API to change messaging for an entire segment, instantly, without touching client code. This is the fastest iteration cycle possible for paywall optimization.

---

## Tracking Conversion by Offering

Since per-customer assignment bypasses RC's built-in experiment analytics, track it yourself:

```python
async def log_paywall_event(user_id: str, offering_key: str, event: str):
    """Track paywall impressions and conversions by offering."""
    await db.execute("""
        INSERT INTO paywall_events (user_id, offering_key, event, created_at)
        VALUES (?, ?, ?, ?)
    """, [user_id, offering_key, event, datetime.utcnow()])

# On paywall impression:
await log_paywall_event(user_id, "winback", "impression")

# On purchase:
await log_paywall_event(user_id, "winback", "conversion")

# Weekly: conversion rate by offering
# SELECT offering_key, 
#        SUM(CASE WHEN event='conversion' THEN 1 ELSE 0 END) * 100.0 / 
#        SUM(CASE WHEN event='impression' THEN 1 ELSE 0 END) AS conversion_rate
# FROM paywall_events GROUP BY offering_key
```

---

## RC Notes

- Assignment via `/actions/assign_offering` (note: `/actions/` prefix — the RC pattern for non-CRUD operations)
- Assignment survives across devices for the same App User ID
- Anonymous users get assignment per-device until `logIn()` is called
- `Purchases.shared.invalidateCustomerInfoCache()` after assignment if you need immediate reflection in the current session
