---
name: contextdev-monitor-a-website
description: >-
  Use this skill to watch a webpage, sitemap, or extracted dataset for changes
  with the Context.dev Monitors API and react to signed webhooks. Grounds every
  step in real operationIds from the Context.dev OpenAPI.
api: openapi/contextdev-openapi.yml
operations:
- createMonitor
- getMonitor
- listMonitorRuns
- listMonitorChanges
- getChange
- runMonitorNow
- updateMonitor
- deleteMonitor
method: generated
source: derived from openapi/contextdev-openapi.yml + docs.context.dev/api-reference/monitors
---

# Monitor a website for changes with Context.dev

Base URL `https://api.context.dev/v1`. Authenticate with a bearer token read from
`CONTEXT_DEV_API_KEY` (keys start with `ctxt_secret_`):

```
Authorization: Bearer $CONTEXT_DEV_API_KEY
```

## Steps

1. **Create the monitor** — `POST /monitors` (`createMonitor`). The request body is
   a union of supported target + change-detection combinations (page, sitemap, or
   extracted dataset). Set the `webhook` URL to receive events. The monitor runs
   immediately to establish its baseline.
2. **Confirm the baseline** — `GET /monitors/{monitor_id}` (`getMonitor`) to read
   the monitor's status, schedule, and current baseline.
3. **Receive events** — your webhook endpoint gets a signed `change.detected`
   payload (diffs, added/removed URLs, semantic evidence, confidence, importance)
   and, if subscribed, a `run.completed` payload after every run. Verify the
   signature, then return any `2xx` to acknowledge.
4. **Inspect history** — `GET /monitors/{monitor_id}/runs` (`listMonitorRuns`) and
   `GET /monitors/{monitor_id}/changes` (`listMonitorChanges`). Both paginate with
   `cursor` / `next_cursor` / `has_more`.
5. **Fetch a full change** — `GET /monitors/changes/{change_id}` (`getChange`).
6. **Force a run** — `POST /monitors/{monitor_id}/run` (`runMonitorNow`) queues an
   immediate asynchronous run outside the schedule.
7. **Update or remove** — `PATCH /monitors/{monitor_id}` (`updateMonitor`;
   changing `target`/`change_detection` rebuilds the baseline) and
   `DELETE /monitors/{monitor_id}` (`deleteMonitor`).

## Rules

- **Rate limits**: pace with `X-RateLimit-Remaining`; on `429` honor `Retry-After`
  (1-60s). See `conventions/contextdev-conventions.yml`.
- **Errors**: flat `{message, status, error_code}` envelope; see
  `errors/contextdev-problem-types.yml`. `429` uses `code` instead of `error_code`.
- **No idempotency key** is offered; monitor CRUD is not retry-idempotent, so
  read back with `getMonitor` before recreating.
- **Tags**: Monitors keep their own `tags` semantics (grouping), distinct from the
  account-wide usage `tags` on other endpoints.
