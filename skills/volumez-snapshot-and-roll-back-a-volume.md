---
name: Snapshot and roll back a volume
description: Take point-in-time and consistency-group snapshots of Volumez volumes, and use rollback as the reversal path for a bad write.
api: openapi/volumez-orchestrator-api-openapi.yaml
operations: [SnapshotsList, SnapshotsListAll, SnapshotCreate, SnapshotGet, SnapshotModify, SnapshotRollback, SnapshotDelete, ConsistencyGroupSnapshotCreate, ConsistencyGroupGet, JobGet, VolumeRecoverInitiate]
generated: '2026-09-04'
method: generated
source: Grounded in operationIds verified verbatim in openapi/volumez-orchestrator-api-openapi.yaml.
---

# Snapshot and roll back a volume

Snapshots are the **only** thing that survives a volume delete in this API, and rollback is the
primary reversal path for a bad write. Treat this skill as the safety net around every other one.

## Take a snapshot

- One volume: `SnapshotCreate` (`POST /volumes/{volume}/snapshots`).
- Several volumes atomically: `ConsistencyGroupSnapshotCreate` (`POST /volumes/snapshot`) — use this
  whenever the volumes hold one application's state (a database and its log, for example). Read the
  group back with `ConsistencyGroupGet` (`GET /volumes/snapshot/{snapshot_group_name}`).
- Both are asynchronous. Poll `JobGet` (`GET /jobs/{job}`) with the id returned in `Message`.

Policies can also schedule snapshots without any API call: the Policy fields `snapshotkeep`,
`snapshotfrequency`, `snapshotday`, `snapshothour` and `snapshotminute` define the schedule and the
retention. **How far back you can roll is whatever that policy says** — Volumez publishes no default
and no provider-side retention guarantee.

## Inspect

`SnapshotsList` (`GET /volumes/{volume}/snapshots`) for one volume, `SnapshotsListAll`
(`GET /snapshots`) for the tenant, `SnapshotGet`
(`GET /volumes/{volume}/snapshots/{snapshot}`) for one. Each carries `time`, `consistency`, `used`,
`state` and `numberofattachments`. `SnapshotModify` (`PATCH …/{snapshot}`) renames or re-tags it.

## Roll back

`SnapshotRollback` (`PATCH /volumes/{volume}/snapshots/{snapshot}/rollback`) restores the volume to
that snapshot. It returns 200 with a `RegularResponse` on success and a bare `{"message": string}`
500 on failure — there is no partial-rollback state to inspect, so re-read the volume with
`VolumeGet` afterwards.

Before rolling back:

1. Confirm nothing is writing — check `AttachmentsListForVolume` (`GET /volumes/{volume}/attachments`).
2. Confirm the snapshot you picked with `SnapshotGet`; the rollback is addressed by **name**, and
   names are the identity in this API.
3. There is no dry-run for rollback and no published undo of a rollback. Take a fresh snapshot first
   if the current contents might still be wanted.

## If the volume itself is unhealthy

`VolumeRecoverInitiate` (`POST /volumes/{volume}/recover`) starts recovery; watch progress through
the volume's `volumerecoveryjob` field and `JobGet`.

## Clean up

`SnapshotDelete` (`DELETE /volumes/{volume}/snapshots/{snapshot}`, optional `force`). Deleting a
snapshot removes a reversal path — check `numberofattachments` and whether any volume was created
from it (`contentsnapshot`) before you do.
