---
name: axiom-bot-dashboard
description: "Create or update a single-bot Axiom dashboard using verified successful throughput and the Chabot-style four-table layout: hourly counts, daily counts, cumulative timing, and daily timing. Use for bot dashboard creation or completion-query repairs."
---

# Axiom bot dashboard

Create a single-bot dashboard that counts successfully completed work using exactly four tables modeled on the Chabot Pharmacy dashboard: hourly counts, daily counts, a cumulative timing summary, and daily timing averages. Adapt the counted unit to the bot: prescriptions, orders, documents, jobs, or the unit requested by the user. Do not assume every bot emits Chabot's marker or uses its schema.

## Find what actually means success

- Read applicable repository instructions. Use `rg` to locate the runner, completion/finalization code, logging setup, Axiom dataset configuration, and candidate messages such as `success`, `completed`, `finalized`, `processing time`, or `SCRIPT_OUTCOME`.
- Trace callers and control flow around each candidate. Establish whether it fires after successful finalization, once per counted unit, and whether retries, resume paths, skipped work, or failures can also emit it. A timer or an attempted UI action is not sufficient evidence of success. Check shared package implementations when they emit the event.
- Record the selected source file and line, exact stable text prefix or structured event/status predicate, counted unit, and any duplicate-emission risks. Prefer an explicit structured success outcome when its semantics and live availability are verified. Otherwise use a stable log prefix, excluding variable drug names, IDs, and durations.
- Inspect logging configuration for the dataset and service/environment routing without printing credentials. Confirm production versus development scope; use observed service/environment fields if a dataset is shared. Never silently include test traffic.
- Analyze the repository read-only. Do not add a logging event merely to make the dashboard possible. If no reliable completion event exists, explain the gap and propose the smallest instrumentation change separately. Respect walkthrough document protections.

## Verify against Axiom

Discover the available Axiom tools; typical names are `listDatasets`, `getDatasetFields`, `queryDataset`, `listDashboards`, `exportDashboard`, `createDashboard`, and `updateDashboard`. Follow their current schemas. If the connector is unavailable, finish repository analysis and provide a clearly unvalidated query/definition rather than claiming a live dashboard exists.

1. Confirm the dataset with `listDatasets`, then inspect its schema before substantive queries. Sample map keys when necessary; do not assume the text column is `body` or that structured fields are top-level. Follow tool guidance for cardinality probes and narrow API time bounds.
2. Query a small recent active window, widening only as needed. Verify that the candidate exists in deployed logs and inspect a few surrounding events to establish completion semantics. A local source line may be newer or older than deployed code. Zero matches alone do not prove a broken marker or broken ingestion: check recent ingestion, activity, and deployment differences.
3. Use one shared dataset/scope/success predicate for every chart. Prefer a field equality or stable prefix over a broad `contains "success"` test. Do not OR historical and current markers unless their overlap is understood and duplicates are prevented.
4. Use `count()` only when one matching event represents one completed unit. If duplicates exist, derive a stable completion identity from actual fields and use exact grouping/deduplication before time bucketing. Choose first/last completion time based on the event semantics. Do not deduplicate unrelated prescriptions by drug name, or repeated legitimate executions by a reused job ID. Do not label approximate `dcount()` results as exact.
5. Run all final queries over the same bounded interval and compare totals across hourly/daily buckets. State the bucketing timezone (UTC unless explicitly converted and verified). If evidence is insufficient, report that limitation rather than presenting unverified counts as successful throughput.

For a verified text marker, this is the query shape; substitute observed dataset, field, scope, prefix, and unit label:

```apl
['BOT_DATASET']
| where ['body'] startswith "VERIFIED_SUCCESS_PREFIX"
| summarize ['Units Processed'] = count() by bin(['_time'], 1h)
```

Use `1d` for daily tables. Name the bucket `Hour = bin(['_time'], 1h)` or `Day = bin(['_time'], 1d)` and sort that column descending. Apply the same deduplication stage, when needed, to all queries.

## Average processing time

Build one observation per successfully completed unit:

1. Reuse the verified production scope, completion identity, and deduplication rules from throughput. Prefer a duration attached to the completion event. Otherwise correlate timing events by the same unit identity and attempt. Prevent many-to-many joins and never attach a failed attempt's timer to a later successful attempt.
2. Inspect the timer's actual start and stop in code and logs. Prefer a recorded elapsed duration; otherwise subtract matched start/end timestamps on a consistent clock. Name the measured interval explicitly. Do not substitute LLM latency, idle gaps, or a whole-document duration divided by its number of prescriptions.
3. Keep every verified completion in the completed count. Include only finite positive durations in the average, report the count with valid timing, and report missing or invalid durations separately. Do not replace missing values with zero or silently remove slow valid observations.
4. Calculate the arithmetic mean over individual durations and display minutes explicitly. Never average daily averages to produce the cumulative average. Bucket daily timing by the verified completion timestamp, not the start event.

Use this cumulative aggregation shape after the validated one-row-per-completion stage:

```apl
| extend valid_duration=isnotnull(duration_s) and isfinite(duration_s) and duration_s > 0
| summarize ['Completed Rxs']=count(),
            ['Completed Rxs with timing']=countif(valid_duration),
            ['Missing or invalid duration']=countif(not(valid_duration)),
            ['Cumulative average processing time (minutes)']=avgif(duration_s, valid_duration) / 60.0
```

Use the same fields grouped by `Day=bin(completed_at, 1d)` for the daily timing table, and sort newest first. Do not add flow-specific tables, category splits, or timing trends unless the user explicitly requests them. If logs lack a reliable timer, retain the timing tables with the coverage fields and a null average, then explain the instrumentation gap; never invent timing data.

## Build or extend the dashboard

Check `listDashboards` first to avoid duplicate dashboards. For an existing matching dashboard, inspect it and preserve unrelated charts/settings; update its throughput queries when that fulfills the request. Do not overwrite a different bot's dashboard.

When available, export UID `chabot-pharmacy-rx-throughput` as the structural template. Use the nested `dashboard` document, not the export envelope. Copy only its four-table structure and layout. Replace all bot-specific names, descriptions, datasets, chart IDs, filters, query text, and units. Never reuse Chabot's success markers, migration logic, or legacy/standard merge without independently validating that exact behavior for the target bot. Do not copy exported IDs, versions, timestamps, or creator metadata into a new dashboard.

For a newly created single-bot dashboard, create exactly these four `Table` charts and no others unless the user explicitly asks:

1. **Exact Completed Rx Count by Hour (UTC):** `Hour`, `Rxs Processed`; newest first.
2. **Exact Completed Rx Count by Day (UTC):** `Day`, `Rxs Processed`; newest first.
3. **Cumulative Average Processing Time:** one row containing `Completed Rxs`, `Completed Rxs with timing`, `Missing or invalid duration`, and `Cumulative average processing time (minutes)`.
4. **Daily Average Processing Time (UTC):** `Day` plus the same count, timing-coverage, and average fields; newest first.

Use the target unit instead of `Rx` when the bot processes another unit. Each table needs a unique `id`, the verified `datasetId`, `showChart: true`, and `query.apl`. Use a 12-column grid matching Chabot: hourly `(0,0,6,12)`, daily counts `(6,0,6,12)`, cumulative timing `(0,12,12,6)`, and daily timing `(0,18,12,12)`. Each layout entry's `i` must exactly match its chart ID.

Defaults are `schemaVersion: 2`, `refreshTime: 60`, `timeWindowStart: "qr-now-30d"`, `timeWindowEnd: "qr-now"`, and `datasets: [verified_dataset]`. For a new shared team dashboard, use `owner: "X-AXIOM-EVERYONE"` unless the user requests another scope. Name it `<Bot Name> — Successful <Unit> Throughput`; its description must identify the success event, production scope, deduplication rules, timer boundary, timezone, and any historical-coverage limitation.

When updating an existing dashboard, preserve unrelated or explicitly user-added panels unless the user asks to standardize it to this four-table layout. Do not generate statistics, time-series charts, hourly timing tables, or flow-split panels by default.

A request to create the dashboard authorizes creation; do not add a redundant confirmation step. A request for a draft or a skill alone does not authorize creating a live dashboard. Use `createDashboard` with a unique bot-specific UID, or `updateDashboard` for an authorized update. For updates, prefer the current exported version and optimistic concurrency. Re-read after a conflict rather than overwriting concurrent edits. After an uncertain create response, check for the UID before retrying to avoid duplicate creation.

## Verify and hand off

Export/get the saved dashboard and confirm it has the four required tables, every query uses the same verified dataset/scope/completion population, every chart/layout ID matches, and time settings are correct. Reconcile hourly and daily totals over the same bounded interval. Verify cumulative and daily averages, valid-timing counts, missing-duration counts, exclusions, and timer boundaries. Report its verified URL when returned, or its name and UID; do not invent a link. Include the chosen source file/line and log predicate, the counted unit, a validation count with its exact time window, and any unresolved limitation.

Keep exported logs or QA artifacts outside the repository. For pharmacy QA, follow the applicable `/tmp/codex-qa-artifacts/<pharmacy-slug>/<case-or-document-id>/` convention; do not put patient data into the reusable skill or dashboard description.
