---
name: Subscribe to Strangeworks job events instead of polling
description: >-
  Register an HTTPS endpoint against a Strangeworks EventType so the platform pushes job
  state changes to you, and handle the fact that deliveries are unsigned and unretried.
api: graphql/strangeworks-platform.graphql
operations: [Mutation.eventSubscriptionCreate, Mutation.eventSubscriptionUpdate, Mutation.eventSubscriptionDelete, Workspace.eventSubscriptions, Workspace.eventSubscription]
generated: '2026-08-05'
method: generated
source: graphql/strangeworks-platform.graphql
---

# Subscribe to Strangeworks job events

Jobs on real quantum hardware queue for a long time. Polling burns tokens and calls;
subscribe instead.

## 1. Register the endpoint

```graphql
mutation($input: EventSubscriptionCreateInput!) {
  eventSubscriptionCreate(input: $input) {
    eventSubscription { slug name eventType endpoint isDisabled } } }
```

`EventSubscriptionCreateInput` takes exactly four fields: `workspaceSlug`, `name`,
`eventType`, `endpoint`.

One subscription carries **one** event type. To receive more than one, create more than
one subscription.

## 2. The event types

`EventType` has exactly three values:

| Value | Meaning |
|---|---|
| `JOB_COMPLETE` | A job reached a terminal state |
| `JOB_UPDATE` | A job changed state short of terminal |
| `REQ_RESP_HEADER` | Undocumented — Strangeworks publishes no description for it |

Do not build on `REQ_RESP_HEADER`. Its semantics are not documented anywhere public.

## 3. Handle the delivery defensively

Strangeworks publishes **no payload schema** for any event type, **no signing secret**,
**no retry policy**, and **no delivery log**. Treat the webhook as a *hint*, not as data:

1. On receipt, ignore the body's contents as authoritative.
2. Re-read the job over GraphQL (`Workspace.job(jobSlug:)`) and trust `isTerminalState`.
3. Keep a slow reconciliation poll as a backstop, because a dropped delivery is not
   retried and you will not be told.
4. Since the request is unsigned, treat the endpoint as public: use an unguessable path,
   accept POST only, and never act on the body alone.

`JOB_COMPLETE` fires on terminal state, which includes `FAILED`, `CANCELLED` and `ERROR` —
not just `COMPLETED`. Check the status; do not assume completion means success.

## 4. Manage subscriptions

- List: `Workspace.eventSubscriptions`
- Read one: `Workspace.eventSubscription(slug:)`
- Pause without deleting: `eventSubscriptionUpdate` with `isDisabled: true`
- Remove: `eventSubscriptionDelete` with `workspaceSlug` + `eventSubscriptionSlug`

Reconcile on startup — list what exists before creating, or you will accumulate duplicate
subscriptions delivering the same event repeatedly. There is no idempotency key on the
create mutation.
