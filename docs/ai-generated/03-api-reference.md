# Kamaji API Reference

> AI-Generated Documentation | Scan Date: 2026-01-19

## Custom Resource Definitions

Kamaji extends Kubernetes with three CRDs in the `kamaji.clastix.io` API group.

---

## TenantControlPlane

**API Version:** `kamaji.clastix.io/v1alpha1`
**Kind:** `TenantControlPlane`
**Scope:** Namespaced
**Short Name:** `tcp`

### Description

TenantControlPlane represents a managed Kubernetes control plane running as Pods in the management cluster.

### Spec Fields

#### Root Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `dataStore` | string | No | (from flag) | Name of DataStore resource |
| `dataStoreSchema` | string | No | `{namespace}_{name}` | Database schema/key prefix (immutable) |
| `dataStoreUsername` | string | No | `{namespace}_{name}` | Database username (immutable) |
| `dataStoreOverrides` | []DataStoreOverride | No | - | Per-resource datastore overrides |
| `writePermissions` | Permissions | No | all allowed | Block create/update/delete operations |
| `controlPlane` | ControlPlane | Yes | - | Deployment and service configuration |
| `kubernetes` | KubernetesSpec | Yes | - | Kubernetes version and settings |
| `networkProfile` | NetworkProfileSpec | No | - | Network configuration |
| `addons` | AddonsSpec | No | - | CoreDNS, KubeProxy, Konnectivity |

#### KubernetesSpec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `version` | string | Yes | - | Kubernetes version (e.g., "1.29.0") |
| `kubelet` | KubeletSpec | No | - | Kubelet configuration |
| `admissionControllers` | []string | No | 25 defaults | Enabled admission controllers |

#### KubeletSpec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `configurationJSONPatches` | []JSONPatch | No | - | RFC 6902 patches for kubeadm config |
| `preferredAddressTypes` | []string | No | `[InternalIP, ExternalIP, Hostname]` | Node address type order |
| `cgroupfs` | string | No | - | Cgroup driver (deprecated) |

#### NetworkProfileSpec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `address` | string | No | - | Static API server address |
| `port` | int32 | No | 6443 | API server port |
| `clusterDomain` | string | No | `cluster.local` | DNS domain (immutable) |
| `serviceCidr` | string | No | `10.96.0.0/16` | Service network CIDR |
| `podCidr` | string | No | `10.244.0.0/16` | Pod network CIDR |
| `dnsServiceIPs` | []string | No | auto-computed | DNS service IPs |
| `certSANs` | []string | No | - | Extra certificate SANs |
| `loadBalancerSourceRanges` | []string | No | - | IP ranges for LoadBalancer |
| `loadBalancerClass` | string | No | - | LoadBalancer class (immutable) |
| `allowAddressAsExternalIP` | bool | No | false | Include address in ExternalIPs |

#### ControlPlane

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `deployment` | DeploymentSpec | No | - | Pod deployment settings |
| `service` | ServiceSpec | Yes | - | Service exposure settings |
| `ingress` | IngressSpec | No | - | Optional Ingress (mutually exclusive with gateway) |
| `gateway` | GatewaySpec | No | - | Optional Gateway API (mutually exclusive with ingress) |

#### DeploymentSpec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `replicas` | *int32 | No | 2 | Number of pod replicas |
| `registrySettings` | RegistrySettings | No | registry.k8s.io | Container registry config |
| `nodeSelector` | map[string]string | No | - | Pod node selector |
| `tolerations` | []Toleration | No | - | Pod tolerations |
| `affinity` | *Affinity | No | - | Pod affinity rules |
| `topologySpreadConstraints` | []TopologySpreadConstraint | No | - | Pod spread constraints |
| `strategy` | DeploymentStrategy | No | RollingUpdate (blue/green) | Update strategy |
| `runtimeClassName` | string | No | - | RuntimeClass name |
| `resources` | *ControlPlaneComponentsResources | No | - | CPU/memory per component |
| `extraArgs` | *ControlPlaneExtraArgs | No | - | Additional CLI arguments |
| `additionalMetadata` | AdditionalMetadata | No | - | Labels/annotations for Deployment |
| `podAdditionalMetadata` | AdditionalMetadata | No | - | Labels/annotations for Pod |
| `additionalInitContainers` | []Container | No | - | Extra init containers |
| `additionalContainers` | []Container | No | - | Extra sidecar containers |
| `additionalVolumes` | []Volume | No | - | Extra volumes |
| `additionalVolumeMounts` | *AdditionalVolumeMounts | No | - | Volume mounts per component |
| `serviceAccountName` | string | No | `default` | Service account |

#### ServiceSpec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `serviceType` | string | Yes | - | `ClusterIP`, `NodePort`, or `LoadBalancer` |
| `additionalMetadata` | AdditionalMetadata | No | - | Labels/annotations |
| `additionalPorts` | []AdditionalPort | No | - | Extra service ports |

#### AddonsSpec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `coreDNS` | *AddonSpec | No | - | CoreDNS configuration |
| `kubeProxy` | *AddonSpec | No | - | kube-proxy configuration |
| `konnectivity` | *KonnectivitySpec | No | - | Konnectivity tunnel |

#### KonnectivitySpec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `server` | KonnectivityServerSpec | No | - | Server (control plane) config |
| `agent` | KonnectivityAgentSpec | No | - | Agent (worker node) config |

#### KonnectivityServerSpec

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `port` | int32 | - | Server listening port |
| `image` | string | `registry.k8s.io/kas-network-proxy/proxy-server` | Container image |
| `version` | string | auto-detected | Container version |
| `resources` | *ResourceRequirements | - | CPU/memory limits |
| `extraArgs` | []string | - | Additional arguments |

#### KonnectivityAgentSpec

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `image` | string | `registry.k8s.io/kas-network-proxy/proxy-agent` | Container image |
| `version` | string | auto-detected | Container version |
| `mode` | string | `DaemonSet` | `DaemonSet` or `Deployment` |
| `replicas` | *int32 | - | Replicas (Deployment mode only) |
| `tolerations` | []Toleration | `[CriticalAddonsOnly]` | Node tolerations |
| `hostNetwork` | bool | false | Use host network |
| `extraArgs` | []string | - | Additional arguments |

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `storage` | StorageStatus | DataStore connection status |
| `certificates` | CertificatesStatus | All certificate details |
| `kubeconfig` | KubeconfigStatus | Admin/manager/scheduler kubeconfigs |
| `kubernetes` | KubernetesStatus | Deployment, Service, version status |
| `kubeadmConfig` | KubeadmConfigStatus | kubeadm configuration |
| `kubeadmPhase` | KubeadmPhaseStatus | Bootstrap phase status |
| `controlPlaneEndpoint` | string | Exposed endpoint (host:port) |
| `addons` | AddonsStatus | Addon status |

### Example

```yaml
apiVersion: kamaji.clastix.io/v1alpha1
kind: TenantControlPlane
metadata:
  name: tenant-1
  namespace: tenants
spec:
  dataStore: default-etcd
  kubernetes:
    version: "1.29.0"
    kubelet:
      preferredAddressTypes:
        - InternalIP
        - ExternalIP
    admissionControllers:
      - CertificateApproval
      - CertificateSigning
      - DefaultStorageClass
      - MutatingAdmissionWebhook
      - ValidatingAdmissionWebhook
  networkProfile:
    port: 6443
    serviceCidr: "10.96.0.0/16"
    podCidr: "10.244.0.0/16"
    certSANs:
      - "tenant-1.example.com"
  controlPlane:
    deployment:
      replicas: 3
      resources:
        apiServer:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2"
            memory: "2Gi"
    service:
      serviceType: LoadBalancer
  addons:
    coreDNS: {}
    kubeProxy: {}
    konnectivity:
      server:
        port: 8132
      agent:
        mode: DaemonSet
```

---

## DataStore

**API Version:** `kamaji.clastix.io/v1alpha1`
**Kind:** `DataStore`
**Scope:** Cluster

### Description

DataStore defines a backing storage for TenantControlPlane state data.

### Spec Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `driver` | string | Yes | - | `etcd`, `MySQL`, `PostgreSQL`, or `NATS` (immutable) |
| `endpoints` | []string | Yes | - | Connection endpoints (min 1) |
| `basicAuth` | *BasicAuth | Conditional | - | Username/password (required for non-etcd) |
| `tlsConfig` | *TLSConfig | Conditional | - | TLS configuration |

#### TLSConfig

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `certificateAuthority` | CertKeyPair | Yes | CA certificate and private key |
| `clientCertificate` | *ClientCertificate | etcd only | Client mTLS certificate |

#### BasicAuth

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `username` | ContentRef | Yes | Username (inline or Secret) |
| `password` | ContentRef | Yes | Password (inline or Secret) |

#### ContentRef

| Field | Type | Description |
|-------|------|-------------|
| `content` | []byte | Base64-encoded inline content (takes precedence) |
| `secretReference` | *SecretReference | Reference to Secret |

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `usedBy` | []string | List of TenantControlPlanes using this DataStore |

### Validation Rules

**For etcd driver:**
- `tlsConfig.certificateAuthority.privateKey` must have content or secretReference
- `tlsConfig.clientCertificate.certificate` must have content or secretReference
- `tlsConfig.clientCertificate.privateKey` must have content or secretReference

**For non-etcd drivers:**
- Either `tlsConfig` or `basicAuth` must be provided
- If `basicAuth`, both username and password must have content or secretReference

### Examples

#### etcd DataStore

```yaml
apiVersion: kamaji.clastix.io/v1alpha1
kind: DataStore
metadata:
  name: etcd-default
spec:
  driver: etcd
  endpoints:
    - etcd-0.etcd.kamaji-system:2379
    - etcd-1.etcd.kamaji-system:2379
    - etcd-2.etcd.kamaji-system:2379
  tlsConfig:
    certificateAuthority:
      certificate:
        secretReference:
          name: etcd-certs
          namespace: kamaji-system
          keyPath: ca.crt
      privateKey:
        secretReference:
          name: etcd-certs
          namespace: kamaji-system
          keyPath: ca.key
    clientCertificate:
      certificate:
        secretReference:
          name: etcd-certs
          namespace: kamaji-system
          keyPath: tls.crt
      privateKey:
        secretReference:
          name: etcd-certs
          namespace: kamaji-system
          keyPath: tls.key
```

#### PostgreSQL DataStore

```yaml
apiVersion: kamaji.clastix.io/v1alpha1
kind: DataStore
metadata:
  name: postgres-default
spec:
  driver: PostgreSQL
  endpoints:
    - postgres.database.svc:5432
  basicAuth:
    username:
      secretReference:
        name: postgres-credentials
        namespace: kamaji-system
        keyPath: username
    password:
      secretReference:
        name: postgres-credentials
        namespace: kamaji-system
        keyPath: password
  tlsConfig:
    certificateAuthority:
      certificate:
        secretReference:
          name: postgres-ca
          namespace: kamaji-system
          keyPath: ca.crt
```

#### MySQL DataStore

```yaml
apiVersion: kamaji.clastix.io/v1alpha1
kind: DataStore
metadata:
  name: mysql-default
spec:
  driver: MySQL
  endpoints:
    - mysql.database.svc:3306
  basicAuth:
    username:
      content: cm9vdA==  # base64 encoded "root"
    password:
      secretReference:
        name: mysql-credentials
        namespace: kamaji-system
        keyPath: password
```

---

## KubeconfigGenerator

**API Version:** `kamaji.clastix.io/v1alpha1`
**Kind:** `KubeconfigGenerator`
**Scope:** Cluster
**Short Name:** `kc`

### Description

KubeconfigGenerator creates kubeconfig secrets for selected TenantControlPlanes with customizable user/group settings.

### Spec Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `namespaceSelector` | *LabelSelector | No | - | Filter namespaces by labels |
| `tenantControlPlaneSelector` | *LabelSelector | No | - | Filter TCPs by labels |
| `groups` | []CompoundValue | No | - | X.509 organization names |
| `user` | CompoundValue | No | - | X.509 common name |
| `controlPlaneEndpointFrom` | string | No | `admin.svc` | Endpoint key from TCP |

#### CompoundValue

Either `stringValue` or `fromDefinition` must be set, not both.

| Field | Type | Description |
|-------|------|-------------|
| `stringValue` | string | Static string value |
| `fromDefinition` | string | JSON path from TCP (e.g., `metadata.name`) |

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `resources` | int | Total targeted TCPs |
| `availableResources` | int | Successfully generated |
| `errors` | []ErrorStatus | Generation errors |

### Example

```yaml
apiVersion: kamaji.clastix.io/v1alpha1
kind: KubeconfigGenerator
metadata:
  name: developer-access
spec:
  namespaceSelector:
    matchLabels:
      environment: production
  tenantControlPlaneSelector:
    matchLabels:
      tier: standard
  user:
    fromDefinition: "metadata.name"
  groups:
    - stringValue: "developers"
    - fromDefinition: "metadata.namespace"
  controlPlaneEndpointFrom: "admin.svc"
```

---

## Enums and Constants

### ServiceType

| Value | Description |
|-------|-------------|
| `ClusterIP` | Internal cluster access only |
| `NodePort` | Expose on node ports |
| `LoadBalancer` | Cloud provider load balancer |

### Driver

| Value | Description |
|-------|-------------|
| `etcd` | Native etcd storage |
| `MySQL` | MySQL via Kine |
| `PostgreSQL` | PostgreSQL via Kine |
| `NATS` | NATS JetStream via Kine |

### KubeletPreferredAddressType

| Value | Description |
|-------|-------------|
| `Hostname` | Node hostname |
| `InternalIP` | Internal IP address |
| `ExternalIP` | External IP address |
| `InternalDNS` | Internal DNS name |
| `ExternalDNS` | External DNS name |

### KonnectivityAgentMode

| Value | Description |
|-------|-------------|
| `DaemonSet` | Run agent on all nodes |
| `Deployment` | Run specified replicas |

### VersionStatus

| Value | Description |
|-------|-------------|
| `Provisioning` | Initial setup |
| `CertificateAuthorityRotating` | CA rotation in progress |
| `Upgrading` | Version upgrade |
| `Migrating` | DataStore migration |
| `Ready` | Fully operational |
| `NotReady` | Not healthy |
| `Sleeping` | Scaled to zero |
| `WriteLimited` | Read-only mode |

---

## Labels and Annotations

### Standard Labels

| Label | Description |
|-------|-------------|
| `kamaji.clastix.io/project=kamaji` | Project identifier |
| `kamaji.clastix.io/name={tcp-name}` | TenantControlPlane name |
| `kamaji.clastix.io/component={type}` | Component type |
| `kamaji.clastix.io/managed-by=kamaji` | Management marker |

### Annotations

| Annotation | Description |
|------------|-------------|
| `kamaji.clastix.io/paused=true` | Pause reconciliation |
| `kamaji.clastix.io/kubeconfig-secret-key` | Kubeconfig secret key |
