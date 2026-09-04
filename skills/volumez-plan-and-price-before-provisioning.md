---
name: Plan and price infrastructure before provisioning
description: Use the Volumez planner and pricing endpoints to rehearse and cost a provisioning decision without creating anything.
api: openapi/volumez-orchestrator-api-openapi.yaml
operations: [PolicyPlan, BatchVolumesPlan, createInfraPlan, createPublicInfraPlan, ProviderPricingInfo, ConnectivityCheck, getVMZones, getVMVPCs, getVMVPCsByRegion, AutoProvisionVolumes, Provision, JobGet]
generated: '2026-09-04'
method: generated
source: Grounded in operationIds verified verbatim in openapi/volumez-orchestrator-api-openapi.yaml.
---

# Plan and price before provisioning

This API has a genuine rehearsal surface — unusual, and the single most useful thing in it for an
agent, because provisioning here spends real cloud money and cannot be un-spent.

## Rehearse

- `PolicyPlan` (`GET /policies/{policy}/size/{size}/zone/{zone}`) — the placement one volume of that
  size in that zone would get under that policy.
- `BatchVolumesPlan` (`POST /volumes/plan`, `verbose=true`) — the same for a batch. With
  `verbose=false` the plan is omitted from the response, so pass `true` when you actually want to
  read it.
- `createInfraPlan` (`POST /infra-planner/create-infra-plan`) — the cloud infrastructure (instance
  types, counts, media) a policy would require. `createPublicInfraPlan`
  (`POST /infra-planner/create-infra-plan/public`) is the **unauthenticated** variant; it is one of
  the sixteen operations in this API that needs no token.
- `ConnectivityCheck` (`POST /connectivities/test`) — validate a path before `ConnectivityCreate`.

None of these create anything.

## Price

`ProviderPricingInfo` (`POST /infra-planner/provider-pricing-info`) returns `ProviderPriceItem`
entries for the plan. **Read what this prices:** it prices the *cloud provider's* instances and
media, not Volumez's own subscription. Volumez publishes no rate card of its own — see
`../plans/volumez-plans-pricing.yml`.

## Check the target environment

`getVMZones` (`GET /tenant-cloud-resources/vm/zones`), `getVMVPCs`
(`GET /tenant-cloud-resources/{cloudProviderAccountId}/vm/vpcs`) and `getVMVPCsByRegion`
(`GET /tenant-cloud-resources/{cloudProviderAccountId}/vm/{region}/vpcs`) tell you which zones and
VPCs the linked cloud account can actually place into. Plan against those, not against a region you
assumed.

## Then commit

- `AutoProvisionVolumes` (`POST /autoprovisionvolumes`) provisions volumes together with the
  infrastructure they need, in one call.
- `Provision` (`POST /provision`) is the newer provisioning-service surface, added in the 2025-09-28
  contract revision.

Both are asynchronous and both return the `ErrorJobResponse` shape (`Message`, `ErrorCode`,
`ObjectID`, `JobID`) on failure. Poll `JobGet` (`GET /jobs/{job}`).

**There is no idempotency on either.** Plan, price, decide, then make exactly one call and follow the
job — never a blind retry. See `../conventions/volumez-conventions.yml`.
