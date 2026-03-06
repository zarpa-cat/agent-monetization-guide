# Pattern: Subscription + Credits (Hybrid)

**Use when:** Your app has AI features with real inference costs that vary significantly by user.

**The problem it solves:** Some users want unlimited access (subscribe once, forget it). Others use your app occasionally and resent paying monthly for something they use twice. Pure subscription loses the occasional users. Pure usage-based loses the power users who want price certainty.

Hybrid gives both groups what they want.

---

## Structure

```
Monthly subscription ($X/mo)
  → grants 100 credits/month (resets at billing cycle)
  → subscribers never see a "you're out of credits" message in normal use

Annual subscription ($Y/yr)
  → grants 1200 credits (equivalent to 100/mo, no mid-year reset)

Credit top-up ($Z one-time)
  → 500 additional credits, don't expire

Free tier
  → 10 free credits on signup (let them try before asking for money)
```

---

## The Credit Design Decision

**What does 1 credit cost to you?**

Work backwards from your LLM costs:
- GPT-4o: ~$2.50/1M input tokens, ~$10/1M output tokens
- A typical response: ~500 input + 800 output tokens ≈ $0.009
- At 1 credit = 1 response: 100 credits = ~$0.90 in compute

Your $5.99/mo subscription grants 100 credits = you're spending ~$0.90 on compute, keeping ~$5.09.

That math works until users start using it heavily. Check your 90th percentile user's monthly usage — if heavy users break the model, cap credits more aggressively or tier them by plan.

---

## RevenueCat Implementation

```python
# rc-agent-starter --with-credits handles this setup
# Manual steps:

# 1. Create virtual currency
POST /v2/projects/{project_id}/virtual_currencies
{
  "name": "Credits",
  "code": "CRED",
  "description": "1 credit = 1 AI generation"
}

# 2. Grant credits via subscription
POST /v2/projects/{project_id}/virtual_currencies/{vc_id}/product_grants
{
  "product_ids": ["prod_monthly_id"],
  "amount": 100,
  "expire_at_cycle_end": true   # Reset monthly — don't let credits accumulate
}
```

Key RC fields:
- `expire_at_cycle_end: true` — credits reset each billing cycle. Without this, long-term subscribers build an enormous balance that makes them churn-proof in a bad way.
- `trial_amount` — credits granted during a free trial. Set lower than paid amount to incentivize conversion.

---

## Agent Operations

The agent checks credit balance before each billable operation:

```python
async def run_billable_operation(user_id: str, operation: Callable) -> Result:
    # Check balance via RC REST API (server-side — don't trust client)
    balance = await get_credit_balance(user_id)
    
    if balance <= 0:
        # Zero credits — show upsell, don't silently fail
        return Result.insufficient_credits(upsell_offering_id="ofrng_xxx")
    
    result = await operation()
    await deduct_credit(user_id, amount=1)
    
    # Warn if approaching limit
    if balance == 5:
        await send_low_credits_notification(user_id)
    
    return result
```

**The agent-specific discipline:** Deduct credits *after* a successful operation, not before. If the operation fails, the user shouldn't be charged. Log every deduction — if there's a billing dispute, you need the audit trail.

---

## Monitoring

```python
# Weekly: check if credit economics are working
async def check_credit_health():
    # Average credits consumed per subscriber per month
    # vs. credits granted per subscription
    # If consumed > granted by > 20%, your pricing is off
    pass

# On RENEWAL webhook: grant new credits
async def on_subscription_renewal(user_id: str, product_id: str):
    await grant_credits(user_id, amount=100)  # Or read from product config
```

---

## When to Choose This

✅ AI features (LLM calls, image generation, API-heavy operations)
✅ B2B SaaS where users have variable usage patterns
✅ When you want to serve both casual and power users at appropriate price points

❌ Simple subscription apps with flat feature access
❌ When you can't measure per-user operation costs accurately
❌ When your user base expects unlimited access (fitness apps, note-taking)
