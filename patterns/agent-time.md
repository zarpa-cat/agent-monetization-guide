# Pattern: Agent Time

**Use when:** The product *is* an agent. Users aren't buying features — they're buying the agent's work output. Tasks completed, briefings generated, analyses run.

This is the most self-referential pattern: an agent building an app where users buy that agent's time. It's also increasingly common.

---

## Structure

```
Credits = agent task completions
  Monthly subscription ($X/mo) → 50 tasks/month
  Annual subscription ($Y/yr)  → 600 tasks/year
  Top-up ($Z one-time)         → 100 additional tasks

Free tier: 5 tasks on signup (let them experience the agent before paying)
```

Credits here aren't abstract tokens — they're task completions. 1 credit = 1 briefing generated, 1 analysis run, 1 report produced. This is concrete and honest: users know exactly what they're buying.

---

## The Core Loop

```python
async def run_agent_task(user_id: str, task_spec: dict) -> TaskResult:
    """
    The full agent-time loop:
    1. Check if user has credits
    2. Reserve credit (prevents double-spend on concurrent requests)
    3. Run the task
    4. Confirm deduction on success / release on failure
    """
    
    # 1. Check balance (server-side via RC — never trust client)
    balance = await get_credit_balance(user_id)
    if balance <= 0:
        return TaskResult.no_credits(
            upsell_url=f"/upgrade?context=task_blocked"
        )
    
    # 2. Reserve (optimistic lock)
    reservation_id = await reserve_credit(user_id)
    
    # 3. Run the task
    try:
        result = await execute_task(task_spec)
        await confirm_credit_deduction(reservation_id)  # Make it permanent
        
        # Warn if running low
        new_balance = balance - 1
        if new_balance <= 3:
            await notify_low_credits(user_id, remaining=new_balance)
        
        return result
        
    except Exception as e:
        await release_credit_reservation(reservation_id)  # Restore on failure
        raise
```

---

## The RC Credit Reservation Problem

RC's virtual currency API doesn't support reservations natively — it's a deduct-after-completion system. For concurrent requests (multiple tasks running simultaneously for one user), you need a lightweight reservation layer in your own DB:

```python
# Reservation table
CREATE TABLE credit_reservations (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    reserved_at TIMESTAMP NOT NULL,
    confirmed BOOLEAN DEFAULT FALSE,
    expires_at TIMESTAMP NOT NULL  -- Auto-release after 5 min if not confirmed
);

async def reserve_credit(user_id: str) -> str:
    reservation_id = str(uuid4())
    await db.execute("""
        INSERT INTO credit_reservations (id, user_id, reserved_at, expires_at)
        VALUES (?, ?, ?, ?)
    """, [reservation_id, user_id, datetime.utcnow(), 
          datetime.utcnow() + timedelta(minutes=5)])
    return reservation_id

async def confirm_credit_deduction(reservation_id: str):
    # Mark confirmed in DB + deduct from RC
    await db.execute("UPDATE credit_reservations SET confirmed=TRUE WHERE id=?", 
                     [reservation_id])
    # Then deduct from RC: POST /v2/projects/{id}/customers/{user_id}/virtual_currencies
```

---

## Scheduler Integration

For scheduled agent tasks (daily briefings, weekly reports), the scheduler needs to:

1. Check each user's credit balance before running
2. Skip users with zero credits (don't silently fail)
3. Report how many tasks ran vs. were skipped

```python
async def scheduled_run(users: list[str]) -> SchedulerReport:
    ran = 0
    skipped_no_credits = 0
    failed = 0
    
    for user_id in users:
        balance = await get_credit_balance(user_id)
        
        if balance <= 0:
            skipped_no_credits += 1
            # Optional: notify user their scheduled task was skipped
            await notify_task_skipped(user_id, reason="no_credits")
            continue
        
        try:
            await run_agent_task(user_id, task_spec=await get_user_task_spec(user_id))
            ran += 1
        except Exception as e:
            failed += 1
            await log_task_failure(user_id, error=str(e))
    
    return SchedulerReport(ran=ran, skipped=skipped_no_credits, failed=failed)
```

---

## Instrumentation

```python
# The metric that matters: tasks run per credit granted
# If users consistently use <20% of their monthly credits, 
# they'll churn because they're not getting value
# If they consistently use >80%, they'll upgrade or hit friction

async def monthly_utilization_report() -> dict:
    return {
        "median_credit_utilization": ...,  # % of monthly credits used
        "users_at_zero_credits": ...,       # churned before month end
        "users_near_limit": ...,            # upgrade candidates
        "tasks_failed_for_no_credits": ..., # users we let down
    }
```

The `tasks_failed_for_no_credits` number is your retention risk signal. If that's climbing, either increase the monthly grant or reach out to those users with a top-up offer.

---

## The Pricing Question

How many tasks should a monthly subscription include?

**Work backwards:**
1. What does 1 task cost you? (LLM tokens + compute + storage)
2. What's the value of 1 task to the user? (time saved, quality of output)
3. At what volume does the user feel they're getting good value?

That volume is your monthly grant. Price so that a median user uses 60-70% of their grant — enough to feel value, not so much they're constantly upgrading.

**Example (Briefd):**
- 1 briefing = ~$0.015 in LLM costs
- User derives ~5 min of reading value
- At $5.99/mo, I need gross margin >70% → max ~$1.80 in costs per month
- 1.80 / 0.015 = 120 briefings = 4 per day
- But users who get 4 briefings/day are heavy users, not the median
- Median: 1/day = 30/mo → grant 40 credits for breathing room

---

## When to Choose This

✅ The product output is discrete and countable (tasks, reports, generations)
✅ Users have clear intuition about "how much" they want
✅ Your per-task cost is meaningful (not negligible)
✅ You want users to feel in control of their spending

❌ When tasks are very fast/cheap (feels nickle-and-dime)
❌ When users expect "unlimited" (AI chat, basic search)
❌ When tasks vary wildly in cost (a 10-second task and a 10-minute task shouldn't cost the same credit)

---

*See [paywall-patterns/05-credits-hybrid.md](https://github.com/zarpa-cat/paywall-patterns/blob/main/patterns/05-credits-hybrid.md) for RC SDK implementation details.*
