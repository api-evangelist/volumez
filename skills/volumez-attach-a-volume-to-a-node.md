---
name: Attach a volume to a node
description: Bind a Volumez volume or snapshot to a connector host, verify the attachment, and detach it cleanly.
api: openapi/volumez-orchestrator-api-openapi.yaml
operations: [NodesList, NodeGet, VolumesList, AttachmentsListForVolume, AttachmentCreate, AttachmentGet, AttachmentModify, AttachmentDelete, AttachmentsListAll, JobGet]
generated: '2026-09-04'
method: generated
source: Grounded in operationIds verified verbatim in openapi/volumez-orchestrator-api-openapi.yaml.
---

# Attach a volume to a node

An **Attachment** is the binding of a volume (addressed through one of its snapshots in the path
grammar) to a **Node** — a host running the Volumez connector. It carries the `mountpoint`,
a `readonly` flag and `allocated_resources`.

## Steps

1. **Pick the node.** `NodesList` (`GET /nodes`) then `NodeGet` (`GET /nodes/{node}`). Check `state`,
   `connectorversion` and that `zone`/`region` match the volume's placement.

2. **Check what is already attached.** `AttachmentsListForVolume`
   (`GET /volumes/{volume}/attachments`) shows every node this volume is bound to;
   `AttachmentsListAll` (`GET /attachments`) shows the whole tenant. Attaching a volume that is
   already attached elsewhere is a real decision, not an accident — make it deliberately.

3. **Create the attachment.**
   `AttachmentCreate` (`POST /volumes/{volume}/snapshots/{snapshot}/attachments`). Note the path
   shape: attachments hang off a snapshot of the volume, not off the volume directly.
   Asynchronous — poll `JobGet` (`GET /jobs/{job}`) with the id returned in `Message`.

4. **Verify.** `AttachmentGet`
   (`GET /volumes/{volume}/snapshots/{snapshot}/attachments/{node}`) — confirm `state`, `mountpoint`
   and `readonly`. `AttachmentModify` (`PATCH …/attachments/{node}`) changes it in place.

5. **Detach.** `AttachmentDelete` (`DELETE …/attachments/{node}`) takes an optional `force` query
   flag. Detach before draining or deleting the node, or the node operation will contend with a live
   binding.

## Publishing instead of attaching

If the consumer is not a Volumez-connected node, use an **Export** instead: `ExportCreate`
(`POST /exports/`) then `ExportConnectScript` (`GET /exports/{export}/connect-script`), which returns
the client-side connect script. `ExportDelete` (`DELETE /exports/{export}`) reverses it.

## Reversibility

Attach/detach is fully reversible with no data loss and no published window — `AttachmentDelete`
reverses `AttachmentCreate`, `ExportDelete` reverses `ExportCreate`. Deleting the underlying volume
is not.
