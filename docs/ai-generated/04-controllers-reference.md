# Kamaji Controllers Reference

> AI-Generated Documentation | Scan Date: 2026-01-19

## Overview

Kamaji uses the controller-runtime framework to implement the operator pattern. Controllers are divided into two categories:

1. **Admin Cluster Controllers** - Run in the Kamaji operator pod
2. **SOOT Controllers** - Run per-TenantControlPlane inside tenant clusters

---

## Admin Cluster Controllers

### TenantControlPlaneReconciler

**Location:** `controllers/tenantcontrolplane_controller.go`

The main controller responsible for managing TenantControlPlane resources.

#### Responsibilities

- Create and manage all TCP child resources
- Handle certificate generation and rotation
- Manage DataStore connections
- Orchestrate control plane deployment lifecycle
- Handle deletion with finalizers

#### Reconciliation Flow

```
1. Acquire mutex lock (prevents concurrent reconciliation)
2. Check pause annotation → skip if paused
3. Check deletion timestamp:
   - If deleting: cleanup resources, remove finalizer
4. Retrieve DataStore and create connection
5. For each resource in pipeline:
   - Handle resource (create/update)
   - Update TCP status
   - Requeue if requested
6. Log completion
```

#### RBAC Requirements

```yaml
- apiGroups: [kamaji.clastix.io]
  resources: [tenantcontrolplanes]
  verbs: [get, list, watch, create, update, patch, delete]
- apiGroups: [kamaji.clastix.io]
  resources: [tenantcontrolplanes/status, tenantcontrolplanes/finalizers]
  verbs: [get, update, patch]
- apiGroups: [""]
  resources: [secrets, configmaps, services]
  verbs: [get, list, watch, create, update, patch, delete]
- apiGroups: [apps]
  resources: [deployments]
  verbs: [get, list, watch, create, update, patch, delete]
- apiGroups: [networking.k8s.io]
  resources: [ingresses]
  verbs: [get, list, watch, create, update, patch, delete]
- apiGroups: [batch]
  resources: [jobs]
  verbs: [get, list, watch, create, delete]
- apiGroups: [gateway.networking.k8s.io]
  resources: [httproutes, grpcroutes, tlsroutes]
  verbs: [get, list, watch, create, update, patch, delete]
- apiGroups: [gateway.networking.k8s.io]
  resources: [gateways]
  verbs: [get, list, watch]
```

#### Owned Resources

- Secrets (certificates, kubeconfigs, datastore config)
- ConfigMaps (kubeadm config)
- Deployments (control plane)
- Services
- Ingresses
- HTTPRoutes, GRPCRoutes, TLSRoutes (Gateway API)

#### Configuration

| Field | Type | Description |
|-------|------|-------------|
| `DefaultDataStoreName` | string | Fallback DataStore name |
| `KineContainerImage` | string | Kine sidecar image |
| `TmpBaseDirectory` | string | Temporary file directory |
| `CertExpirationThreshold` | Duration | Certificate rotation threshold |
| `ReconcileTimeout` | Duration | Context timeout per reconcile |
| `MaxConcurrentReconciles` | int | Worker goroutines |

---

### DataStoreController

**Location:** `controllers/datastore_controller.go`

Manages DataStore resources and tracks their usage.

#### Responsibilities

- Update DataStore status with list of using TCPs
- Trigger TCP reconciliation when DataStore changes
- Track TCP creation/update/deletion

#### Reconciliation Flow

```
1. Get DataStore resource
2. Check pause annotation
3. List all TCPs using this DataStore (field selector)
4. Update DataStore.Status.UsedBy
5. Trigger reconciliation for all dependent TCPs
```

#### Watches

- **For:** DataStore (with GenerationChangedPredicate)
- **Watches:** TenantControlPlane (all events)

---

### CertificateLifecycleController

**Location:** `controllers/certificate_lifecycle_controller.go`

Monitors certificates for expiration and triggers rotation.

#### Responsibilities

- Watch certificate Secrets
- Check validity against threshold
- Trigger owner reconciliation before expiration

#### Reconciliation Flow

```
1. Get Secret containing certificate
2. Check pause annotation on owner
3. Determine certificate type (X509 or Kubeconfig)
4. Extract and parse certificate
5. Calculate: deadline = now + threshold
6. If deadline > cert.NotAfter:
   - Enqueue owner (TCP or KubeconfigGenerator)
7. Else:
   - Requeue after (NotAfter - deadline)
```

#### Label Selectors

Watches Secrets with labels:
- `kamaji.clastix.io/controller-resource: CertificateX509`
- `kamaji.clastix.io/controller-resource: CertificateKubeconfig`

---

### KubeconfigGeneratorReconciler

**Location:** `controllers/kubeconfiggenerator_controller.go`

Generates dynamic kubeconfigs based on selectors.

#### Responsibilities

- Match namespaces and TCPs by label selectors
- Generate X.509 certificates signed by TCP CA
- Create kubeconfig Secrets
- Track errors in status

#### Reconciliation Flow

```
1. Get KubeconfigGenerator resource
2. Check pause annotation
3. List matching namespaces
4. For each namespace, list matching TCPs
5. For each TCP:
   - Get admin kubeconfig template
   - Extract user/groups (static or JSON path)
   - Generate certificate signed by TCP CA
   - Create/update Secret with kubeconfig
6. Update status with counts and errors
```

---

### KubeconfigGeneratorWatcher

**Location:** `controllers/kubeconfiggenerator_watcher.go`

Watches TCP changes to trigger KubeconfigGenerator reconciliation.

#### Responsibilities

- Watch TenantControlPlane updates
- Evaluate all KubeconfigGenerators against changed TCP
- Route matching generators to reconciler

---

### TelemetryController

**Location:** `controllers/telemetry_controller.go`

Collects and reports usage statistics.

#### Responsibilities

- Count TCPs by state (Running, Sleeping, NotReady, Upgrading)
- Count DataStores by driver type
- Report to telemetry service every 5 minutes

#### Collected Metrics

| Metric | Description |
|--------|-------------|
| TCP states | Running, Sleeping, NotReady, Upgrading counts |
| DataStore types | etcd, MySQL, PostgreSQL, NATS counts |
| Kubernetes version | Management cluster version |
| Kamaji version | Operator version |

---

## SOOT Controllers

SOOT (Supervisor Of Operator Tenants) runs controllers inside each tenant cluster.

### SOOT Manager

**Location:** `controllers/soot/manager.go`

Orchestrates in-tenant controllers per TenantControlPlane.

#### Responsibilities

- Create isolated controller-runtime Manager per TCP
- Register and start sub-controllers
- Handle cleanup on TCP deletion/sleeping
- Restart on CA rotation or failures

#### Lifecycle States

| State | Action |
|-------|--------|
| TCP Provisioning | Skip (not ready) |
| TCP Ready | Start/maintain Manager |
| TCP Sleeping | Cleanup Manager |
| TCP Deleted | Cleanup Manager, remove finalizer |
| CA Rotated | Restart Manager |
| Manager Failed | Cleanup and restart |

#### Sub-Controllers Registered

1. WritePermissions
2. Migrate
3. CoreDNS
4. KubeProxy
5. KonnectivityAgent
6. KubeadmPhase

---

### WritePermissions Controller

**Location:** `controllers/soot/controllers/write_permissions.go`

Manages read-only mode via ValidatingWebhookConfiguration.

#### Responsibilities

- Create/update ValidatingWebhookConfiguration
- Block CREATE/UPDATE/DELETE based on TCP.Spec.WritePermissions
- Exclude kube-system and kube-node-lease namespaces

#### Webhook Configuration

```yaml
webhooks:
  - name: write-permissions.kamaji.clastix.io
    rules:
      - operations: [CREATE, UPDATE, DELETE]  # Based on spec
        apiGroups: ["*"]
        apiVersions: ["*"]
        resources: ["*"]
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: [kube-system, kube-node-lease]
```

---

### Migrate Controller

**Location:** `controllers/soot/controllers/migrate.go`

Manages migration-related webhooks.

#### Responsibilities

- Create ValidatingWebhookConfiguration during migration
- Intercept lease operations in kube-node-lease
- Clean up after migration completes

---

### CoreDNS Controller

**Location:** `controllers/soot/controllers/coredns.go`

Manages CoreDNS addon in tenant cluster.

#### Resources Managed

| Resource | Name | Purpose |
|----------|------|---------|
| ClusterRole | system:coredns | DNS RBAC |
| ClusterRoleBinding | system:coredns | Bind to ServiceAccount |
| ServiceAccount | coredns | Pod identity |
| ConfigMap | coredns | Corefile configuration |
| Service | kube-dns | DNS service |
| Deployment | coredns | DNS pods |

---

### KubeProxy Controller

**Location:** `controllers/soot/controllers/kubeproxy.go`

Manages kube-proxy addon in tenant cluster.

#### Resources Managed

| Resource | Name | Purpose |
|----------|------|---------|
| ClusterRole | system:node-proxier | Proxy RBAC |
| ClusterRoleBinding | system:node-proxier | Bind to ServiceAccount |
| ServiceAccount | kube-proxy | Pod identity |
| Role | kube-proxy | Namespace-scoped RBAC |
| RoleBinding | kube-proxy | Bind Role |
| ConfigMap | kube-proxy | Proxy configuration |
| DaemonSet | kube-proxy | Proxy pods |

---

### Konnectivity Controller

**Location:** `controllers/soot/controllers/konnectivity.go`

Manages Konnectivity agent in tenant cluster.

#### Resources Managed

| Resource | Name | Purpose |
|----------|------|---------|
| ServiceAccount | konnectivity-agent | Pod identity |
| ClusterRoleBinding | konnectivity-agent | RBAC binding |
| DaemonSet/Deployment | konnectivity-agent | Agent pods |

#### Modes

- **DaemonSet:** Agent on every node (default)
- **Deployment:** Specified replica count

---

## Resource Pipeline

The TenantControlPlane reconciler processes resources in a specific order defined in `controllers/resources.go`:

```go
func GetResources(ctx context.Context, config GroupResourceBuilderConfiguration) []resources.Resource {
    return []resources.Resource{
        // DataStore setup
        &datastore.Setup{...},
        &datastore.Certificate{...},
        &datastore.StorageConfig{...},

        // Certificates
        &resources.CACertificate{...},
        &resources.FrontProxyCACertificate{...},
        &resources.SACertificate{...},
        &resources.APIServerCertificate{...},
        &resources.APIServerKubeletClientCertificate{...},
        &resources.FrontProxyClientCertificate{...},

        // Kubeconfigs
        &resources.KubeconfigResource{Name: "admin", ...},
        &resources.KubeconfigResource{Name: "controller-manager", ...},
        &resources.KubeconfigResource{Name: "scheduler", ...},

        // Kubeadm
        &resources.KubeadmConfigResource{...},
        &resources.KubeadmPhase{...},

        // Kubernetes resources
        &resources.KubernetesDeploymentResource{...},
        &resources.KubernetesServiceResource{...},
        &resources.KubernetesIngressResource{...},
        &resources.KubernetesGatewayResource{...},

        // Konnectivity (if enabled)
        &konnectivity.CertificateResource{...},
        &konnectivity.KubeconfigResource{...},
        &konnectivity.EgressSelectorConfigurationResource{...},
        &konnectivity.ServiceAccountResource{...},
        &konnectivity.ClusterRoleBindingResource{...},
        &konnectivity.ServiceResource{...},
        &konnectivity.DeploymentResource{...},
        &konnectivity.GatewayResource{...},

        // Migration
        &datastore.Migrate{...},
    }
}
```

---

## Event Channels

Controllers communicate via typed channels:

| Channel | Producer | Consumer | Purpose |
|---------|----------|----------|---------|
| `TriggerChan` | DataStoreController | TCPReconciler | Trigger TCP reconciliation |
| `CertificateChan` | CertificateLifecycleController | TCPReconciler, KCGReconciler | Trigger cert rotation |

---

## Finalizers

| Finalizer | Resource | Purpose |
|-----------|----------|---------|
| `kamaji.clastix.io/datastore` | TenantControlPlane | Ensure DataStore cleanup |
| `kamaji.clastix.io/soot` | TenantControlPlane | Ensure SOOT cleanup |

---

## Predicates

Controllers use predicates to filter watch events:

| Predicate | Usage | Description |
|-----------|-------|-------------|
| `GenerationChangedPredicate` | DataStore | Only on spec changes |
| `LabelSelectorPredicate` | Secrets | Filter by labels |
| Custom status predicate | SOOT Manager | Skip provisioning state |
