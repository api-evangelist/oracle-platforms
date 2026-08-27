---
name: deploy-data-science-model
description: >-
  Take an OCI Data Science model from project to live model deployment, run and cancel
  training jobs, and read the work-request trail when something fails.
api: Oracle Cloud Infrastructure Data Science API
spec: openapi/oracle-platforms-data-science-openapi.yml
operations:
  - ListProjects
  - CreateProject
  - CreateModel
  - ListModels
  - GetModel
  - CreateModelDeployment
  - GetModelDeployment
  - ListModelDeployments
  - ActivateModelDeployment
  - DeactivateModelDeployment
  - CreateJob
  - CreateJobRun
  - GetJobRun
  - CancelJobRun
  - CancelWorkRequest
generated: '2026-08-27'
method: generated
source: Grounded in verified operationIds from openapi/oracle-platforms-data-science-openapi.yml
---

# Deploy a model on OCI Data Science

Base host: `https://datascience.{region}.oci.oraclecloud.com/20190101`. Note the `.oci.`
segment — this host differs from the Core and Database hosts, and Oracle's own spec index is
the authority for it.

## 1. Project first

Everything hangs off a project. `ListProjects` — `GET /projects?compartmentId={ocid}` — or
`CreateProject` — `POST /projects`. Every model, job and pipeline carries `projectId`
alongside `compartmentId`; the entity graph in `data-model/oracle-platforms-data-model.yml`
shows why `projectId` is the second-most-referenced id in this contract.

## 2. Register the model

`CreateModel` — `POST /models` — with `projectId` and `compartmentId`. Send
`opc-retry-token`.

`ListModels` — `GET /models?compartmentId={ocid}&projectId={ocid}` — and `GetModel` —
`GET /models/{modelId}` — to read back.

## 3. Deploy it

`CreateModelDeployment` — `POST /modelDeployments` — is where the model becomes an
inference endpoint and starts costing money. Asynchronous: take `opc-work-request-id` from
the response and poll `GetModelDeployment` —
`GET /modelDeployments/{modelDeploymentId}` — until `lifecycleState` is `ACTIVE`.

Lifecycle control after that:

- `DeactivateModelDeployment` — `POST /modelDeployments/{id}/actions/deactivate` — stops
  serving and stops the bill. Reversible.
- `ActivateModelDeployment` — `POST /modelDeployments/{id}/actions/activate` — brings it
  back.

Prefer deactivate over delete. Deactivation is the reversible cost control here, the same
role `StopAutonomousDatabase` plays for the database API.

## 4. Training jobs

- `CreateJob` — `POST /jobs` — defines the job.
- `CreateJobRun` — `POST /jobRuns` — starts an execution.
- `GetJobRun` — `GET /jobRuns/{jobRunId}` — poll `lifecycleState`.
- `CancelJobRun` — `POST /jobRuns/{jobRunId}/actions/cancelJobRun` — stops an in-flight run.

`CancelJobRun` is the cheapest reversal available in this API: it works while the run is in
progress and needs no window negotiation. Reach for it the moment a run is wrong, rather
than letting it finish and cleaning up after.

## 5. When something fails

`CancelWorkRequest` — `DELETE /workRequests/{workRequestId}` — aborts an in-flight
asynchronous operation. The work-request surface is also where the failure detail lives:
the work request carries error and log-entry children, and the `opc-request-id` you sent is
the correlation key Oracle support will ask for.

## Ground rules that apply to every call here

Idempotency is `opc-retry-token` (24h). Concurrency is `etag` + `if-match`. Pagination is
`limit`/`page` with the `opc-next-page` cursor, terminating on the header's absence. A 2xx
on any create is acceptance, never completion. Full detail in
`conventions/oracle-platforms-conventions.yml`.
