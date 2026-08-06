---
name: Browse Vidmob scorecards and scoring guidelines
description: Enumerate a workspace's scorecards, list the media inside a scorecard, and retrieve the criteria (guidelines) an organization scores against, so an agent knows what is being measured before it interprets a score.
api: openapi/vidmob-creative-scoring-openapi.json
operations:
  - get-organization-copy
  - get-workspace-scorecards
  - get_v1scoringworkspace{workspaceId}scorecards-1
  - get-criteria-metadata
generated: '2026-08-05'
method: generated
source: https://vidmob-api-docs.readme.io/docs/api-reference
---

# Browse scorecards and guidelines

Base URL `https://public-api.vidmob.com`, `Authorization: Bearer <api-key>`, scope `scoring:read`.

## 1. Pick the workspace

`get-organization-copy` (`GET /v1/workspaces`) lists reachable workspaces and their integer `workspaceId`.

## 2. List scorecards

`get-workspace-scorecards` (`GET /v1/scoring/workspace/{workspaceId}/scorecards`) returns the workspace's
scorecards. It supports a wide filter set: `scoreDetail`, `sortOrder`, `sortBy`, `types`, `channels`,
`startDate`, `endDate`, `markets`, `creators`, `statuses`, `searchText`, `brands`. Pagination caps at
**perPage = 20**.

Scorecard types matter: date filters apply to **inflight** scorecards; they are disregarded for **preflight**
scorecards.

## 3. List the media in a scorecard

`get_v1scoringworkspace{workspaceId}scorecards-1` (`GET /v1/scoring/scorecard/{scorecardId}/media-metadata`)
returns metadata objects for the media in a scorecard, filtered by `startDate` / `endDate` and paged with
`perPage` / **`offSet`**.

Note the casing: this operation uses `offSet` while `get-criteria-metadata` uses `offset`. Match the exact
spelling per operation — do not normalise.

## 4. Retrieve the guidelines

`get-criteria-metadata` (`POST /v1/scoring/criteria/metadata`) — a POST because the filter set travels in the
body (e.g. `{"workspaces":[45914]}`). It returns criteria across all workspaces in the organization **plus**
organization-level criteria, with groups, paged via `perPage` / `offset`. Each criterion carries the channel it
applies to and its weight — which is what makes a weighted score interpretable.

## 5. Poll for changes

`get_v1mediaupdated-scores` (`GET /v1/media/updated-scores`) lists media whose scores changed after a given
date. Use it instead of re-walking scorecards. Its four query parameters are published as unnamed placeholders
(`param`, `param1`, `param2`, `param3`) in the spec — consult the live docs before relying on them.

## Channels

Channel identifiers are case-insensitive on input and have aliases (`FACEBOOK`/`INSTAGRAM` → `META`,
`TWITTER` → `X`, `GOOGLE` → `ADWORDS`, `YOUTUBE` → `DV360`, `AMAZON` → `AMAZONADVERTISING`). `ALL_PLATFORMS`
identifies criteria recommended for the Vidmob channel — it is **not** an aggregate across platforms. See
`conventions/vidmob-conventions.yml`.
