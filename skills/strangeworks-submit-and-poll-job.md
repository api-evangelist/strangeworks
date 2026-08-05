---
name: Submit a compute job to a Strangeworks resource and poll it to completion
description: >-
  Authenticate with a workspace API key, find an ACTIVE resource, submit work to it
  through the resource REST proxy, then poll the job with the platform GraphQL API until
  it reaches a terminal state and read its result files.
api: graphql/strangeworks-platform.graphql
operations: [Query.workspace, Workspace.resources, Workspace.defaultResource, Workspace.job, Workspace.jobs]
generated: '2026-08-05'
method: generated
source: graphql/strangeworks-platform.graphql + graphql/strangeworks-sdk.graphql
---

# Submit and poll a Strangeworks job

Strangeworks is a broker in front of many compute providers. You never call a QPU or a
solver directly — you submit to a **Resource** (an activated instance of a **Product**
inside your **Workspace**) and the platform creates a **Job** you then poll.

## 1. Authenticate

Exchange your workspace API key (Strangeworks Portal home page) for a JWT.

```
POST https://api.strangeworks.com/users/token
Content-Type: application/json

{"key": "<workspace api key>"}
```

Send `Authorization: Bearer <jwt>` on every subsequent call. The token is short-lived —
re-exchange on expiry. An empty key returns `400 {"message":"key cannot be empty"}`.

Keys are per-workspace. If you belong to several workspaces you hold several keys.

## 2. Find a resource

```graphql
query { workspace { slug  resources(pagination: {first: 25}) {
  edges { node { slug status product { slug name type } } }
  pageInfo { hasNextPage endCursor } } } }
```

Only submit to a resource whose `status` is `ACTIVE`. `PENDING_ACTIVATION`,
`PENDING_DEACTIVATION` and `DEACTIVE` will not accept work. `Workspace.defaultResource(productSlug:)`
returns the workspace default for a product if one is set.

If no resource exists for the product you want, activate one with `resourceCreate`.

## 3. Submit the work

Submission goes through the resource REST proxy, not GraphQL:

```
POST https://api.strangeworks.com/products/{product_slug}/resource/{resource_slug}/{path}
Authorization: Bearer <jwt>
```

The `{path}` segment and the request body are **defined by the product**, not by
Strangeworks — read that product's page under https://docs.strangeworks.com/ (D-Wave,
Gurobi, Toshiba, IBM, Braket, Azure Quantum, …). Do not guess a path.

For bulk submission use the two-phase batch pair instead: `batchJobInitiateCreate` then
`batchJobFinalizeCreate`. This pair exists because **there is no idempotency key anywhere
in this API** — see `conventions/strangeworks-conventions.yml`. Never blind-retry a
submission; look the job up first.

## 4. Poll to completion

```graphql
query($jobSlug: String!) { workspace { job(jobSlug: $jobSlug) {
  slug status isTerminalState remoteStatus
  dateJobStarted dateJobCompleted
  files { file { slug url } } } } }
```

Stop when `isTerminalState` is true — that is the schema's own finished flag, and it is
more reliable than matching statuses yourself.

`JobStatus` is `CREATED, QUEUED, RUNNING, COMPLETED, FAILED, CANCELLING, CANCELLED, ERROR`.
Success is `COMPLETED`; `FAILED`, `CANCELLED` and `ERROR` are terminal failures.
`remoteStatus` is an opaque string from the downstream provider — surface it in errors,
do not branch on it, it is not enumerated.

Prefer a webhook over polling where you can: register `JOB_COMPLETE` with
`eventSubscriptionCreate` (see `asyncapi/strangeworks-webhooks.yml`). Note there is no
documented signature on the delivery, so verify by re-reading the job.

## 5. Read results

Job results arrive as files. `Job.files` returns `JobFile` entries wrapping a `File`;
fetch the file's signed URL directly. `Job.data` is untyped `JSON` whose shape is declared
out-of-band by `dataSchemaSlug`. Note `Job.jobData` and `Job.jobDataSchema` are
`@deprecated` — use `data` and `dataSchemaSlug`.

A long job may spawn children: check `Job.childJobs` before deciding the work is done.

## Errors

GraphQL failures come back as **HTTP 200** with a top-level `errors[]` array — always
check `errors` even on a 200. The token endpoints and the REST proxy use real HTTP status
codes with `{"message": ..., "description": ...}`. There is no RFC 9457 problem+json.
See `errors/strangeworks-problem-types.yml`.

## Cost

Every paid action creates a `BillingTransaction` against the workspace billing account.
Workspaces and members carry spending limits (`SpendingLimitPeriod`), and some work needs
billing approval (`requestBillingApproval`, `BillingApprovalStatus`). Check
`Workspace.billingTransactions` before running a large batch.
