# Kamaji Project Overview

> AI-Generated Documentation | Scan Date: 2026-01-19 | Scan Level: Exhaustive

## Project Summary

**Kamaji** is a Kubernetes Control Plane Manager that implements the Hosted Control Plane (HCP) pattern. It allows you to run Kubernetes Control Plane components (API Server, Controller Manager, Scheduler) as Pods in a management cluster, enabling multi-tenant Kubernetes clusters at scale with reduced operational overhead.

| Attribute | Value |
|-----------|-------|
| **Repository** | github.com/clastix/kamaji |
| **Type** | Kubernetes Operator (Go) |
| **License** | Apache 2.0 |
| **Organization** | CLASTIX Labs |
| **Go Version** | 1.25 |
| **Kubernetes Version** | v1.35.0 |

## Source Code Statistics

| Metric | Count |
|--------|-------|
| Go Source Files | 189 |
| Lines of Go Code | ~24,146 |
| CRDs Defined | 3 |
| Controllers | 10 |
| Webhook Handlers | 15 |

## Core Concepts

### TenantControlPlane (TCP)

The primary Custom Resource representing a managed Kubernetes control plane. Each TCP:

- Runs as a Deployment in the management cluster
- Contains kube-apiserver, kube-controller-manager, and kube-scheduler
- Optionally includes Kine sidecar for non-etcd storage
- Manages its own certificates, kubeconfigs, and secrets
- Can expose API via Service (ClusterIP/NodePort/LoadBalancer), Ingress, or Gateway API

### DataStore

Cluster-scoped resource defining the backing storage for control plane state:

- **etcd** - Native Kubernetes etcd storage
- **MySQL** - MySQL/MariaDB via Kine proxy
- **PostgreSQL** - PostgreSQL via Kine proxy
- **NATS** - NATS JetStream via Kine proxy

### KubeconfigGenerator

Cluster-scoped resource for generating dynamic kubeconfigs:

- Selector-based targeting of namespaces and TenantControlPlanes
- Dynamic user/group extraction via JSON path expressions
- Automatic certificate lifecycle management

## Architecture Pattern

Kamaji follows the standard Kubernetes Operator pattern using:

- **controller-runtime** for reconciliation loop management
- **Kubebuilder v3** for scaffolding and code generation
- **Admission Webhooks** for validation and mutation
- **SOOT Controllers** - In-tenant controllers running inside each TCP

### Reconciliation Flow

```
TenantControlPlane CR
        │
        ▼
┌───────────────────┐
│ TenantControlPlane│
│    Reconciler     │
└───────┬───────────┘
        │
        ├─► DataStore Connection
        │
        ├─► Certificate Generation
        │
        ├─► Kubeconfig Creation
        │
        ├─► Deployment Management
        │
        ├─► Service/Ingress/Gateway
        │
        └─► Status Updates
```

## Directory Structure

```
kamaji/
├── api/v1alpha1/           # CRD type definitions
│   ├── tenantcontrolplane_types.go
│   ├── datastore_types.go
│   └── kubeconfiggenerator_types.go
├── cmd/                    # CLI commands
│   ├── manager/            # Main operator command
│   ├── migrate/            # Migration utility
│   └── kubeconfig-generator/
├── controllers/            # Kubernetes controllers
│   ├── tenantcontrolplane_controller.go
│   ├── datastore_controller.go
│   ├── certificate_lifecycle_controller.go
│   └── soot/               # In-tenant controllers
├── internal/               # Internal packages
│   ├── datastore/          # Storage driver implementations
│   ├── resources/          # Resource builders
│   ├── webhook/            # Admission handlers
│   ├── kubeadm/            # Kubeadm integration
│   └── builders/           # Object builders
├── charts/                 # Helm charts
│   ├── kamaji/             # Main chart
│   └── kamaji-crds/        # CRD-only chart
├── deploy/                 # Deployment manifests
├── e2e/                    # End-to-end tests
├── docs/                   # MkDocs documentation
└── hack/                   # Development scripts
```

## Key Features

1. **Fast Provisioning** - Control planes ready in ~16 seconds
2. **Zero-Downtime Upgrades** - Blue/green deployment strategy
3. **Multi-Driver Storage** - etcd, MySQL, PostgreSQL, NATS
4. **Konnectivity Support** - Worker nodes in different networks
5. **Gateway API Integration** - Modern ingress routing
6. **Cluster API Provider** - Integration with CAPI
7. **Write Permissions** - Read-only mode for quota protection
8. **Automated Certificate Rotation** - Proactive certificate management

## Configuration Options

### CLI Flags (manager command)

| Flag | Default | Description |
|------|---------|-------------|
| `--metrics-bind-address` | `:8080` | Metrics endpoint |
| `--health-probe-bind-address` | `:8081` | Health check endpoint |
| `--leader-elect` | `true` | Enable leader election |
| `--datastore` | `""` | Default DataStore name |
| `--kine-image` | `rancher/kine:v0.11.10` | Kine sidecar image |
| `--max-concurrent-tcp-reconciles` | `1` | Worker count |
| `--controller-reconcile-timeout` | `30s` | Reconciliation timeout |
| `--certificate-expiration-deadline` | `24h` | Cert rotation threshold |

## Dependencies

### Core Dependencies

- `sigs.k8s.io/controller-runtime` v0.22.4
- `k8s.io/client-go` v0.35.0
- `k8s.io/kubernetes` v1.35.0
- `sigs.k8s.io/gateway-api` v1.4.0
- `github.com/go-sql-driver/mysql` v1.9.1
- `github.com/go-pg/pg/v10` v10.14.0
- `github.com/nats-io/nats.go` v1.42.0

### Development Tools

- `controller-gen` v0.20.0
- `golangci-lint` v2.0.2
- `ginkgo` (testing framework)
- `ko` v0.18.1 (container builds)
- `helm` v3.9.0

## Related Projects

- [Cluster API Control Plane Provider](https://github.com/clastix/cluster-api-control-plane-provider-kamaji)
- [Kamaji Console](https://github.com/clastix/kamaji-console)
- [Capsule](https://github.com/projectcapsule/capsule) - Multi-tenant Kubernetes
