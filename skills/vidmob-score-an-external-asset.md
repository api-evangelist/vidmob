---
name: Score an external creative asset with Vidmob
description: Register a creative asset with Vidmob, poll until scoring completes, and retrieve the weighted guideline score and per-guideline breakdown for one or more advertising channels.
api: openapi/vidmob-creative-scoring-openapi.json
operations:
  - get-organization
  - get-organization-copy
  - upload-media-for-scoring
  - get-media-scoring-status
  - get-media-score
generated: '2026-08-05'
method: generated
source: https://vidmob-api-docs.readme.io/docs/media-scoring-api-guide
---

# Score an external creative asset

Base URL `https://public-api.vidmob.com`. Every request needs `Authorization: Bearer <api-key>`.
Submitting requires the `scoring:read_write` scope; reads require `scoring:read`.

## 1. Find the workspace

Call `get-organization` (`GET /v1/organization`) to confirm the key works, then `get-organization-copy`
(`GET /v1/workspaces`) to list the workspaces this key can reach. Keep the integer `workspaceId` — most
scoring operations are scoped by it. Both operations need a valid key but no specific scope.

## 2. Submit the asset

Call `upload-media-for-scoring` (`POST /v1/media`) with at minimum a caller-side `id` and a `url`.

- `url` must be a **direct, publicly accessible download URL**. Presigned S3 URLs and CDN links work; auth-gated
  or redirecting URLs cause silent failures.
- `version` joins `id` and `source` to form the deduplication key.
- `workspaceId` is optional but strongly recommended — it applies workspace-specific guidelines and makes the
  asset visible in the platform UI.
- `channels` is optional; it defaults to the channels configured on the workspace scorecard. Values are
  case-insensitive on input (`META`, `TIKTOK`, `DV360`, `ADWORDS`, …) — see `conventions/vidmob-conventions.yml`.

The response envelope is `{"status":"OK","result":{"uniqueId":"<uuid>"}}`. **Store the `uniqueId`** — it
permanently identifies the asset.

**Retrying is safe.** Re-submitting the same `id` + `version` + `source` returns HTTP **409** with the existing
`uniqueId` rather than re-ingesting. A 409 here is a successful deduplicated submit, not a failure: read the
`uniqueId` out of it and continue. Do not implement retry backoff around it.

## 3. Poll for completion

Call `get-media-scoring-status` (`GET /v1/media/{mediaId}/status`) every **10–15 minutes**. Polling faster does
not speed processing and may trigger per-organization throttling. Typical completion is 30–60 minutes.

- `PROCESSING` — keep polling.
- `COMPLETE` — go to step 4.
- `ERROR` — non-terminal; retry after a delay.
- `UNSUPPORTED` / `FAILED` — terminal. Stop; do not retry.

You may look the asset up by your own `id` instead of the `uniqueId` by adding `?source=<source>&version=<version>`.

## 4. Retrieve the scores

Call `get-media-score` (`GET /v1/scoring/media/{mediaId}/scores`).

- `format=summary` (default) — per-channel `score` (weighted, 0.0–1.0), `adherencePercent` (pass rate, 0.0–1.0)
  and pass/fail counts.
- `format=detail` — adds the per-guideline breakdown (name, rule, result, weight).
- `format=summary,detail` — both.
- Narrow with `channel`, and disambiguate a caller-side id with `source` / `version`.

Guidelines that cannot be evaluated do not count against the asset.

## 5. Interpret per-guideline results

`PASS`, `FAIL`, `NOT_APPLICABLE`, legacy `NO_DATA`, or a specific `NO_DATA_<CODE>`. Match the specific codes —
`NO_DATA` is being phased out. Several codes are actionable rather than terminal: `NO_DATA_MEDIA_NOT_TAGGED`
means run the Aperture tagging skill first; `NO_DATA_MISSING_REQUIRED_TAGS`, `NO_DATA_ERROR_RETRIEVING_TAGS`
and `NO_DATA_SCORING_BACKLOG` mean rescore later. Full table: `errors/vidmob-no-data-codes.yml`.

## Errors

Two envelopes. `401` returns a flat `{"statusCode":401,"message":"Request is Unauthorized","error":"Unauthorized"}`.
Everything else returns `{"status":"ERROR","traceId":...,"error":{"identifier","type","system","message"}}`. Branch
on the presence of `error.identifier` versus `statusCode`. Quote the `traceId` to support. See
`errors/vidmob-problem-types.yml`. No 429 or 5xx response is declared, so treat unexpected statuses defensively.
