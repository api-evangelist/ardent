---
name: Give an AI agent an isolated ephemeral database
description: >-
  From an existing connector, spin up a disposable branch database so a coding agent can run
  migrations, backfills, or destructive tests against production-like data with zero blast radius,
  then reuse or discard it.
api: openapi/ardent-openapi-original.json
operations:
- list_connectors_endpoint_v1_connectors_get
- create_service_branch_v1_branch_create_post
- get_operation_endpoint_v1_operations__operation_id__get
- get_branches_v1_branches__connector_id__get
---

# Give an AI agent an isolated ephemeral database

Use this once a Postgres source is already connected (see the connect-and-branch skill). All calls
use `Authorization: Bearer <key>` against `https://api.tryardent.com`. Prefer a `sk-ard_test_` key
for agent experimentation.

## Steps

1. **Find the connector** — `GET /v1/connectors`
   (`list_connectors_endpoint_v1_connectors_get`) and pick the `connector_id` for the source you
   want a branch of.
2. **Create the branch** — `POST /v1/branch/create`
   (`create_service_branch_v1_branch_create_post`) referencing the connector. Returns an async
   operation handle.
3. **Poll to ready** — `GET /v1/operations/{operation_id}`
   (`get_operation_endpoint_v1_operations__operation_id__get`) until `status` is `succeeded`.
4. **Get the connection URL** — `GET /v1/branches/{connector_id}`
   (`get_branches_v1_branches__connector_id__get`) and hand the branch's connection URL to the
   agent. The branch is isolated at compute and storage; nothing it does can reach production.

## Rules

- The branch is disposable: run risky migrations/tests freely, then delete it when done (`ardent
  branch delete` via CLI, or the branch delete flow) — do not leave idle branches; they scale to
  zero but still count against project limits.
- Anonymize PII by running SQL on the new branch before handing it to an untrusted agent.
- Honor the async contract: never treat a 202 as done — poll the operation handle. Retry only 429/5xx.
