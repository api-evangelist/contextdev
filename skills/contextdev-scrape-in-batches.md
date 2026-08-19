---
name: contextdev-scrape-in-batches
description: >-
  Use this skill to scrape many pages or crawl a whole site with the Context.dev
  Batch API instead of calling the per-URL scrape endpoints. Use it whenever the
  job covers more than a handful of URLs, whenever a crawl would otherwise be a
  weighted POST /web/crawl call, or whenever a per-URL loop would burn the
  per-minute rate limit. Grounds every step in real operationIds from the
  Context.dev OpenAPI.
api: openapi/contextdev-batch-api-openapi.yml
operations:
- submitBatch
- getBatch
- getBatchResults
- listBatches
- cancelBatch
- deleteBatch
method: generated
source: >-
  derived from openapi/contextdev-batch-api-openapi.yml +
  https://docs.context.dev/guides/scrape-websites-in-batches +
  https://docs.context.dev/optimization/rate-limits
---

# Scrape at volume with the Context.dev Batch API

Base URL `https://api.context.dev/v1`. Authenticate with a bearer token read from
`CONTEXT_DEV_API_KEY` (keys start with `ctxt_secret_`):

```
Authorization: Bearer $CONTEXT_DEV_API_KEY
```

## Why batch instead of a loop

A batch is **one request** against your per-minute rate limit no matter how large
it is. `POST /batch/submit` carrying 25,000 URLs costs the same one request as a
batch carrying one URL — the pages inside are queued and scraped by Context.dev's
own workers and never draw on your budget. The same work as 5,000 individual
`POST /web/scrape/markdown` calls costs 5,000 requests and saturates a Pro plan
(300 req/min) for roughly 17 minutes.

Crawls benefit most: a synchronous `POST /web/crawl` is a **weighted** endpoint
costing 10 requests, while the same crawl submitted as a batch input costs 1.

## Steps

1. **Submit the batch** — `POST /batch/submit` (`submitBatch`). The body is a
   `BatchSubmitRequest`: a list of URLs *or* a crawl input, plus optional `tags`
   and a `webhookUrl`. Send an **`Idempotency-Key` header** (any string unique to
   this submission, max 200 chars) so a network retry returns the original batch
   instead of creating a duplicate. Costs 1 credit per successfully scraped URL.
2. **Prefer the webhook over polling** — if you passed `webhookUrl`, Context.dev
   calls you when the batch settles and you spend zero requests waiting. This is
   the difference between a job that costs 1 request and one that costs hundreds.
3. **If you must poll, poll slowly** — `GET /batch/{batch_id}` (`getBatch`)
   returns progress and, once finished, the download links. Every poll is an
   ordinary 1-request call: polling every two seconds spends 30 requests a minute
   doing nothing but asking. Every 10–30 seconds resolves a multi-minute job just
   as well.
4. **Read the results** — either download the gzipped NDJSON from the links on the
   settled batch (cheapest), or page through JSON with
   `GET /batch/{batch_id}/results` (`getBatchResults`) using `limit` (1–100) and
   `cursor` from the previous page's `next_cursor`.
5. **Find batches later** — `GET /batch/list` (`listBatches`), newest first.
   Filter with `status` (`queued`, `running`, `cancelling`, `completed`,
   `cancelled`, …), `tags` (comma-separated), or free-text `q` with
   `search_type=prefix` for as-you-type search. Paginate with `limit` + `cursor`.
6. **Stop a run** — `POST /batch/{batch_id}/cancel` (`cancelBatch`). In-progress
   pages finish and unused credits are refunded.
7. **Clean up** — `DELETE /batch/{batch_id}` (`deleteBatch`) permanently removes a
   finished batch and its stored results. Active batches must settle first.

## Limits that bite before the rate limit does

Concurrent in-flight batches are capped by plan — **1** (Free), **2** (Developer),
**5** (Pro), **20** (Scale). Submitting past the cap returns
`403 BATCH_LIMIT_EXCEEDED`. On any volume job this binds long before the
per-minute request cap, so queue submissions against the concurrency limit rather
than against `X-RateLimit-Remaining`.

## Error handling

Errors are a flat `{message, status, error_code}` JSON envelope — **not**
`application/problem+json`.

| Status | `error_code` | What to do |
| --- | --- | --- |
| 400 | `INPUT_VALIDATION_ERROR` | Malformed submission or result request. Fix and resubmit; do not retry blind. |
| 401 | — | Missing/invalid/expired key. Declared on all six operations. |
| 403 | `BATCH_LIMIT_EXCEEDED` | Concurrency cap reached. Wait for a batch to settle. |
| 404 | — | No batch with that `batch_id` on this account. |
| 409 | `IDEMPOTENCY_KEY_CONFLICT` | Same `Idempotency-Key`, different body. Reuse the original body or mint a new key. |
| 409 | `BATCH_NOT_COMPLETED` | Results or delete requested too early. Poll `getBatch` or take the webhook. |
| 409 | `BATCH_NOT_CANCELLABLE` | Batch already settled. Read its terminal status. |
| 429 | — | Rate limited. Honor `Retry-After` (1–60s). |
| 500 | — | Transient. Retry with exponential backoff — safe, because you sent an `Idempotency-Key`. |

Every response carries `X-RateLimit-Limit`, `X-RateLimit-Remaining` and
`X-RateLimit-Reset`; pace off `X-RateLimit-Remaining` rather than waiting for a
429.

## Cross-references

- `conventions/contextdev-conventions.yml` — auth, pagination, idempotency, compression
- `rate-limits/contextdev-rate-limits.yml` — weighted endpoints, exemptions, headers
- `errors/contextdev-problem-types.yml` — full error catalog
- `plans/contextdev-plans-pricing.yml` — credit allowances and concurrency by tier
