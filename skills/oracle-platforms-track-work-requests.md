---
name: track-work-requests
description: >-
  The completion model for the whole Oracle platform — how to tell whether an asynchronous
  OCI operation actually finished, why it failed, and how to cancel it.
api: Oracle Integration Cloud API
spec: openapi/oracle-platforms-integration-openapi.yml
operations:
  - ListWorkRequests
  - GetWorkRequest
  - ListWorkRequestErrors
  - ListWorkRequestLogs
  - CancelWorkRequest
generated: '2026-08-27'
method: generated
source: >-
  Grounded in verified operationIds from openapi/oracle-platforms-integration-openapi.yml
  (ListWorkRequests, GetWorkRequest, ListWorkRequestErrors, ListWorkRequestLogs) and
  openapi/oracle-platforms-data-science-openapi.yml (CancelWorkRequest)
---

# Track OCI work requests

This is the most important skill in this profile and the least obvious one. Every one of
the six Oracle APIs here returns `opc-work-request-id` on mutations that cannot complete
synchronously — 388 operation definitions across the harvested contracts carry it. **A 2xx
response on those operations means "accepted", not "done."** Treating the 202 as success is
the single most common way an agent gets Oracle wrong: it moves on to the next step against
a resource that is still `PROVISIONING`, and the next call fails with a state error that
looks unrelated.

## The loop

1. Make the mutating call. Keep the `opc-work-request-id` response header.
2. `GetWorkRequest` — `GET /workRequests/{workRequestId}` — poll it.
3. Read `status` and `percentComplete`. `ACCEPTED` and `IN_PROGRESS` mean keep waiting.
   `SUCCEEDED`, `FAILED` and `CANCELED` are terminal.
4. Only after `SUCCEEDED` should you read the resource itself and expect it to be usable.

Back off between polls. There is no `Retry-After` on this API and a tight poll loop is a
good way to earn `429 TooManyRequests`, which also carries no retry hint.

## When it fails

- `ListWorkRequestErrors` — `GET /workRequests/{workRequestId}/errors` — the machine-readable
  reason. This is where the real failure lives; the original call's response will not tell
  you.
- `ListWorkRequestLogs` — `GET /workRequests/{workRequestId}/logs` — the step-by-step trail.

Both paginate with `limit`/`page` and terminate on the absence of `opc-next-page`.

## Finding requests you lost

`ListWorkRequests` — `GET /workRequests?compartmentId={ocid}` — scoped by compartment, and
filterable by the affected resource. Use this to reconcile after a timeout, especially
after an `opc-retry-token` replay you are unsure about.

## Cancelling

`CancelWorkRequest` — `DELETE /workRequests/{workRequestId}` — aborts an operation that has
not reached a terminal state. Whether the partial work is rolled back depends on the
service; do not assume the resource returns to its prior state. Verify with a read.

## Note on scope

The operationIds above are verified in the Integration and Data Science contracts in this
repo. The work-request pattern is platform-wide and Core Services, Database, Analytics and
Content Management all expose an equivalent surface, but only the operations verified in
those two documents are named here.
