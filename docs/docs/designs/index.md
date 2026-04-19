---
title: Design Documents
description: Proposed designs that are not yet part of the settled GilmanLab architecture baseline.
slug: /designs/
---

# Design Documents

This section holds proposed designs that are specific enough to guide
implementation, but are not yet part of the settled architecture baseline.

Use these documents when:

- a design is clear enough to review formally
- the target implementation does not exist yet
- the architecture overview should stay conservative until the design is proven

Current designs:

- [Bootstrap and Core Delivery Model](./bootstrap-core-delivery.md)
- [Service Exposure and Control Plane Endpoints](./service-exposure-and-control-plane-endpoints.md)
- [Multi-Cluster GitOps Model](./gitops-multi-cluster.md)
- [kro Consumption Model](./kro-consumption-model.md)
- [Platform RGD Delivery Model](./platform-rgd-delivery.md)
- [App RGD Design](./app-rgd.md)

Once a design is implemented and considered durable, its steady-state shape
should be folded back into the architecture overview and any relevant runbooks.
