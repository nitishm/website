+++
title = "Mesh in a Mesh: A Model for Stronger Multi-tenancy of Kubernetes Workloads"
date = 2020-08-18T02:13:50Z
author = "Nitish Malhotra"
slug = "kubecon-eu-2020"
tags = ["kubernetes","golang", "istio", "open-source"]
+++
---

## KubeCon + CloudNativeCon Europe 2020

*[Mesh in a Mesh: A Model for Stronger Multi-tenancy of Kubernetes Workloads](https://kccnceu20.sched.com/event/Zelb)*

{{< youtube 7XMgtfey5YI >}}

Most organizations use some flavor of multi-tenancy in their clusters for their teams, applications or customers. Namespaces paired with RBAC, ResourceQuota & NetworkPolicy provide ways to isolate tenants at some level. However these primitives are insufficient for setting L7 policies on the ingress/egress traffic per tenant.

If you’re administering clusters for multiple tenants, attend this talk to learn a new model for deploying workloads in tenant specific sub-meshes within your service mesh. This model uses Envoy based ingress/egress gateways per-tenant, which helps:

- set traffic policies and scale resources per-tenant
- hide tenant application topology when communicating with external entities
- assign per-tenant identities (SPIFFE) for use with transport authentication (mTLS), authorization (OAuth2) & origin authentication (JWT)
- share mesh control plane resources across tenants

{{ template "_internal/disqus.html" . }}
