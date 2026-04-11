---
title: Tenant System
description: "Learn about tenants, the way Cozystack helps manage resources and improve security."
weight: 17
---

## Introduction

A **tenant** in Cozystack is the primary unit of isolation and security, analogous to a Kubernetes namespace but with enhanced scope.
Each tenant represents an isolated environment with its own resources, networking, and RBAC (role-based access control).
Some cloud providers use the term "projects" for a similar entity.

Cozystack administrators and users create tenants using the [Tenant application]({{% ref "/docs/v1/applications/tenant" %}})
from the application catalog.
Tenants can be created via the Cozystack dashboard (UI), `kubectl`, or directly via Cozystack API.


### Tenant Nesting

All user tenants belong to the base `root` tenant.
This `root` tenant is used only to deploy user tenants and system components.
All user-side applications are deployed in their respective tenants.

Tenants can be nested further: an administrator of a tenant can create sub-tenants as applications in the Cozystack catalog.
Parent tenants can share their resources with their children and oversee their applications.
In turn, children can use their parent's services.

![tenant hierarchy diagram](./tenants1.png)


### Sharing Cluster Services

Tenants may have [cluster services]({{% ref "/docs/v1/operations/services" %}}) deployed in them.
Cluster services are middleware services providing core functionality to the tenants and user-facing applications.

The `root` tenant has a set of services like `etcd`, `ingress`, and `monitoring` by default.
Lower-level tenants can run their own cluster services or access ones of their parent.

For example, a Cozystack user creates the following tenants and services:

- Tenant `foo` inside of tenant `root`, having its own instances of `etcd` and `monitoring` running.
- Tenant `bar` inside of tenant `foo`, having its own instance of `etcd`.
- [Tenant Kubernetes cluster]({{% ref "/docs/v1/kubernetes" %}}) and a
  [Postgres database]({{% ref "/docs/v1/applications/postgres" %}}) in the tenant `bar`.

All applications need services like `ingress` and `monitoring`. 
Since tenant `bar` does not have these services, the applications will use the parent tenant's services.

Here's how this configuration will be resolved:

-   The tenant Kubernetes cluster will store its data in the `bar` tenant's own `etcd` service.
-   All metrics will be collected in the monitoring stack of the parent tenant `foo`.
-   Access to the applications will be through the common `ingress` deployed in the tenant `root`.

![tenant services](./tenants2.png)


### Network Isolation Between Tenants

Every tenant namespace is isolated from its siblings by Cilium network
policies installed automatically by the `tenant` chart. There is no
per-tenant opt-out: the previous `isolated` field was removed in
Cozystack v1.0. Pods inside a tenant namespace also cannot reach
`kube-apiserver` by default, or the tenant's own `etcd` when the tenant
was created with `etcd: true` — they need to opt in with one of two pod
labels:

-   `policy.cozystack.io/allow-to-apiserver: "true"` — reach the
    in-cluster Kubernetes API (for operators, dashboards, etc.).
-   `policy.cozystack.io/allow-to-etcd: "true"` — reach the tenant's
    own etcd (only applicable when the tenant was created with
    `etcd: true`).

See [Tenant `isolated` flag removed]({{% ref "/docs/v1/operations/upgrades#tenant-isolated-flag-removed" %}})
in the upgrade notes for a full worked example.


### Unique Domain Names

Each tenant has its own domain.
By default, (unless otherwise specified), it inherits the domain of its parent with a prefix of its name.
For example, if the `root` tenant has domain `example.org`, then tenant `foo` gets the domain `foo.example.org` by default.
However, it can be redefined to have another domain, such as `example.com`.

Kubernetes clusters created in this tenant namespace would get domains like: `kubernetes-cluster.foo.example.org`


### Tenant Naming Limitations

Tenant names must be alphanumeric.
Using dashes (`-`) in tenant names is not allowed, unlike with other services.
This limitation exists to keep consistent naming in tenants, nested tenants, and services deployed in them.

For example:

-   The root tenant is named `root`, but internally it's referenced as `tenant-root`.
-   A user tenant is named `foo`, which results in `tenant-foo`.
-   However, a tenant cannot be named `foo-bar`, because parsing names like `tenant-foo-bar` can be ambiguous.


### Tenant Namespace Layout

Each tenant corresponds to a Kubernetes workload namespace whose name encodes
the tenant's position in the hierarchy. The root tenant is always
`tenant-root`, and nested tenants follow two rules:

-   Tenants created directly inside `tenant-root` get a **flat** namespace of
    the form `tenant-<name>`. There is no `tenant-root-` prefix.
-   Tenants created at any deeper level get a **hierarchical** namespace of
    the form `<parent-workload-namespace>-<name>`.

For example, starting from `tenant-root`:

| Tenant path             | Workload namespace         |
| ---                     | ---                        |
| `root`                  | `tenant-root`              |
| `root/alpha`            | `tenant-alpha`             |
| `root/alpha/beta`       | `tenant-alpha-beta`        |
| `root/alpha/beta/gamma` | `tenant-alpha-beta-gamma`  |

This layout is produced by both the `tenant` Helm chart
([`packages/apps/tenant/templates/_helpers.tpl`](https://github.com/cozystack/cozystack/blob/main/packages/apps/tenant/templates/_helpers.tpl))
and the aggregated API
([`pkg/registry/apps/application/rest.go::computeTenantNamespace`](https://github.com/cozystack/cozystack/blob/main/pkg/registry/apps/application/rest.go)).
Because tenant names themselves are constrained to be alphanumeric (see
*Tenant Naming Limitations* above), namespace fragments never contain
tenant-internal dashes.


### Deriving Parent and Child Relationships

Downstream integrations — custom dashboards, audit tooling, cost-allocation
jobs, policy engines — sometimes need to walk the tenant tree to render
breadcrumbs, compute inherited settings, or scope queries. It is **not
reliable** to do this by string-parsing the workload namespace name: the
root-level case (`tenant-alpha`) is flat while deeper levels
(`tenant-alpha-beta`) are hierarchical, so a single `strings.Split` rule is
wrong for one of the two cases.

Use the `Tenant` custom resource itself instead. Cozystack stores every
`Tenant` CR in its parent's workload namespace, so:

-   **`metadata.namespace`** of a `Tenant` CR equals the **parent's** workload
    namespace. This is the reliable pointer to the parent — no string parsing
    required.
-   **`status.namespace`** of a `Tenant` CR equals the tenant's **own** workload
    namespace (the one where the tenant's applications, nested tenants, and
    `HelmRelease`s live).
-   To list the direct children of a tenant with workload namespace `N`, list
    `Tenant` CRs whose `metadata.namespace == N`.

This approach is stable regardless of whether the tenant is a direct child of
`tenant-root` or a deeper descendant, and it survives any future adjustments
to the namespace layout because it does not depend on the layout at all.


### Reference

See the reference for the application implementing tenant management: [`tenant`]({{% ref "/docs/v1/applications/tenant#parameters" %}})

