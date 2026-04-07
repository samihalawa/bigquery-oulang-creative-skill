---
name: bigquery-oulang-analysis
description: >
  Analyze Oulang platform data via BigQuery for profit opportunities, conversion bottlenecks,
  and growth levers. Uses `bq` CLI (authenticated via gcloud). Triggers: "analyze oulang",
  "BQ analysis", "profit opportunities", "revenue analysis", "bigquery oulang",
  "find opportunities", "platform analysis", "monetization", "growth analysis",
  "conversion funnel", "user behavior", "listing economics", "oulang data".
---

# BigQuery Oulang Analysis Skill

## When To Use

Use this skill whenever asked to analyze the Oulang platform's data, find profit opportunities,
audit monetization, or investigate user behavior. This is the **primary data analysis path** —
do not query MySQL directly or PostHog API unless BigQuery cannot answer the question.

## Architecture (Important)

All data is accessible from BigQuery via two methods:

### 1. Live MySQL via Federated Query (preferred — always fresh)

```sql
bq query --use_legacy_sql=false --location=us-east1 '
SELECT * FROM EXTERNAL_QUERY(
  "megacursos.us-east1.oulang-mysql-live",
  "YOUR_MYSQL_QUERY_HERE"
)'
```

This queries the production Cloud SQL MySQL (`oulang-db`) in real-time. No sync lag.
**Always use `--location=us-east1`** for federated queries.

Example — live user count:
```bash
bq query --use_legacy_sql=false --location=us-east1 '
SELECT * FROM EXTERNAL_QUERY(
  "megacursos.us-east1.oulang-mysql-live",
  "SELECT COUNT(*) as total_users FROM users WHERE createdAt > DATE_SUB(NOW(), INTERVAL 7 DAY)"
)'
```

### 2. Snapshot Tables (for heavy analytics — may be up to a few days old)

Dataset `megacursos.oulang_analysis` contains snapshot copies of all MySQL tables plus PostHog
summary tables. Use these for heavy JOINs or window functions that would be slow over federated queries.

Key tables: `users`, `listings`, `categories`, `regions`, `orders`, `credit_transactions`,
`payments`, `boost_plans`, `advertisements`, `messages`, `favorites`, `listing_views`,
`listing_bumps`, `listing_contact_reveals`, `view_history`, `tv_dramas`, `tv_episodes`,
`user_subscriptions`, `pricing_plans`, `posthog_daily_events`, `posthog_event_summary`

Check freshness first:
```bash
bq query --use_legacy_sql=false 'SELECT table_id, row_count, TIMESTAMP_MILLIS(last_modified_time) as last_mod FROM `megacursos.oulang_analysis.__TABLES__` ORDER BY last_modified_time DESC LIMIT 10'
```

### 3. PostHog Data

PostHog batch exports are configured (hourly) to `megacursos.posthog_oulang` dataset (events + persons models).
If that dataset is empty or stale, fall back to:
- `megacursos.oulang_analysis.posthog_event_summary` (aggregated event counts)
- `megacursos.oulang_analysis.posthog_daily_events` (daily event breakdown)
- PostHog API only as last resort: `https://posthog.pime.ai/api/projects/1/query/` with Bearer `$POSTHOG_API_KEY (set via env or check project .env)`

## Execution Rules

1. **Always use shell command execution tools** (serena:execute_shell_command, Desktop Commander, bash) — never create Python scripts for queries
2. **Check data freshness** before using snapshot tables — if >5 days old, use federated query instead
3. **Use `--format=csv`** for large result sets to keep output compact
4. **Limit results** with `LIMIT` or `--max_rows` to avoid context overflow
5. **Never assume schema** — run `bq show --schema --format=json megacursos:oulang_analysis.TABLE` first if unsure

## Analysis Playbook

When asked to "analyze" or "find opportunities", run these categories in order of revenue impact:

### A. Revenue Health (run first)
```bash
# Live payment + order status
bq query --use_legacy_sql=false --location=us-east1 '
SELECT * FROM EXTERNAL_QUERY("megacursos.us-east1.oulang-mysql-live",
  "SELECT
    (SELECT COUNT(*) FROM orders WHERE status=\"confirmed\") as confirmed_orders,
    (SELECT COUNT(*) FROM payments WHERE status=\"completed\") as completed_payments,
    (SELECT SUM(CAST(amount AS DECIMAL(10,2))) FROM credit_transactions WHERE type=\"purchase\") as purchased_credits,
    (SELECT SUM(CAST(amount AS DECIMAL(10,2))) FROM credit_transactions WHERE type=\"admin_grant\") as admin_granted_credits,
    (SELECT COUNT(*) FROM user_subscriptions WHERE status=\"active\") as active_subs,
    (SELECT COUNT(*) FROM ad_subscriptions WHERE status=\"active\") as active_ad_subs"
)'
```

### B. User Conversion Funnel
```bash
bq query --use_legacy_sql=false --format=csv 'WITH
  total AS (SELECT COUNT(*) as n FROM `megacursos.oulang_analysis.users`),
  posters AS (SELECT COUNT(DISTINCT userId) as n FROM `megacursos.oulang_analysis.listings`),
  buyers AS (SELECT COUNT(DISTINCT userId) as n FROM `megacursos.oulang_analysis.orders` WHERE status="confirmed"),
  revealers AS (SELECT COUNT(DISTINCT viewerUserId) as n FROM `megacursos.oulang_analysis.listing_contact_reveals`),
  messagers AS (SELECT COUNT(DISTINCT senderId) as n FROM `megacursos.oulang_analysis.messages`)
SELECT total.n as registered, posters.n as posted_listing, revealers.n as revealed_contact,
  messagers.n as sent_message, buyers.n as paid_order
FROM total, posters, buyers, revealers, messagers'
```

### C. Category Economics
```bash
bq query --use_legacy_sql=false --format=csv 'SELECT c.name, COUNT(l.id) as listings,
  COUNT(DISTINCT l.userId) as sellers,
  COUNTIF(l.status="active") as active,
  (SELECT COUNT(*) FROM `megacursos.oulang_analysis.listing_contact_reveals` r WHERE r.listingId = l.id) as reveals
FROM `megacursos.oulang_analysis.listings` l
JOIN `megacursos.oulang_analysis.categories` c ON c.id = l.categoryId
GROUP BY c.name ORDER BY listings DESC'
```

### D. Credit Economy
```bash
bq query --use_legacy_sql=false --format=csv 'SELECT type,
  COUNT(*) as txns, ROUND(SUM(CAST(amount AS FLOAT64)),0) as total_credits,
  COUNT(DISTINCT userId) as unique_users
FROM `megacursos.oulang_analysis.credit_transactions` GROUP BY type ORDER BY total_credits DESC'
```

### E. TV Engagement + Cross-sell
```bash
bq query --use_legacy_sql=false --format=csv 'WITH tv AS (
  SELECT DISTINCT userId FROM `megacursos.oulang_analysis.view_history` WHERE visitorId IS NOT NULL
)
SELECT COUNT(*) as tv_users,
  COUNTIF(userId IN (SELECT DISTINCT userId FROM `megacursos.oulang_analysis.listings`)) as also_posted,
  COUNTIF(userId IN (SELECT DISTINCT userId FROM `megacursos.oulang_analysis.orders`)) as also_ordered
FROM tv'
```

### F. Top Power Users (monetization targets)
```bash
bq query --use_legacy_sql=false --location=us-east1 --format=csv '
SELECT * FROM EXTERNAL_QUERY("megacursos.us-east1.oulang-mysql-live",
  "SELECT u.id, u.nickname, u.phone, COUNT(l.id) as listing_count,
    (SELECT SUM(CAST(amount AS DECIMAL(10,2))) FROM credit_transactions ct WHERE ct.userId=u.id AND ct.type=\"purchase\") as spent
  FROM users u JOIN listings l ON l.userId=u.id
  GROUP BY u.id ORDER BY listing_count DESC LIMIT 20"
)'
```

## Key Platform Facts (for context)

- ~12K users, ~26K listings, ~80K WeChat distribution via partner (Tony)
- Currently earning near €0 despite high engagement
- 84% of users post within 1 hour of signup (very high intent from WeChat)
- Credit economy is 84% subsidized (admin grants vs purchases)
- 招工 (jobs) is the #1 category by volume
- TV streaming has thousands of active viewers but zero monetization
- All categories have `requiresPayment=0` (everything free)
- RevenueCat Web Billing is integrated but has near-zero conversions

## Do NOT

- Create Python scripts to run queries — use `bq` CLI directly
- Assume table schemas — always verify with `bq show --schema`
- Present stale data without noting the snapshot date
- Run queries without `LIMIT` on tables with >100K rows
- Query MySQL directly via `cors.trigox.workers.dev/db` when BQ federated query works
- Over-detail the skill instructions — keep analysis creative and goal-oriented
