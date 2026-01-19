# Kamaji Architecture Deep Dive

> AI-Generated Documentation | Scan Date: 2026-01-19

## System Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Management Cluster                                 │
│                                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐        │
│  │  Kamaji        │    │  TenantControl  │    │  TenantControl  │        │
│  │  Operator      │───▶│  Plane (TCP-1)  │    │  Plane (TCP-2)  │        │
│  │                │    │                 │    │                 │        │
│  │ - Controllers  │    │ ┌─────────────┐ │    │ ┌─────────────┐ │        │
│  │ - Webhooks     │    │ │ API Server  │ │    │ │ API Server  │ │        │
│  │ - SOOT Mgr     │    │ ├─────────────┤ │    │ ├─────────────┤ │        │
│  └────────┬───────┘    │ │ Controller  │ │    │ │ Controller  │ │        │
│           │            │ │ Manager     │ │    │ │ Manager     │ │        │
│           │            │ ├─────────────┤ │    │ ├─────────────┤ │        │
│           │            │ │ Scheduler   │ │    │ │ Scheduler   │ │        │
│           │            │ ├─────────────┤ │    │ ├─────────────┤ │        │
│           │            │ │ Kine (opt)  │ │    │ │ Kine (opt)  │ │        │
│           │            │ └─────────────┘ │    │ └─────────────┘ │        │
│           │            └────────┬────────┘    └────────┬────────┘        │
│           │                     │                      │                  │
│           ▼                     ▼                      ▼                  │
│  ┌─────────────────────────────────────────────────────────────────┐     │
│  │                        DataStore Layer                           │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │     │
│  │  │  etcd    │  │  MySQL   │  │ Postgres │  │   NATS   │        │     │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │     │
│  └─────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Konnectivity (optional)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Worker Nodes (Tenant Cluster)                      │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                      │
│  │   Worker 1   │  │   Worker 2   │  │   Worker N   │                      │
│  │              │  │              │  │              │                      │
│  │ - kubelet    │  │ - kubelet    │  │ - kubelet    │                      │
│  │ - kube-proxy │  │ - kube-proxy │  │ - kube-proxy │                      │
│  │ - CNI        │  │ - CNI        │  │ - CNI        │                      │
│  └──────────────┘  └──────────────┘  └──────────────┘                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Controller Architecture

### Controller Hierarchy

```
┌───────────────────────────────────────────────────────────────┐
│                    Admin Cluster Controllers                   │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  TenantControlPlaneReconciler (Main)                         │
│  ├── Manages: Secrets, ConfigMaps, Deployments, Services     │
│  ├── Owns: Ingress, HTTPRoute, GRPCRoute, TLSRoute          │
│  └── Watches: Jobs (migration), CertificateChan, TriggerChan │
│                                                               │
│  DataStoreController                                          │
│  ├── Tracks: TenantControlPlane usage                        │
│  └── Triggers: TCP reconciliation on DataStore changes       │
│                                                               │
│  CertificateLifecycleController                              │
│  ├── Monitors: X.509 certificates, Kubeconfig certs          │
│  └── Triggers: Rotation before expiration threshold          │
│                                                               │
│  KubeconfigGeneratorReconciler                               │
│  ├── Generates: Dynamic kubeconfigs                          │
│  └── Watches: Namespaces, TenantControlPlanes                │
│                                                               │
│  TelemetryController                                          │
│  └── Reports: Usage statistics to telemetry service          │
│                                                               │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                 SOOT Controllers (Per-TCP)                     │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  SOOT Manager                                                 │
│  ├── Creates: Isolated controller-runtime manager per TCP    │
│  └── Lifecycle: Start/Stop based on TCP state                │
│                                                               │
│  WritePermissions Controller                                  │
│  └── Manages: ValidatingWebhookConfiguration for R/O mode    │
│                                                               │
│  Migrate Controller                                           │
│  └── Manages: ValidatingWebhook for migration interception   │
│                                                               │
│  CoreDNS Controller                                           │
│  └── Manages: DNS addon (Deployment, RBAC, ConfigMap)        │
│                                                               │
│  KubeProxy Controller                                         │
│  └── Manages: kube-proxy addon (DaemonSet, RBAC)             │
│                                                               │
│  Konnectivity Controller                                      │
│  └── Manages: Network proxy agent (DaemonSet/Deployment)     │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

## Resource Pipeline

### TenantControlPlane Reconciliation Resources

The TenantControlPlane reconciler processes resources in a specific order:

```go
// From controllers/resources.go - Resource execution order

1.  DatastoreSetup           // Create DB, user, privileges
2.  DatastoreCertificate     // Store TLS certs for datastore
3.  DatastoreStorageConfig   // Generate connection secret
4.  CACertificate            // Root CA for cluster
5.  FrontProxyCACertificate  // Aggregation layer CA
6.  SACertificate            // Service account keypair
7.  APIServerCertificate     // API server TLS cert
8.  APIServerKubeletClient   // Kubelet client cert
9.  FrontProxyClientCert     // Aggregation client cert
10. AdminKubeconfig          // Admin kubeconfig
11. ControllerManagerKubeconfig
12. SchedulerKubeconfig
13. KubeadmConfig            // kubeadm configuration
14. KubeadmPhase             // Bootstrap phases
15. KubernetesDeployment     // Control plane deployment
16. KubernetesService        // Service exposure
17. KubernetesIngress        // Optional ingress
18. KubernetesGateway        // Optional gateway routes
19. KonnectivityResources    // If addon enabled
20. DatastoreMigrate         // If migration in progress
```

## DataStore Architecture

### Connection Abstraction

```go
// internal/datastore/datastore.go

type Connection interface {
    // Database operations
    CreateDB(ctx context.Context, name string) error
    DeleteDB(ctx context.Context, name string) error
    DBExists(ctx context.Context, name string) (bool, error)

    // User management
    CreateUser(ctx context.Context, user, password string) error
    DeleteUser(ctx context.Context, user string) error

    // Privilege management
    GrantPrivileges(ctx context.Context, user, db string) error
    RevokePrivileges(ctx context.Context, user, db string) error

    // Migration
    Migrate(ctx context.Context, tcp TenantControlPlane, target Connection) error

    // Connection management
    Close() error
}
```

### Driver Implementations

| Driver | Package | Storage Model | Key Features |
|--------|---------|---------------|--------------|
| **etcd** | `internal/datastore/etcd.go` | Key-Value with prefix | Native RBAC, key prefix isolation |
| **MySQL** | `internal/datastore/mysql.go` | Relational (via Kine) | Database per tenant, GRANT-based auth |
| **PostgreSQL** | `internal/datastore/postgresql.go` | Relational (via Kine) | Role-based access, schema per tenant |
| **NATS** | `internal/datastore/nats.go` | JetStream KeyValue | Bucket per tenant, simple isolation |

### Kine Integration

For non-etcd drivers, Kamaji uses [Kine](https://github.com/k3s-io/kine) as a shim:

```
┌─────────────────────────────────────────────────────────┐
│                    TCP Deployment Pod                    │
│                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │ API Server  │───▶│    Kine     │───▶│  Database   │ │
│  │             │    │ (sidecar)   │    │ (external)  │ │
│  │ etcd client │    │ etcd shim   │    │ MySQL/PG/   │ │
│  │ protocol    │    │             │    │ NATS        │ │
│  └─────────────┘    └─────────────┘    └─────────────┘ │
│        │                  ▲                            │
│        │    Unix Domain   │                            │
│        └──────Socket──────┘                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Webhook Architecture

### Admission Webhook Flow

```
                            Kubernetes API Server
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Kamaji Webhook Server (:9443)                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Mutating Webhooks                                              │
│  ├── /mutate-tcp-defaults     → Set defaults (replicas, DNS)   │
│  └── /mutate-tcp-telemetry    → Add telemetry annotations      │
│                                                                 │
│  Validating Webhooks                                            │
│  ├── /validate-tcp            → Name, version, datastore, CIDR │
│  ├── /validate-tcp-gateway    → Gateway API configuration      │
│  ├── /validate-tcp-freeze     → Migration state protection     │
│  ├── /validate-tcp-write      → Write permission enforcement   │
│  ├── /validate-datastore      → DataStore configuration        │
│  └── /validate-ds-secrets     → Secret reference validation    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Handler Chain Pattern

```go
// internal/webhook/routes/route.go

type Route interface {
    GetPath() string
    GetObject() runtime.Object
}

// Handlers are chained per route
map[routes.Route][]handlers.Handler{
    routes.TenantControlPlaneValidate{}: {
        handlers.TenantControlPlaneCertSANs{},
        handlers.TenantControlPlaneName{},
        handlers.TenantControlPlaneVersion{},
        handlers.TenantControlPlaneDataStore{},
        handlers.TenantControlPlaneDeployment{},
        handlers.TenantControlPlaneServiceCIDR{},
        handlers.TenantControlPlaneLoadBalancerSourceRanges{},
        handlers.TenantControlPlaneGatewayValidation{},
    },
}
```

## Certificate Management

### Certificate Hierarchy

```
                    ┌─────────────────────┐
                    │   Root CA (10yr)    │
                    │   ca.crt / ca.key   │
                    └─────────┬───────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ API Server    │   │ Kubelet       │   │ Controller    │
│ Certificate   │   │ Client Cert   │   │ Manager Cert  │
└───────────────┘   └───────────────┘   └───────────────┘

                    ┌─────────────────────┐
                    │ Front Proxy CA      │
                    │ (Aggregation)       │
                    └─────────┬───────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ Front Proxy       │
                    │ Client Cert       │
                    └───────────────────┘

                    ┌─────────────────────┐
                    │ Service Account    │
                    │ Keypair (RSA)      │
                    └─────────────────────┘
```

### Certificate Lifecycle

```
┌──────────────────────────────────────────────────────────────┐
│              CertificateLifecycleController                   │
│                                                              │
│  1. Watch Secrets with certificate labels                    │
│  2. Extract certificate from Secret                          │
│  3. Check: (now + threshold) > cert.NotAfter ?               │
│     ├── Yes: Trigger TCP reconciliation → Regenerate         │
│     └── No:  Requeue after (NotAfter - threshold)            │
│                                                              │
│  Default threshold: 24 hours before expiration               │
│  Certificate validity: 10 years                              │
└──────────────────────────────────────────────────────────────┘
```

## SOOT (Supervisor Of Operator Tenants)

SOOT runs controllers inside each tenant cluster to manage in-cluster resources.

### SOOT Manager Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                     SOOT Manager                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  TCP Status Check                                           │
│  │                                                          │
│  ├─► Provisioning/NotReady → Skip (not ready)              │
│  │                                                          │
│  ├─► Deleted/Sleeping → Cleanup manager, remove finalizer  │
│  │                                                          │
│  └─► Ready → Start/Maintain manager                        │
│              │                                              │
│              ├─► Create isolated Manager                    │
│              ├─► Register sub-controllers                   │
│              ├─► Start in goroutine                         │
│              └─► Store reference in map[UID]*Manager        │
│                                                             │
│  Cleanup Triggers:                                          │
│  - CA rotation detected                                     │
│  - Manager failure (annotation set)                         │
│  - TCP enters not-ready state                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### In-Tenant Resources Managed

| Controller | Resources Created | Purpose |
|------------|-------------------|---------|
| **CoreDNS** | Deployment, Service, ConfigMap, ClusterRole, ClusterRoleBinding, ServiceAccount | Cluster DNS |
| **KubeProxy** | DaemonSet, ConfigMap, ClusterRole, Role, RoleBinding, ServiceAccount | Network proxying |
| **Konnectivity** | DaemonSet/Deployment, ServiceAccount, ClusterRoleBinding | Network tunnel |
| **WritePermissions** | ValidatingWebhookConfiguration | Read-only mode |
| **Migrate** | ValidatingWebhookConfiguration | Migration protection |

## Deployment Architecture

### Control Plane Pod Structure

```yaml
# Simplified TCP Deployment Pod Spec

spec:
  containers:
    - name: kube-apiserver
      image: registry.k8s.io/kube-apiserver:v1.x.x
      ports:
        - containerPort: 6443
      volumeMounts:
        - name: pki
          mountPath: /etc/kubernetes/pki

    - name: kube-controller-manager
      image: registry.k8s.io/kube-controller-manager:v1.x.x
      volumeMounts:
        - name: pki
          mountPath: /etc/kubernetes/pki

    - name: kube-scheduler
      image: registry.k8s.io/kube-scheduler:v1.x.x
      volumeMounts:
        - name: pki
          mountPath: /etc/kubernetes/pki

    - name: kine  # Only for MySQL/PostgreSQL/NATS
      image: rancher/kine:v0.11.10
      env:
        - name: DB_CONNECTION_STRING
          valueFrom: secretKeyRef
      volumeMounts:
        - name: kine-socket
          mountPath: /run/kine

  volumes:
    - name: pki
      projected:
        sources:
          - secret: tcp-name-ca
          - secret: tcp-name-api-server
          - secret: tcp-name-sa
          # ... more certificates
    - name: kine-socket
      emptyDir: {}
```

### Service Exposure Options

```
┌─────────────────────────────────────────────────────────────┐
│                   Service Exposure Options                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. ClusterIP (Internal only)                               │
│     └─► Access via port-forward or in-cluster clients       │
│                                                             │
│  2. NodePort                                                │
│     └─► Access via any node IP + allocated port             │
│                                                             │
│  3. LoadBalancer                                            │
│     └─► Cloud provider provisions external IP               │
│     └─► Supports LoadBalancerClass selection                │
│     └─► Supports source IP ranges restriction               │
│                                                             │
│  4. Ingress (Optional, with LoadBalancer/NodePort)          │
│     └─► TLS termination at ingress controller               │
│     └─► Host-based routing                                  │
│                                                             │
│  5. Gateway API (Optional, modern alternative)              │
│     └─► HTTPRoute for HTTP/HTTPS                            │
│     └─► GRPCRoute for gRPC                                  │
│     └─► TLSRoute for TLS passthrough                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Konnectivity Architecture

For scenarios where worker nodes are in a different network:

```
┌─────────────────────────────────────────────────────────────┐
│                    Management Cluster                        │
│                                                             │
│  ┌─────────────────────────────────────┐                   │
│  │       TCP Deployment Pod             │                   │
│  │                                      │                   │
│  │  ┌──────────────────────────────┐   │                   │
│  │  │     Konnectivity Server      │   │                   │
│  │  │     (proxy-server)           │◄──┼───────────┐       │
│  │  │     Port: 8132               │   │           │       │
│  │  └──────────────────────────────┘   │           │       │
│  │              ▲                       │           │       │
│  │              │ Egress Selector       │           │       │
│  │  ┌──────────┴───────────────────┐   │           │       │
│  │  │       API Server             │   │           │       │
│  │  └──────────────────────────────┘   │           │       │
│  └─────────────────────────────────────┘           │       │
│                                                     │       │
└─────────────────────────────────────────────────────┼───────┘
                                                      │
                          Konnectivity Tunnel         │
                          (gRPC over TLS)             │
                                                      │
┌─────────────────────────────────────────────────────┼───────┐
│                     Worker Nodes                     │       │
│                                                      │       │
│  ┌──────────────────────────────────────────────────┼──┐   │
│  │              Konnectivity Agent                   │  │   │
│  │              (DaemonSet/Deployment)               │  │   │
│  │                                                   ▼  │   │
│  │  Connects to: konnectivity-server.tcp-ns:8132       │   │
│  │  Proxies: kubelet, node-to-control-plane traffic    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Indexers and Caching

Kamaji uses field indexers for efficient resource lookups:

| Indexer | Field | Purpose |
|---------|-------|---------|
| `TenantControlPlaneStatusDataStore` | `status.storage.dataStoreName` | Find TCPs by DataStore |
| `DatastoreUsedSecret` | `secretRef` | Find DataStores by Secret |
| `GatewayListener` | `spec.listeners.name` | Find Gateways by listener |

## Metrics and Observability

### Prometheus Metrics

All resource handlers emit reconciliation metrics:

```go
// internal/resources/metrics.go

var reconciliationDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name: "kamaji_resource_reconciliation_duration_seconds",
        Help: "Duration of resource reconciliation",
    },
    []string{"resource", "operation"},
)
```

### Health Endpoints

| Endpoint | Port | Purpose |
|----------|------|---------|
| `/healthz` | 8081 | Liveness probe |
| `/readyz` | 8081 | Readiness probe |
| `/metrics` | 8080 | Prometheus metrics |
