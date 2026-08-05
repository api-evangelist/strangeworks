---
name: Discover which Strangeworks compute backends are available and online
description: >-
  Query the backend catalog to pick a QPU, simulator or optimization solver that is
  actually ONLINE, understand which product it belongs to, and activate a resource for it.
api: graphql/strangeworks-sdk.graphql
operations: [Query.backends, Query.backend, Query.productCatalog, Query.product, Mutation.resourceCreate]
generated: '2026-08-05'
method: generated
source: graphql/strangeworks-sdk.graphql + graphql/strangeworks-platform.graphql
---

# Discover Strangeworks backends

Strangeworks brokers many providers behind one contract. Before submitting work, resolve
which **Backend** you want and whether it is reachable right now.

## 1. List backends

```graphql
query { backends { slug name status product { slug name type }
  tags { tag { slug } }
  backendRegistrations { backendType { slug } } } }
```

Available on all three endpoints (`/sdk`, `/platform`, `/products`) — use `/sdk` with a
workspace token.

## 2. Respect status

`BackendStatus` is `ONLINE, OFFLINE, MAINTENANCE, RETIRED, UNKNOWN`.

- Submit only to `ONLINE`.
- `MAINTENANCE` is temporary — retry later, do not fail the workflow.
- `RETIRED` will not come back. Re-plan onto a different backend.
- `UNKNOWN` means Strangeworks could not reach the provider; treat as not-submittable.

Never cache a backend's status across a run. QPU availability changes on windowed
schedules set by the hardware provider, not by Strangeworks.

## 3. Understand what you are picking

A Backend belongs to a **Product**, and `ProductType` tells you what kind of thing it is:
`MANAGED_APP`, `COMPUTE_PROVIDER`, `TOOL`, `ENTERPRISE`, `DEMO`. Quantum hardware and
solvers are `COMPUTE_PROVIDER`. Browse everything offered with `productCatalog`, and read
one product with `product`.

Providers reachable through the platform include IBM Quantum, Amazon Braket, Azure
Quantum, IonQ, Rigetti, Quantinuum, QuEra, IQM, AQT and Pasqal on the quantum side, and
D-Wave, Toshiba, Hitachi, Fujitsu, NEC, Gurobi, JIJ, LightSolver and Quantagonia on the
optimization side.

## 4. Activate a resource

You cannot submit to a Backend directly. Activate its Product into a **Resource**:

```graphql
mutation($input: ResourceCreateInput!) { resourceCreate(input: $input) { resource { slug status } } }
```

The new resource starts `PENDING_ACTIVATION`. Poll `Workspace.resource(resourceSlug:)`
until `status` is `ACTIVE`. Set a workspace default with `resourceSetDefault`.

Then submit through the resource proxy — see the submit-and-poll skill.

## Cost caution

Activating a resource is usually free; **running** on it is not. Real QPU time and
commercial solver time bill per use, and each paid action writes a `BillingTransaction`.
Confirm cost with a human before submitting to hardware. Simulators are the cheap path for
a first run.
