---
name: Onboard a node and assign media
description: Bring a connector host and its NVMe devices under Volumez control, then drain and remove them safely.
api: openapi/volumez-orchestrator-api-openapi.yaml
operations: [NodesList, NodeGet, NodeDescribe, NodeHwScan, NodeSetTags, NodeUpgrade, NodeDrain, NodeDelete, MediaList, MediaGet, MediaAssign, MediaUnassign, MediaDrain, MediaProfileModify, MediaDelete, capacityGroupGet, JobGet]
generated: '2026-09-04'
method: generated
source: Grounded in operationIds verified verbatim in openapi/volumez-orchestrator-api-openapi.yaml.
---

# Onboard a node and assign media

Volumez composes volumes out of the **Media** — NVMe devices — it finds on **Nodes** running the
connector. Nothing can be provisioned until media is assigned.

## Bring the node in

1. Stand up the host with the first-party Terraform/Bicep modules
   (`https://github.com/VolumezTech/volumez`) and install the connector with the tenant token. That
   part is infrastructure, not API.
2. `NodesList` (`GET /nodes`) — the node appears with its `instanceid`, `os`, `kversion`,
   `connectorversion`, `region`, `zone` and fault/resiliency domains.
3. `NodeDescribe` (`GET /nodes/{node}/describe`) for the full hardware view; `NodeGet`
   (`GET /nodes/{node}`) for state.
4. `NodeSetTags` (`PATCH /nodes/tags/{node}`) to label it. Tags are the only free-form metadata
   surface in this API — volumes have none — so put anything you need to correlate later here.

## Discover and assign media

1. `NodeHwScan` (`POST /nodes/{node}/hw`, optional `properties[]`) rescans the host's hardware.
2. `MediaList` (`GET /media`) / `MediaGet` (`GET /media/{media}`) — each device reports `size`,
   `model`, measured `iopsread`/`iopswrite`/`bandwidthread`/`bandwidthwrite` and their `free*`
   counterparts, plus `assignment` and `capacitygroup`. Media ids look like `AWS29D8D9AAAC8C865C6`
   (cloud-provider prefix + hex).
3. `MediaAssign` (`PATCH /media/{media}/assign`) brings a device under Volumez control. Asynchronous
   — the response is `{"Message": "<jobId>"}`; poll `JobGet` (`GET /jobs/{job}`) until
   `state == "done"`.
4. `MediaProfileModify` (`PATCH /media/{media}/profile`) sets its performance profile.
5. `capacityGroupGet` (`GET /capacitygroups`) shows the capacity groups volumes and media land in.

## Take it back out — in this order

1. `MediaDrain` (`POST /media/{media}/drain`) — moves data off the device.
2. `MediaUnassign` (`PATCH /media/{media}/unassign`) — the direct reversal of `MediaAssign`.
3. `NodeDrain` (`POST /nodes/{node}/drain`, optional `cleanup=true`) — moves work off the host.
4. `NodeDelete` (`DELETE /nodes/{node}`) — takes `force` **and** `delayDelete`. `delayDelete=true`
   defers the removal; its length is not documented anywhere, so do not depend on a specific window.

Draining is what makes this reversible. Deleting media (`MediaDelete`) or a node without draining
first can strand volume replicas — check `volumescount` on the media and
`PolicyGetVolumes`/`VolumesList` for anything placed on that node before you act.

## Upgrades

`NodeUpgrade` (`POST /nodes/upgrade/{node}`) upgrades the connector. Do one node at a time and wait
for the job; there is no batch upgrade operation and no maintenance-window concept in the API.
