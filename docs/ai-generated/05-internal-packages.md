# Kamaji Internal Packages Reference

> AI-Generated Documentation | Scan Date: 2026-01-19

## Package Overview

```
internal/
├── builders/
│   └── controlplane/     # Deployment and Konnectivity builders
├── constants/            # Labels and annotations
├── crypto/               # Certificate utilities
├── datastore/            # Storage driver implementations
│   └── errors/           # Datastore-specific errors
├── errors/               # Common error types
├── kubeadm/              # Kubeadm integration
│   └── printers/         # Output printers
├── resources/            # Resource handlers
│   ├── addons/           # CoreDNS, KubeProxy
│   │   └── utils/        # Managed labels
│   ├── datastore/        # Datastore resources
│   ├── konnectivity/     # Konnectivity resources
│   └── utils/            # Resource utilities
├── upgrade/              # Version management
├── utilities/            # Common utilities
└── webhook/              # Admission webhooks
    ├── handlers/         # Webhook handlers
    └── routes/           # Webhook routing
```

---

## builders/controlplane

Constructs Kubernetes Deployment objects for control planes.

### Deployment Builder

**File:** `internal/builders/controlplane/deployment.go`

```go
type Deployment struct {
    Client             client.Client
    KineContainerImage string
}

func (d Deployment) Build(
    ctx context.Context,
    tcp *kamajiv1alpha1.TenantControlPlane,
    ds kamajiv1alpha1.DataStore,
    dso []DataStoreOverrides,
    address string,
) (*appsv1.Deployment, error)
```

#### Features

- Multi-container pod: kube-apiserver, kube-controller-manager, kube-scheduler
- Optional Kine sidecar for non-etcd datastores
- Volume management for PKI certificates
- Liveness, readiness, and startup probes
- Blue/green deployment strategy by default
- Extra arguments via annotations for change detection

### Konnectivity Builder

**File:** `internal/builders/controlplane/konnectivity_server.go`

Builds Konnectivity server container configuration.

---

## constants

Defines standard labels and annotations.

### Labels

**File:** `internal/constants/labels.go`

```go
const (
    ProjectNameLabelKey      = "kamaji.clastix.io/project"
    ProjectNameLabelValue    = "kamaji"
    ControlPlaneLabelKey     = "kamaji.clastix.io/name"
    ComponentLabelKey        = "kamaji.clastix.io/component"
)
```

### Annotations

**File:** `internal/constants/annotations.go`

```go
const (
    PausedAnnotation        = "kamaji.clastix.io/paused"
    KubeconfigSecretKeyAnno = "kamaji.clastix.io/kubeconfig-secret-key"
)
```

---

## crypto

Certificate generation and validation utilities.

**File:** `internal/crypto/crypto.go`

### Functions

```go
// Generate CA certificate and private key
func GenerateCACertificatePrivateKeyPair(commonName string) ([]byte, []byte, error)

// Generate certificate signed by CA
func GenerateCertificatePrivateKeyPair(
    caCert, caKey []byte,
    commonName string,
    organizations []string,
    dnsNames []string,
    ips []net.IP,
) ([]byte, []byte, error)

// Validate certificate-key pair
func CheckCertificateAndPrivateKeyPairValidity(certPEM, keyPEM []byte) (bool, error)

// Check certificate SANs
func CheckCertificateNamesAndIPs(
    certPEM []byte,
    dnsNames []string,
    ips []net.IP,
) (bool, error)

// Verify certificate chain
func VerifyCertificate(caCertPEM, certPEM []byte) error

// Create certificate template
func NewCertificateTemplate(
    commonName string,
    organizations []string,
    dnsNames []string,
    ips []net.IP,
    validity time.Duration,
) *x509.Certificate
```

### Constants

- Default validity: 10 years
- RSA key size: 2048 bits

---

## datastore

Multi-driver storage abstraction layer.

### Connection Interface

**File:** `internal/datastore/datastore.go`

```go
type Connection interface {
    // Database operations
    CreateDB(ctx context.Context, name string) error
    DeleteDB(ctx context.Context, name string) error
    DBExists(ctx context.Context, name string) (bool, error)

    // User management
    CreateUser(ctx context.Context, user, password string) error
    DeleteUser(ctx context.Context, user string) error

    // Privileges
    GrantPrivileges(ctx context.Context, user, db string) error
    RevokePrivileges(ctx context.Context, user, db string) error

    // Migration
    Migrate(ctx context.Context, tcp TenantControlPlane, target Connection) error

    // Lifecycle
    Close() error
}
```

### Driver Implementations

#### etcd Driver

**File:** `internal/datastore/etcd.go`

- Native etcd v3 client
- RBAC with users and roles
- Key prefix isolation (`/tenant-name/`)
- Certificate-based authentication

#### MySQL Driver

**File:** `internal/datastore/mysql.go`

- go-sql-driver/mysql client
- Database per tenant
- GRANT/REVOKE privileges
- TLS support

#### PostgreSQL Driver

**File:** `internal/datastore/postgresql.go`

- go-pg client
- Role-based access control
- Schema per tenant
- Connection pooling

#### NATS Driver

**File:** `internal/datastore/nats.go`

- JetStream KeyValue buckets
- Bucket per tenant
- Simplified isolation model

### Connection Configuration

**File:** `internal/datastore/connection.go`

```go
type ConnectionConfig struct {
    Endpoints   []string
    Username    string
    Password    string
    TLSConfig   *tls.Config
    Certificate []byte
    PrivateKey  []byte
    CA          []byte
}
```

---

## errors

Custom error types for controller logic.

**File:** `internal/errors/errors.go`

```go
// Migration in progress, requeue
type MigrationInProcessError struct{}

// LoadBalancer not yet assigned IP
type NonExposedLoadBalancerError struct{}

// Missing valid IP for endpoint
type MissingValidIPError struct{}

// Check if error should trigger requeue
func ShouldReconcileErrorBeIgnored(err error) bool
```

### Datastore Errors

**File:** `internal/datastore/errors/errors.go`

- User creation/deletion errors
- Database operation errors
- Privilege management errors
- Connection errors

---

## kubeadm

Integration with kubeadm for Kubernetes bootstrapping.

### Configuration

**File:** `internal/kubeadm/configuration.go`

```go
type Configuration struct {
    InitConfiguration *kubeadmapi.InitConfiguration
    Kubeconfig        *clientcmdapi.Config
    Parameters        Parameters
}

type Parameters struct {
    TenantControlPlaneName      string
    TenantControlPlaneNamespace string
    TenantControlPlaneAddress   string
    TenantControlPlanePort      int32
    // ... additional fields
}

// Generate kubeadm InitConfiguration
func CreateKubeadmInitConfiguration(params Parameters) (*kubeadmapi.InitConfiguration, error)
```

### Certificates

**File:** `internal/kubeadm/certificates.go`

```go
// Generate CA certificate and key
func GenerateCACertificatePrivateKeyPair(
    tmpDirectory string,
    clusterName string,
    certName string,
) (crt, key []byte, err error)

// Generate certificate signed by CA
func GenerateCertificatePrivateKeyPair(
    tmpDirectory string,
    ca kubeadmcerts.CertKeyPair,
    certName string,
    config certutil.Config,
) (crt, key []byte, err error)
```

### Kubeconfig

**File:** `internal/kubeadm/kubeconfig.go`

```go
// Generate kubeconfig for component
func CreateKubeconfig(
    clusterName string,
    endpoint string,
    caCert []byte,
    clientCert []byte,
    clientKey []byte,
) (*clientcmdapi.Config, error)
```

### Bootstrap Token

**File:** `internal/kubeadm/bootstraptoken.go`

```go
// Generate bootstrap token for node joining
func GenerateBootstrapToken() (*bootstraptokenv1.BootstrapToken, error)
```

---

## resources

Resource handlers for TenantControlPlane reconciliation.

### Base Interface

**File:** `internal/resources/resource.go`

```go
type Resource interface {
    // Get resource name
    GetName() string

    // Initialize resource
    Define(ctx context.Context, tcp *kamajiv1alpha1.TenantControlPlane) error

    // Create or update resource
    CreateOrUpdate(ctx context.Context, tcp *kamajiv1alpha1.TenantControlPlane) (controllerutil.OperationResult, error)

    // Cleanup resource
    CleanUp(ctx context.Context, tcp *kamajiv1alpha1.TenantControlPlane) (bool, error)

    // Check if status needs update
    ShouldStatusBeUpdated(tcp *kamajiv1alpha1.TenantControlPlane) bool

    // Update status
    UpdateTenantControlPlaneStatus(tcp *kamajiv1alpha1.TenantControlPlane) error
}
```

### Certificate Resources

| Resource | File | Purpose |
|----------|------|---------|
| CACertificate | `ca_certificate.go` | Root CA |
| FrontProxyCACertificate | `front_proxy_ca_certificate.go` | Aggregation CA |
| SACertificate | `sa_certificate.go` | Service account keypair |
| APIServerCertificate | `api_server_certificate.go` | API server TLS |
| APIServerKubeletClientCertificate | `api_server_kubelet_client_certificate.go` | Kubelet client |
| FrontProxyClientCertificate | `front-proxy-client-certificate.go` | Aggregation client |

### Kubernetes Resources

| Resource | File | Purpose |
|----------|------|---------|
| KubernetesDeploymentResource | `k8s_deployment_resource.go` | Control plane deployment |
| KubernetesServiceResource | `k8s_service_resource.go` | Service exposure |
| KubernetesIngressResource | `k8s_ingress_resource.go` | Ingress routing |
| KubernetesGatewayResource | `k8s_gateway_resource.go` | Gateway API routes |

### Kubeadm Resources

| Resource | File | Purpose |
|----------|------|---------|
| KubeadmConfigResource | `kubeadm_config.go` | kubeadm ConfigMap |
| KubeadmPhase | `kubeadm_phases.go` | Bootstrap phases |
| KubeadmUpgrade | `kubeadm_upgrade.go` | Version upgrades |

---

## resources/addons

In-cluster addon management.

### CoreDNS

**File:** `internal/resources/addons/coredns.go`

```go
type CoreDNS struct {
    Client client.Client
}

// Creates: Deployment, Service, ConfigMap, ClusterRole, ClusterRoleBinding, ServiceAccount
```

### KubeProxy

**File:** `internal/resources/addons/kube_proxy.go`

```go
type KubeProxy struct {
    Client client.Client
}

// Creates: DaemonSet, ConfigMap, ClusterRole, Role, RoleBinding, ServiceAccount
```

---

## resources/datastore

DataStore-related resources.

| Resource | File | Purpose |
|----------|------|---------|
| Setup | `datastore_setup.go` | Create DB, user, privileges |
| Certificate | `datastore_certificate.go` | Store TLS certs |
| StorageConfig | `datastore_storage_config.go` | Connection secret |
| Migrate | `datastore_migrate.go` | DataStore migration |
| Multitenancy | `datastore_multitenancy.go` | Tenant isolation |

---

## resources/konnectivity

Konnectivity addon resources.

| Resource | File | Purpose |
|----------|------|---------|
| Agent | `agent.go` | DaemonSet/Deployment |
| CertificateResource | `certificate_resource.go` | mTLS certs |
| ClusterRoleBindingResource | `cluster_role_binding_resource.go` | RBAC |
| DeploymentResource | `deployment_resource.go` | Server deployment |
| EgressSelectorConfigurationResource | `egress_selector_configuration_resource.go` | API server config |
| GatewayResource | `gateway_resource.go` | TLSRoute for server |
| KubeconfigResource | `kubeconfig_resource.go` | Agent kubeconfig |
| ServiceAccountResource | `service_account_resource.go` | Pod identity |
| ServiceResource | `service_resource.go` | Server service |

---

## upgrade

Version management utilities.

**File:** `internal/upgrade/kubeadm_version.go`

```go
// Parse Kubernetes version
func ParseVersion(version string) (*semver.Version, error)

// Get kubeadm version for Kubernetes version
func GetKubeadmVersion(k8sVersion string) string
```

---

## utilities

Common utility functions.

**File:** `internal/utilities/args.go`

```go
// Parse --flag=value format to map
func ArgsFromSliceToMap(args []string) map[string]string

// Convert map to sorted slice
func ArgsFromMapToSlice(args map[string]string) []string

// Remove flag from map
func ArgsRemoveFlag(args map[string]string, flag string)

// Add or update flag
func ArgsAddFlagValue(args map[string]string, flag, value string)
```

---

## webhook

Admission webhook infrastructure.

### Registration

**File:** `internal/webhook/register.go`

```go
func Register(
    mgr ctrl.Manager,
    routes map[routes.Route][]handlers.Handler,
) error
```

### Handler Interface

**File:** `internal/webhook/handlers/handler.go`

```go
type Handler interface {
    OnCreate(obj runtime.Object) AdmissionResponse
    OnDelete(obj runtime.Object) AdmissionResponse
    OnUpdate(newObject, prevObject runtime.Object) AdmissionResponse
}

type AdmissionResponse struct {
    Allowed  bool
    Message  string
    Patches  []jsonpatch.Operation
}
```

### Available Handlers

| Handler | File | Purpose |
|---------|------|---------|
| TenantControlPlaneDefaults | `tcp_defaults.go` | Set defaults |
| TenantControlPlaneName | `tcp_name.go` | Validate name |
| TenantControlPlaneVersion | `tcp_version.go` | Validate K8s version |
| TenantControlPlaneDataStore | `tcp_datastore.go` | Validate DataStore ref |
| TenantControlPlaneCertSANs | `tcp_certsans.go` | Validate SANs |
| TenantControlPlaneServiceCIDR | `tcp_service_cidr.go` | Validate CIDRs |
| TenantControlPlaneDeployment | `tcp_deployment.go` | Validate deployment |
| TenantControlPlaneGatewayValidation | `tcp_gateway_validate.go` | Validate Gateway |
| TenantControlPlaneLoadBalancerSourceRanges | `tcp_lb_src_ranges.go` | Validate LB ranges |
| TenantControlPlaneTelemetry | `tcp_telemetry.go` | Add telemetry |
| WritePermission | `write_permission.go` | Enforce write limits |
| Freeze | `freeze.go` | Migration protection |
| DataStoreValidation | `ds_validate.go` | Validate DataStore |
| DataStoreSecretValidation | `ds_secrets.go` | Validate secrets |

### Routes

**File:** `internal/webhook/routes/route.go`

```go
type Route interface {
    GetPath() string
    GetObject() runtime.Object
}

// Available routes
type TenantControlPlaneDefaults struct{}    // /mutate-tcp-defaults
type TenantControlPlaneValidate struct{}    // /validate-tcp
type TenantControlPlaneTelemetry struct{}   // /mutate-tcp-telemetry
type TenantControlPlaneMigrate struct{}     // /validate-tcp-migrate
type TenantControlPlaneWritePermission struct{} // /validate-tcp-write
type DataStoreValidate struct{}             // /validate-datastore
type DataStoreSecrets struct{}              // /validate-ds-secrets
```
