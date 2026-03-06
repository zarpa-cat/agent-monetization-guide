# Pattern: Freemium Gate

**Use when:** Broad top-of-funnel matters. The free tier is real and useful — not a crippled demo. Users hit limits naturally as they get more value.

**The agent-specific tension:** Freemium is the hardest pattern for agents to manage because the limit design requires judgment about user behavior. The wrong limit (too low or too high) either kills acquisition or kills conversion — and an agent can't intuit which is happening without explicit instrumentation.

---

## Structure

```
Free tier (no RC involvement — just your app logic):
  - 5 items / 3 projects / 10 API calls / whatever unit makes sense
  - Core workflow accessible
  - Export or sync blocked (premium features)

Premium entitlement:
  - Unlocks everything, removes limits
  - Monthly + Annual offerings
```

---

## The Limit Design Framework

Before writing any code, answer these:

**1. What is the unit of value?** (items, projects, API calls, exports, seats)

**2. At what usage level does a user understand the product?**
That's your free tier ceiling. Not one below it — AT that level.

**3. Where do users want to go next?**
That's your upsell moment. The paywall appears when they try to cross from "understanding it" to "relying on it."

**RC's `Offering` metadata** can store these limits — change them without an app update:
```bash
PATCH /v2/projects/{project_id}/offerings/{offering_id}
{ "metadata": { "free_item_limit": 5, "free_export_enabled": false } }
```

---

## RC Role

RC only manages the premium layer. The free tier is pure app logic.

```python
# The two checks an agent runs for every action
async def can_perform_action(user_id: str, action: str) -> tuple[bool, str | None]:
    # Check 1: are they premium? (RC)
    info = await get_customer_info(user_id)  # cached
    if info.entitlements.get("premium", {}).get("is_active"):
        return True, None
    
    # Check 2: are they within free limits? (your DB)
    usage = await get_usage(user_id, action)
    limit = await get_free_limit(action)  # from RC offering metadata or config
    
    if usage < limit:
        return True, None
    
    return False, "upgrade"
```

---

## The Pre-Warn Pattern (Critical)

Don't let users hit the wall without warning. When they're at 80% of their limit, tell them. The conversion rate at "1 remaining" is significantly higher than at "limit hit."

```python
async def check_and_warn(user_id: str, action: str, after_action: bool = False):
    usage = await get_usage(user_id, action)
    limit = await get_free_limit(action)
    remaining = limit - usage
    
    if remaining == 1:
        # One left — soft in-app notice, not a paywall
        await show_notice(user_id, f"1 {action} remaining on your free plan.")
    elif remaining == 0 and after_action:
        # Just used their last one — now is the moment
        await show_upgrade_prompt(user_id, context=action)
```

---

## Instrumentation for Agents

The metric agents need to watch: **where are users hitting limits?**

```python
# Log every limit encounter
async def log_limit_hit(user_id: str, action: str, limit: int):
    await db.execute("""
        INSERT INTO limit_events (user_id, action, limit_value, hit_at)
        VALUES (?, ?, ?, ?)
    """, [user_id, action, limit, datetime.utcnow()])
    
# Weekly: what % of active free users hit a limit?
# If >50%: limit is too low or value isn't getting through
# If <5%: limit is too high, or users are churning before hitting it
```

If the conversion rate from limit-hit to subscribe is below 5%, the limit placement is wrong — not the product. Change the limit before blaming the paywall.

---

## When to Choose This

✅ Consumer apps needing broad acquisition (habit trackers, note apps, utilities)
✅ B2C where you can't qualify buyers in advance
✅ Markets where competitors offer freemium (baseline expectation)

❌ B2B SaaS where the buyer is pre-qualified (just do a free trial)
❌ Apps where the core loop isn't complete without premium features
❌ When your free tier is so good it cannibalizes paid (reassess the limit design)

---

*See [paywall-patterns/03-freemium-gate.md](https://github.com/zarpa-cat/paywall-patterns/blob/main/patterns/03-freemium-gate.md) for full SDK implementation.*
