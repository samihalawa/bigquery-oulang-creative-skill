---
name: bigquery-oulang-creative
description: >
  Analyze Oulang platform data entirely through BigQuery. Covers live MySQL (federated),
  PostHog events (batch export), and all analytics. Uses curl or bq CLI.
  Triggers: "analyze oulang", "BQ analysis", "profit opportunities", "revenue analysis",
  "bigquery oulang", "find opportunities", "platform analysis", "monetization",
  "growth analysis", "conversion funnel", "user behavior", "listing economics", "oulang data".
---

# BigQuery Oulang Analysis – Agent Instructions

**All queries go through BigQuery. No direct MySQL connections. No PostHog API calls.**

## Environment

- GCP Project: `megacursos`
- BQ Location: `us-east1`
- Federated MySQL connection: `megacursos.us-east1.oulang-mysql-live`
- PostHog export dataset: `megacursos.posthog_oulang` (events, persons — hourly batch)
- Snapshot dataset: `megacursos.oulang_analysis` (legacy copies + posthog summaries)

## How to Query

### curl (works everywhere)
```bash
TOKEN=$(gcloud auth print-access-token)
curl -s -X POST "https://bigquery.googleapis.com/bigquery/v2/projects/megacursos/queries" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"query":"YOUR SQL","useLegacySql":false,"location":"us-east1"}'
```
Response: `.rows[].f[].v` for values, `.schema.fields[].name` for column names.

### bq CLI (if available)
```bash
bq query --use_legacy_sql=false --location=us-east1 'YOUR SQL'
```

## Querying MySQL (Live)

All MySQL tables are queryable live via federated connection — no sync, no staleness:
```sql
SELECT * FROM EXTERNAL_QUERY(
  "megacursos.us-east1.oulang-mysql-live",
  "SELECT COUNT(*) FROM users WHERE createdAt > DATE_SUB(NOW(), INTERVAL 7 DAY) LIMIT 100"
)
```
Put all WHERE/LIMIT/JOIN inside the quoted MySQL string. 90+ tables available including:
orders, users, listings, credit_transactions, payments, user_subscriptions,
categories, regions, view_history, advertisements, boost_plans, messages,
favorites, listing_contact_reveals, listing_views, listing_bumps, tv_dramas,
tv_episodes, businesses, pricing_plans, ad_subscriptions, game_transactions,
game_wallets, notifications, content_reports, moderation_queue, campaigns.

## Querying PostHog

PostHog events and persons are batch-exported to `megacursos.posthog_oulang`:
```sql
SELECT event, COUNT(*) as cnt
FROM `megacursos.posthog_oulang.events`
WHERE timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY event ORDER BY cnt DESC LIMIT 20
```
If `posthog_oulang` is empty, use legacy summaries:
```sql
SELECT * FROM `megacursos.oulang_analysis.posthog_event_summary` ORDER BY cnt DESC LIMIT 30
```
240 event definitions including $autocapture, $pageview, $identify, and custom events.

## Combining MySQL + PostHog

Join live MySQL data with PostHog in a single BQ query:
```sql
WITH ph AS (
  SELECT JSON_VALUE(properties, '$.user_id') as uid, COUNT(*) as events
  FROM `megacursos.posthog_oulang.events`
  WHERE timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  GROUP BY 1
),
mysql AS (
  SELECT * FROM EXTERNAL_QUERY("megacursos.us-east1.oulang-mysql-live",
    "SELECT id, nickname, phone, createdAt FROM users ORDER BY createdAt DESC LIMIT 1000")
)
SELECT m.id, m.nickname, COALESCE(ph.events, 0) as posthog_events
FROM mysql m LEFT JOIN ph ON CAST(m.id AS STRING) = ph.uid
ORDER BY posthog_events DESC LIMIT 20
```

## Do Not

- Query MySQL directly (use EXTERNAL_QUERY through BQ)
- Call PostHog API directly (use BQ exported tables)
- Set up sync pipelines, Datastream, or scheduled queries
- Run unfiltered full-table scans without WHERE and LIMIT
- Suggest alternatives to BigQuery

## Goal

Find profit opportunities, detect anomalies, and surface actionable business insights
from the Oulang platform. Cross-reference transactional data (MySQL via federated query)
with behavioral data (PostHog via BQ export) — all within BigQuery.
