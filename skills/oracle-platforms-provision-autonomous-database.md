---
name: provision-autonomous-database
description: >-
  Create, connect to, back up and restore an Oracle Autonomous Database through the OCI
  Database API, including the one reversal window Oracle actually publishes.
api: Oracle Autonomous Database API
spec: openapi/oracle-platforms-database-openapi.yml
operations:
  - ListAutonomousDatabases
  - CreateAutonomousDatabase
  - GetAutonomousDatabase
  - UpdateAutonomousDatabase
  - GenerateAutonomousDatabaseWallet
  - StopAutonomousDatabase
  - StartAutonomousDatabase
  - CreateAutonomousDatabaseBackup
  - ListAutonomousDatabaseBackups
  - RestoreAutonomousDatabase
generated: '2026-08-27'
method: generated
source: Grounded in verified operationIds from openapi/oracle-platforms-database-openapi.yml
---

# Provision and operate an Autonomous Database

Base host: `https://database.{region}.oraclecloud.com/20160918`. Signed requests only.

## 1. Create

`CreateAutonomousDatabase` — `POST /autonomousDatabases`

Send `opc-retry-token`. A duplicate Autonomous Database is expensive in a way a duplicate
list call is not, and this is the single most important header on this operation.

The create is asynchronous. Poll `GetAutonomousDatabase` —
`GET /autonomousDatabases/{autonomousDatabaseId}` — until `lifecycleState` is `AVAILABLE`.
`PROVISIONING` is normal; `FAILED` is terminal.

## 2. Connect

`GenerateAutonomousDatabaseWallet` —
`POST /autonomousDatabases/{autonomousDatabaseId}/actions/generateWallet`

Returns the client credentials wallet. This is the only way to obtain connection
credentials through the API; there is no "get connection string" read operation that
substitutes for it. Treat the response as a secret.

## 3. Modify, with optimistic concurrency

`UpdateAutonomousDatabase` — `PUT /autonomousDatabases/{autonomousDatabaseId}`

Take the `etag` from your last `GetAutonomousDatabase` and send it as `if-match`. If
something else changed the database since you read it you get `412 NoEtagMatch` instead of
silently clobbering the other change. Scaling OCPUs and storage happens here.

## 4. Start and stop

- `StopAutonomousDatabase` — `POST /autonomousDatabases/{id}/actions/stop`
- `StartAutonomousDatabase` — `POST /autonomousDatabases/{id}/actions/start`

Stopping is the primary cost control on this API and it is fully reversible.

## 5. Back up and restore — the reversal that has a stated window

- `CreateAutonomousDatabaseBackup` — `POST /autonomousDatabaseBackups`
- `ListAutonomousDatabaseBackups` — `GET /autonomousDatabaseBackups?autonomousDatabaseId={id}`
- `RestoreAutonomousDatabase` — `POST /autonomousDatabases/{id}/actions/restore`

This is the one place in this whole profile where Oracle publishes a reversal **window**:
automatic backups are kept for "a retention period between 1 day and up to 60 days",
configurable per database
(<https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/backup-restore.html>).
Restore is bounded by that retention. Check the configured retention on the specific
database before promising a user that a point in time is recoverable — the floor is 1 day,
not 60.

Restore is itself destructive to the current state. It is asynchronous; poll
`lifecycleState` back to `AVAILABLE`.

## 6. Find things

`ListAutonomousDatabases` — `GET /autonomousDatabases?compartmentId={ocid}` — paginate with
`limit`/`page` and stop when `opc-next-page` is absent.
