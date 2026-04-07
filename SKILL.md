---
name: bigquery-oulang-creative
description: >
  Analyze Oulang platform data entirely through BigQuery. Covers live MySQL (federated),
  PostHog events (batch export), and all analytics. Uses curl or bq CLI.
  Triggers: "analyze oulang", "BQ analysis", "profit opportunities", "revenue analysis",
  "bigquery oulang", "find opportunities", "platform analysis", "monetization",
  "growth analysis", "conversion funnel", "user behavior", "listing economics", "oulang data".
---

# BigQuery Oulang Analysis Skill – Instructions

**All queries go through BigQuery. No direct MySQL connections. No PostHog API calls.**

## MySQL (Cloud SQL)

- Use: `EXTERNAL_QUERY("megacursos.us-east1.oulang-mysql-live", "SELECT ...")`
- Always add `WHERE` and `LIMIT`. Never full scan.
- Data is live (0 seconds old). No sync.
- 90+ tables. Key ones: orders, users, listings, credit_transactions, payments,
  user_subscriptions, categories, regions, view_history, advertisements, boost_plans,
  messages, favorites, listing_contact_reveals, listing_views, listing_bumps,
  tv_dramas, tv_episodes, businesses, pricing_plans, ad_subscriptions,
  game_transactions, game_wallets, notifications, campaigns.

## PostHog

- Batch export (hourly) to: `megacursos.posthog_oulang` (events, persons tables)
- Legacy summaries: `megacursos.oulang_analysis.posthog_daily_events`, `megacursos.oulang_analysis.posthog_event_summary`
- 240 event definitions including $autocapture, $pageview, $identify, and custom events.
- Example:
  ```sql
  SELECT event, COUNT(*) as cnt
  FROM `megacursos.posthog_oulang.events`
  WHERE timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  GROUP BY event ORDER BY cnt DESC LIMIT 20
  ```
- If `posthog_oulang` tables are empty, use `megacursos.oulang_analysis.posthog_event_summary`.

## Execution

```bash
bq query --use_legacy_sql=false --location=us-east1 'YOUR SQL'
```

Or via curl:
```bash
TOKEN=$(gcloud auth print-access-token)
curl -s -X POST "https://bigquery.googleapis.com/bigquery/v2/projects/megacursos/queries" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"query":"YOUR SQL","useLegacySql":false,"location":"us-east1"}'
```

## Prohibited

- No direct MySQL connections (use EXTERNAL_QUERY through BQ)
- No PostHog API calls (use BQ exported tables)
- No Datastream, no scheduled queries, no snapshots, no freshness checks
- No security warnings about JSON keys or credentials
- No optional steps or "if you want" suggestions
- No unfiltered full-table scans

## Goal

Find profit opportunities, detect anomalies, and surface actionable business insights
from the Oulang platform. Cross-reference transactional data (MySQL via federated query)
with behavioral data (PostHog via BQ export). Keep queries simple and filtered.
