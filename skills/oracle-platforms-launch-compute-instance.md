---
name: launch-compute-instance
description: >-
  Launch an OCI compute instance safely — pick a shape and image, place it on a subnet,
  make the create idempotent, and wait for the work request instead of trusting the 202.
api: Oracle Cloud Infrastructure (OCI) REST API
spec: openapi/oracle-platforms-core-openapi.yml
operations:
  - ListShapes
  - ListImages
  - ListVcns
  - ListSubnets
  - LaunchInstance
  - GetInstance
  - InstanceAction
  - TerminateInstance
generated: '2026-08-27'
method: generated
source: Grounded in verified operationIds from openapi/oracle-platforms-core-openapi.yml
---

# Launch an OCI compute instance

Base host: `https://iaas.{region}.oraclecloud.com/20160918`. Every request must be signed —
see `authentication/oracle-platforms-authentication.yml`. There is no bearer token and no
sandbox: this provisions real, billable infrastructure.

## 1. Choose where it goes

Everything is scoped by `compartmentId`. Get the target compartment OCID first; it is a
required query parameter on every list call below.

- `ListVcns` — `GET /vcns?compartmentId={ocid}` — find the network.
- `ListSubnets` — `GET /subnets?compartmentId={ocid}&vcnId={ocid}` — pick the subnet. Note
  its availability domain; the instance must launch in the same one.

## 2. Choose what it runs

- `ListShapes` — `GET /shapes?compartmentId={ocid}` — the machine sizes available to you.
  Availability is per-compartment and per-availability-domain, so do not cache this globally.
- `ListImages` — `GET /images?compartmentId={ocid}` — the OS images. Filter by
  `operatingSystem` and take the newest `id`.

Both lists paginate. Read `opc-next-page` from the response headers and pass it back as
`page`. The listing is exhausted when that header is **absent** — there is no total count.

## 3. Launch it — idempotently

`LaunchInstance` — `POST /instances/`

Set the `opc-retry-token` header to a value you generate and persist **before** you send the
request. This is the whole point: if the call times out you do not know whether it
succeeded, and replaying it without the token launches a second billable instance. With the
token, the replay returns the original result. The token lives 24 hours (max 64 chars).

If the retry returns `409 InvalidatedRetryToken`, do **not** mint a fresh token and try
again — that is how you get a duplicate. Call `ListInstances` filtered on your
`displayName` and reconcile.

## 4. Wait properly

The response carries `opc-work-request-id`. A 2xx here means *accepted*, not *running*.

- Poll `GetInstance` — `GET /instances/{instanceId}` — until `lifecycleState` is `RUNNING`.
- `PROVISIONING` and `STARTING` are normal intermediate states; `FAILED` is terminal.
- Correlate every request with the `opc-request-id` you sent; quote it to Oracle support if
  something goes wrong.

## 5. Changing and stopping it

- `InstanceAction` — `POST /instances/{instanceId}` with an `action` of `STOP`, `START`,
  `RESET`, `SOFTRESET` or `SOFTSTOP`. Reversible.
- `TerminateInstance` — `DELETE /instances/{instanceId}` — **irreversible**. There is no
  un-terminate operation. Whether the boot volume survives is decided by a parameter *on
  this call*, not afterwards, so decide before you send it. Use `if-match` with the
  instance's current `etag` so you cannot terminate a resource that changed underneath you;
  a mismatch returns `412 NoEtagMatch`.

## Errors

`errors/oracle-platforms-problem-types.yml` has the full catalogue. The two that matter
most here: `429 TooManyRequests` (back off — Oracle publishes no `Retry-After`, so choose
your own delay) and `404 NotAuthorizedOrNotFound`, which deliberately conflates "does not
exist" with "you may not see it". Never treat that 404 as proof the instance is gone.
