---
name: Tag creative assets with Vidmob Creative Aperture
description: Submit one or more creatives to Vidmob's Creative Tags (Aperture) API for AI visual annotation and retrieve the structured element tags once the job completes.
api: openapi/vidmob-creative-aperture-openapi.json
operations:
  - create-creative-aperture-job
  - get-creative-aperture-job-status
generated: '2026-08-05'
method: generated
source: https://vidmob-api-docs.readme.io/docs/creative-tags-api-guide
---

# Tag creative assets with Creative Aperture

Base URL `https://public-api.vidmob.com`. Every request needs `Authorization: Bearer <api-key>`.
Submitting requires `aperture:read_write`; reading job status requires `aperture:read`.

## 1. Create the job

Call `create-creative-aperture-job` (`POST /v1/media/aperture`) with a `creatives[]` array — each entry carries
your own `id` and a direct `url` — plus optional `clientTags`, a free-form object that rides through the job so
you can join results back to your own records.

`creatives` **must contain at least 1 element**; an empty array returns HTTP 400 with
`identifier: vidmob.api-bff.badrequestexception`.

The response returns a `jobId` (UUID) and an initial `status` of `QUEUED`.

## 2. Poll the job

Call `get-creative-aperture-job-status` (`GET /v1/media/aperture/{jobId}`). The polling interval scales with job
size — larger batches take longer. Unlike scoring, **results come back with the status response**; there is no
separate fetch operation.

When `status` is `COMPLETED` the response carries `downloadUrlJSON` (and related download links) for the tag JSON.

## 3. Download promptly

Signed download URLs are **short-lived**. Treat them as ephemeral: fetch the JSON as soon as the job completes
rather than persisting the URL. Persist the tag payload, not the link.

## What comes back

Structured tags describing the content of each asset — on-screen text, logos and brands, objects, scenes, audio
cues, people and emotions, plus higher-level creative decisions such as storytelling approach, messaging and
benefit framing, production style and calls-to-action — each with where and when it appears in the asset.

## Relationship to scoring

Tagging is upstream of scoring. If `get-media-score` returns `NO_DATA_MEDIA_NOT_TAGGED` for a guideline, run this
skill on the asset and then trigger a rescore. `NO_DATA_MEDIA_TAGGING_FAILURE` means re-submit the tagging job.

## Errors

`401` uses the flat gateway envelope; `400` and `404` use
`{"status":"ERROR","traceId":...,"error":{"identifier","type","system","message"}}`. A `404` on the status call
means the `jobId` is wrong. See `errors/vidmob-problem-types.yml`.
