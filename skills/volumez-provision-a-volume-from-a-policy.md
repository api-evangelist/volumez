---
name: Provision a volume from a policy
description: Declare the IO outcome you need as a Volumez Policy, rehearse the placement, create the volume, and wait for the asynchronous job to finish.
api: openapi/volumez-orchestrator-api-openapi.yaml
operations: [PoliciesList, PolicyCreate, PolicyGet, PolicyPlan, BatchVolumesPlan, VolumeCreate, JobGet, VolumeGet, VolumeDescribe]
generated: '2026-09-04'
method: generated
source: Grounded in operationIds verified verbatim in openapi/volumez-orchestrator-api-openapi.yaml.
---

# Provision a volume from a policy

Volumez is declarative: you never ask for a device, you ask for an outcome. A **Policy** states the
IOPS, bandwidth, latency, resiliency and encryption you need; a **Volume** is what Volumez composes
out of assigned media to satisfy it.

## Before you start

- Send a JWT in the `authorization` header on every call (see `../authentication/volumez-authentication.yml`).
- Read `../conventions/volumez-conventions.yml`. Two rules dominate this flow: **creates are
  asynchronous**, and **there is no idempotency**.

## Steps

1. **Find or declare the policy.**
   `PoliciesList` (`GET /policies`) lists what already exists; `PolicyGet` (`GET /policies/{policy}`)
   reads one. If nothing fits, `PolicyCreate` (`POST /policies`) declares a new one. The fields that
   matter are `iopsread`, `iopswrite`, `bandwidthread`, `bandwidthwrite`, `latencyread`,
   `latencywrite`, `failureperformance`, `resiliencymedia`, `resiliencynode`, `resiliencyzone`,
   `capacityoptimization`, `encryption`, `sed`, `integrity`, and the snapshot schedule
   (`snapshotkeep`, `snapshotfrequency`, `snapshotday`, `snapshothour`, `snapshotminute`).

2. **Rehearse before you spend.** This step is not optional for an agent.
   `PolicyPlan` (`GET /policies/{policy}/size/{size}/zone/{zone}`) returns the placement a single
   volume of that size in that zone would get. For several volumes at once use `BatchVolumesPlan`
   (`POST /volumes/plan`) with `verbose=true` to get the plan back in the response. Neither call
   creates anything. If the plan cannot be satisfied, fix the policy or the capacity **before**
   calling create — a failed create still consumes a job and may leave partial state.

3. **Create the volume.**
   `VolumeCreate` (`POST /volumes`) with the volume name, size and policy name.
   **This is not idempotent.** There is no `Idempotency-Key` header and no conditional-request
   support anywhere in this API. If the call times out, do **not** blind-retry: call `VolumesList`
   (`GET /volumes`) and check whether your name already exists, or check `JobsList` (`GET /jobs`) for
   an in-flight job whose `object` is your volume. A blind retry can provision a second volume across
   real cloud media.

4. **Wait for the job.**
   The create returns `{"Message": "<jobId>"}` — the job id arrives in a field named `Message`, not
   `id`. Poll `JobGet` (`GET /jobs/{job}`) until `state == "done"` and `progress == 100`. A failed
   job returns the richer `ErrorJobResponse` shape (`Message`, `ErrorCode`, `ObjectID`, `JobID`); the
   numeric `ErrorCode` values are not published, so log the whole object.

5. **Confirm.**
   `VolumeGet` (`GET /volumes/{volume}`) for state, or `VolumeDescribe`
   (`GET /volumes/{volume}/describe`) for the expanded view. A healthy volume reports its `state`,
   `status` and `progress`, plus the `policy` and `capacitygroup` it landed in.

## Errors

Every operation here declares 400, 404 and 500 with a bare `{"message": string}` body — no error
code, no field path, no retry hint. See `../errors/volumez-problem-types.yml`. Treat a 500 on step 3
as **possibly succeeded**: re-read state before retrying.

## Undoing this

`VolumeDelete` (`DELETE /volumes/{volume}`) removes the volume. There is no soft delete, no
restore-a-deleted-volume operation and no published grace period — the only thing that survives a
delete is a snapshot taken beforehand. See the `reversibility` block in
`../conventions/volumez-conventions.yml`.
