# Operations: Instrumentation

> Agents can't notice things are wrong. The monitoring has to be explicit.

---

## What to Measure and When

### Every cycle (or daily)

**Subscription health via RC Charts API:**

```python
async def daily_health_check(project_id: str) -> HealthReport:
    # Rate limit: 5 req/min — batch these
    overview = await rc_mcp.call("mcp_RC_get_overview_metrics", {"project_id": project_id})
    
    return HealthReport(
        mrr=overview["mrr"],
        active_subscribers=overview["active_subscribers"],
        active_trials=overview["active_trials"],
        churn_rate=overview["churn_rate"],
    )
```

**Thresholds worth alerting on:**
- MRR drops >5% day-over-day → investigate (could be a billing issue, could be a bad deploy)
- Churn rate spikes >2x weekly average → something changed (pricing? UX? competition?)
- Trial conversion rate drops below 15% → paywall or onboarding may be broken

### Weekly

```python
async def weekly_summary(project_id: str) -> str:
    # Pull 30-day chart data for trends
    mrr_chart = await rc_mcp.call("mcp_RC_get_chart_data", {
        "project_id": project_id,
        "chart_name": "mrr",
        "realtime": False  # Required for sandbox; good practice for production too
    })
    churn_chart = await rc_mcp.call("mcp_RC_get_chart_data", {
        "project_id": project_id,
        "chart_name": "churn_rate",
        "realtime": False
    })
    # Format and store/send
```

### On deploy / config change

Run [subscription-sanity](https://github.com/zarpa-cat/subscription-sanity) to catch config drift:

```bash
python subscription_sanity.py --project-id projXXX
```

Checks: orphaned products, entitlements without products, offerings without packages, missing is_current, etc.

---

## The RC Rate Limit Problem

Charts API: **5 requests/minute**.

For an agent making health checks, this is tight. Cache aggressively:

```python
import time

_metrics_cache = {}
CACHE_TTL = 900  # 15 minutes

async def get_metrics_cached(project_id: str, metric: str) -> dict:
    cache_key = f"{project_id}:{metric}"
    cached = _metrics_cache.get(cache_key)
    
    if cached and time.time() - cached["timestamp"] < CACHE_TTL:
        return cached["data"]
    
    data = await fetch_metric(project_id, metric)
    _metrics_cache[cache_key] = {"data": data, "timestamp": time.time()}
    return data
```

Batch your metric calls: don't make 5 separate calls; make one `get_overview_metrics` call that returns the full dashboard snapshot.

---

## Event-Driven vs. Polling

**Never poll for subscription state changes.** Use webhooks.

Polling approach (bad for agents):
```
Every 5 minutes: check all customers for status changes
→ N customers × polling = N API calls
→ Will hit rate limits at scale
→ Misses events between polls
```

Webhook approach (right for agents):
```
RC calls your endpoint when anything changes
→ Zero polling overhead
→ Instant event processing
→ Scales to any customer count
```

See [webhooks.md](webhooks.md) for the full handler implementation.

---

## Logging Everything

The agent won't remember a billing anomaly from last week unless it wrote it down. Log:

```python
# Minimal audit log for every subscription event
async def log_subscription_event(event_type: str, user_id: str, metadata: dict):
    await db.execute("""
        INSERT INTO subscription_events (event_type, user_id, metadata, created_at)
        VALUES (?, ?, ?, ?)
    """, [event_type, user_id, json.dumps(metadata), datetime.utcnow()])
```

This log is your debugging surface when something goes wrong. RC's dashboard shows events, but your logs tie them to your application context (which feature was in use, what the user was trying to do, what error they saw).

---

## The Minimal Monitoring Stack

```
Webhook handler → events logged to SQLite
Daily cron → health check via RC Charts API (cached)
Weekly cron → summary report (stored in memory/)
On config change → subscription-sanity audit
```

This covers the essentials without overengineering. Add Datadog or Grafana if you need it — but for an agent-operated app in early stages, SQLite + cron is enough.
