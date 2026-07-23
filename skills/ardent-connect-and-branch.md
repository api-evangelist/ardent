---
name: Connect a Postgres source and create a branch
description: >-
  Preflight and connect a Postgres database to Ardent, let it discover the schema, choose what to
  replicate, then create an isolated, disposable branch and get its connection URL.
api: openapi/ardent-openapi-original.json
operations:
- preflight_connector_endpoint_v1_connectors_preflight_post
- create_connector_endpoint_v1_connectors_post
- discover_connector_endpoint_v1_connectors__connector_id__discover_post
- set_selection_endpoint_v1_connectors__connector_id__selection_post
- get_operation_endpoint_v1_operations__operation_id__get
- create_service_branch_v1_branch_create_post
- get_branches_v1_branches__connector_id__get
---

# Connect a Postgres source and create a branch

Authenticate every request with `Authorization: Bearer <key>` where the key is an Ardent API key
(`sk-ard_live_…` or `sk-ard_test_…`). Base URL is `https://api.tryardent.com`.

## Steps

1. **Preflight the source** — `POST /v1/connectors/preflight`
   (`preflight_connector_endpoint_v1_connectors_preflight_post`). Send the Postgres connection
   string; this creates nothing and returns a `PreflightReport`. Do not proceed if it reports blockers.
2. **Create the connector** — `POST /v1/connectors`
   (`create_connector_endpoint_v1_connectors_post`). Capture the returned `connector_id`.
3. **Discover the schema** — `POST /v1/connectors/{connector_id}/discover`
   (`discover_connector_endpoint_v1_connectors__connector_id__discover_post`). This is async and
   returns an operation handle.
4. **Poll to completion** — `GET /v1/operations/{operation_id}`
   (`get_operation_endpoint_v1_operations__operation_id__get`) until `status` is `succeeded`
   (stop and report on `failed`, reading `stage` + `error`).
5. **Select tables/schemas** — `POST /v1/connectors/{connector_id}/selection`
   (`set_selection_endpoint_v1_connectors__connector_id__selection_post`) with the discovered paths
   to replicate.
6. **Create a branch** — `POST /v1/branch/create`
   (`create_service_branch_v1_branch_create_post`). Async; poll the returned operation handle as in
   step 4 (branch create can take up to ~65 min for large databases).
7. **Get the connection URL** — `GET /v1/branches/{connector_id}`
   (`get_branches_v1_branches__connector_id__get`) to list branches with their connection URLs.

## Rules

- Errors carry a `{"detail": "..."}` envelope (no RFC 9457). Retry only 429 and 5xx with backoff;
  never retry 4xx (400/401/403/404/409/422).
- No idempotency key exists — do not blindly re-POST a create; instead poll the operation handle
  returned by the first call to learn its outcome.
- 403 means the key's role lacks permission (`role_org_viewer` is read-only); use at least
  `role_org_member` to create connectors and branches.
