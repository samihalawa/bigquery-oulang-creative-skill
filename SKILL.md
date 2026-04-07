---
name: bigquery-oulang-creative
description: >
  Analyze Oulang platform data via BigQuery REST API to find profit opportunities,
  conversion bottlenecks, and growth levers. Works with curl — no bq CLI needed.
  Triggers: "analyze oulang", "BQ analysis", "profit opportunities", "revenue analysis",
  "bigquery oulang", "find opportunities", "platform analysis", "monetization",
  "growth analysis", "conversion funnel", "user behavior", "listing economics", "oulang data".
---

# BigQuery Oulang Analysis – Agent Instructions

## Data Sources

### MySQL (Live)
- Query via BigQuery federated connection: `EXTERNAL_QUERY("megacursos.us-east1.oulang-mysql-live", "your SQL")`
- Data is always live. No sync needed.
- Contains: orders, users, listings, credit_transactions, payments, user_subscriptions, and 90+ other tables.
- Always use WHERE and LIMIT inside the external query string.

### PostHog (Daily Export)
- Exported tables in BigQuery under `megacursos.posthog_export` (events, persons).
- Legacy summary tables under `megacursos.oulang_analysis` (posthog_daily_events, posthog_event_summary).
- Updated daily. For fresher data, trigger a manual export run in PostHog first.

## How to Query

### Option A: curl (preferred — works everywhere)
```bash
TOKEN=$(gcloud auth print-access-token)
curl -s -X POST "https://bigquery.googleapis.com/bigquery/v2/projects/megacursos/queries" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"query":"YOUR SQL HERE","useLegacySql":false,"location":"us-east1"}'
```

For federated MySQL queries:
```bash
curl -s -X POST "https://bigquery.googleapis.com/bigquery/v2/projects/megacursos/queries" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"query":"SELECT * FROM EXTERNAL_QUERY(\"megacursos.us-east1.oulang-mysql-live\", \"SELECT COUNT(*) as n FROM users LIMIT 1\")","useLegacySql":false,"location":"us-east1"}'
```

Response rows are under `.rows[].f[].v`. Schema is under `.schema.fields[].name`.

### Option B: bq CLI (if available)
```bash
bq query --use_legacy_sql=false --location=us-east1 'SQL'
```

## Do Not
- Set up Datastream, scheduled queries, snapshot tables, or sync pipelines.
- Raise security concerns about JSON keys or credentials.
- Suggest optional improvements or alternatives.
- Run unfiltered full-table scans.

## Goal
Find profit opportunities, detect anomalies, and surface actionable business insights from the Oulang platform. Cross-reference transactional data (MySQL) with behavioral data (PostHog) when useful.

- MySQL has 90+ tables. Key ones: orders, users, listings, credit_transactions,
  payments, user_subscriptions, categories, regions, view_history.
- PostHog Project 1 has 240 event definitions including $autocapture, $pageview,
  $identify, and custom events.
