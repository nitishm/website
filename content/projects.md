+++
title = "Open Source"
date = 2020-03-22T02:13:50Z
author = "Nitish Malhotra"
slug = "projects"
tags = ["github","open-source"]
+++
---

### [Azure/Orkestra](https://github.com/Azure/orkestra)

> Orkestra is a cloud-native **Release Orchestration** and **Lifecycle Management (LCM)** platform for a related group of [Helm](https://helm.sh/) releases and their subcharts.

Orkestra is built on top of popular [CNCF](https://cncf.io/) tools and technologies like,

- [Helm](https://helm.sh/)
- [Argo Workflows](https://argoproj.github.io/workflows/),
- [Flux Helm Controller](https://github.com/fluxcd/helm-controller),
- [Chartmuseum](https://chartmuseum.com/)
- [Keptn](https://keptn.sh)

{{< figure src="https://raw.githubusercontent.com/Azure/orkestra/main/docs/assets/orkestra-core.png" title="Azure Orkestra" >}}

Orkestra is one solution to introduce Helm release orchestration. Orkestra provides this by building on top of **Argo Workflows**, a workflow engine on top of Kubernetes for workflow orchestration, where each step in a workflow is executed by a Pod. As such, Argo Workflow engine is a more powerful, more flexible adaptation of what **Init Containers** and **Kubernetes Jobs** provide without the orchestration.

Argo enables a DAG based dependency graph with defined workflow steps and conditions to transition through the graph, as well as detailed insight into the graph and its state. Helm releases matching transitions in the graph are executed by the FluxCD Helm controller operator. The FluxCD Helm controller operator is a Kubernetes operator that is responsible for executing Helm releases in a consistent manner.

The unit of deployment for Orkestra based Helm releases is based on a workflow definition with a custom resource type that models the relationship between individual Helm releases making up the whole. The workflow definition is a **DAG** with defined workflow steps and conditions.

The `ApplicationGroup` spec allows to structure an orchestrated set of releases through grouping Helm releases into an group, either through defining a sequence on non-related charts and/or charts with subcharts, where subcharts are not merged into a single release but are executed as a release of their own inside a workflow step. The `ApplicationGroup` spec also allows to define a set of conditions that are evaluated at the beginning of the workflow and if any of the conditions fail, the whole workflow is aborted.

This gives you the ability to define a set of Helm releases that are orchestrated in a way that is easy to understand and to debug without having to modify the Helm release itself.

#### Features 🌟

- **Layers** - Deploy and manage 'layers' on top of Kubernetes. Each layer is a collection of addons and can have dependencies established between the layers.
- **Dependency management** - DAG-based workflows for groups of application charts and their sub-charts using Argo Workflows.
- **Fail fast during in-service upgrades** - limits the blast radius for failures during in-service upgrade of critical components to the immediate components that are impacted by the upgrade.
- **Failure Remediation** - rollback to last successful spec on encountering failures during in-service upgrades
- **Built for Kubernetes** - custom controller built using  [kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)
- **Easy to use** - familiar declarative spec using Kubernetes [Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- **Works with any Continous Deployment system** - bring your own CD framework to deploy Orkestra Custom Resources. Works with any Kubernetes compatible Continuous Deployment framework like [FluxCD](https://fluxcd.io/) and [ArgoCD](https://argoproj.github.io/argo-cd/)
- **Built for GitOps** - describe your desired set of applications (and dependencies) declaratively and manage them from a version-controlled git repository

### [Go-ReJSON](https://github.com/nitishm/go-rejson)

Go-ReJSON is a Go client for ReJSON Redis Module.

> ReJSON is a Redis module that implements ECMA-404 The JSON Data Interchange Standard as a native data type. It allows storing, updating and fetching JSON values from Redis keys (documents).

Primary features of ReJSON Module:

* Full support of the JSON standard
* JSONPath-like syntax for selecting element inside documents
* Documents are stored as binary data in a tree structure, allowing fast access to sub-elements
* Typed atomic operations for all JSON values types

> Each and every feature of ReJSON Module is fully incorporated in the project.

Enjoy ReJSON with the type-safe Redis client, Go-Redis/Redis or use the print-like Redis-api client GoModule/Redigo. Go-ReJSON supports both the clients. Use any of the above two client you want, Go-ReJSON helps you out with all its features and functionalities in a more generic and standard way.

Support for mediocregopher/radix and other Redis clients is in our RoadMap. Any contributions on the support for other clients is hearty welcome.

---

### [Vegeta-Server](https://github.com/nitishm/vegeta-server)

A RESTful API server for [vegeta](https://github.com/tsenart/vegeta), a load testing tool written in Go.

Vegeta is a versatile HTTP load testing tool built out of a need to drill HTTP services with a constant request rate. The vegeta library is written in Go, which makes it ideal to implement server in Go.

The REST API model enables users to, asynchronously, submit multiple attacks targeting the same (or varied) endpoints, lookup pending or historical reports and update/cancel/re-submit attacks, using a simple RESTful API.

---

### [Engarde](https://github.com/nitishm/engarde)

Parse default [envoy access logs](https://www.envoyproxy.io/docs/envoy/v1.8.0/configuration/access_log#default-format) like a champ with engarde and [jq](https://github.com/stedolan/jq)

---

### [Logger](https://github.com/nitishm/logger)

A wrapper around [logrus](https://github.com/sirupsen/logrus) that allows setting default fields, and supports adding and removing configurable fields.
