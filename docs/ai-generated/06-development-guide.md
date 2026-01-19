# Kamaji Development Guide

> AI-Generated Documentation | Scan Date: 2026-01-19

## Prerequisites

- Go 1.25+
- Docker or Podman
- kubectl
- kind (for local development)
- Helm 3.x

## Getting Started

### Clone Repository

```bash
git clone https://github.com/clastix/kamaji.git
cd kamaji
```

### Install Dependencies

```bash
# Install Go dependencies
go mod download

# Install development tools
make controller-gen
make golangci-lint
make ginkgo
make kind
make helm
make ko
```

## Development Workflow

### Generate Code

```bash
# Generate DeepCopy methods
make generate

# Generate CRDs, RBAC, and webhooks
make manifests

# Generate API documentation
make apidoc
```

### Run Linting

```bash
make golint
```

### Run Tests

```bash
# Unit tests
make test

# E2E tests (creates kind cluster)
make e2e
```

### Build Container Image

```bash
# Build with ko (local)
make build

# Build and push
KO_PUSH=true KO_LOCAL=false make build
```

## Local Development Environment

### Create Kind Cluster

```bash
make env
```

### Install Dependencies

```bash
# Install cert-manager
make cert-manager

# Install MetalLB (for LoadBalancer services)
make metallb

# Install Gateway API CRDs
make gateway-api

# Install Envoy Gateway
make envoy-gateway
```

### Deploy DataStores

```bash
# All datastores
make datastores

# Individual datastores
make datastore-etcd
make datastore-mysql
make datastore-postgres
make datastore-nats
```

### Install Kamaji

```bash
# Install CRDs
helm upgrade --install kamaji-crds ./charts/kamaji-crds \
  --create-namespace --namespace kamaji-system

# Build dependencies
helm dependency build ./charts/kamaji

# Install Kamaji
helm upgrade --install kamaji ./charts/kamaji \
  --namespace kamaji-system \
  --set "image.tag=$(git describe --abbrev=0 --tags)" \
  --set "image.pullPolicy=Never"
```

### Run Locally (Outside Cluster)

```bash
make run
```

## Project Structure

```
.
├── api/v1alpha1/           # CRD type definitions
├── cmd/                    # CLI entry points
│   ├── manager/            # Main operator
│   ├── migrate/            # Migration tool
│   └── kubeconfig-generator/
├── controllers/            # Controller implementations
│   └── soot/               # In-tenant controllers
├── internal/               # Private packages
│   ├── builders/           # Object builders
│   ├── datastore/          # Storage drivers
│   ├── resources/          # Resource handlers
│   └── webhook/            # Admission handlers
├── charts/                 # Helm charts
├── config/                 # Kustomize manifests
├── deploy/                 # Deployment helpers
├── e2e/                    # E2E test suite
├── hack/                   # Development scripts
└── docs/                   # Documentation
```

## Adding a New Feature

### 1. Update API Types

Edit files in `api/v1alpha1/`:

```go
// api/v1alpha1/tenantcontrolplane_types.go

type TenantControlPlaneSpec struct {
    // Add your new field
    NewFeature *NewFeatureSpec `json:"newFeature,omitempty"`
}

type NewFeatureSpec struct {
    Enabled bool   `json:"enabled,omitempty"`
    Config  string `json:"config,omitempty"`
}
```

### 2. Regenerate Code

```bash
make generate
make manifests
```

### 3. Add Resource Handler

Create `internal/resources/new_feature.go`:

```go
package resources

type NewFeatureResource struct {
    Client client.Client
    // fields
}

func (r *NewFeatureResource) GetName() string {
    return "NewFeature"
}

func (r *NewFeatureResource) Define(
    ctx context.Context,
    tcp *kamajiv1alpha1.TenantControlPlane,
) error {
    // Initialize
    return nil
}

func (r *NewFeatureResource) CreateOrUpdate(
    ctx context.Context,
    tcp *kamajiv1alpha1.TenantControlPlane,
) (controllerutil.OperationResult, error) {
    // Create or update logic
    return controllerutil.OperationResultNone, nil
}

// Implement remaining interface methods...
```

### 4. Register Resource

Edit `controllers/resources.go`:

```go
func GetResources(ctx context.Context, config GroupResourceBuilderConfiguration) []resources.Resource {
    result := []resources.Resource{
        // Existing resources...

        // Add your resource
        &resources.NewFeatureResource{
            Client: config.client,
        },
    }
    return result
}
```

### 5. Add Tests

Create `internal/resources/new_feature_test.go`:

```go
package resources_test

import (
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
)

var _ = Describe("NewFeatureResource", func() {
    Context("when creating", func() {
        It("should create the resource", func() {
            // Test logic
        })
    })
})
```

### 6. Update Documentation

```bash
make apidoc
```

## Adding a New Webhook Handler

### 1. Create Handler

Create `internal/webhook/handlers/new_handler.go`:

```go
package handlers

type NewHandler struct {
    Client client.Client
}

func (h NewHandler) OnCreate(obj runtime.Object) AdmissionResponse {
    tcp := obj.(*kamajiv1alpha1.TenantControlPlane)

    // Validation logic
    if !valid {
        return AdmissionResponse{
            Allowed: false,
            Message: "validation failed",
        }
    }

    return AdmissionResponse{Allowed: true}
}

func (h NewHandler) OnUpdate(newObj, oldObj runtime.Object) AdmissionResponse {
    // Update validation
    return AdmissionResponse{Allowed: true}
}

func (h NewHandler) OnDelete(obj runtime.Object) AdmissionResponse {
    return AdmissionResponse{Allowed: true}
}
```

### 2. Register Handler

Edit `cmd/manager/cmd.go`:

```go
err = webhook.Register(mgr, map[routes.Route][]handlers.Handler{
    routes.TenantControlPlaneValidate{}: {
        // Existing handlers...
        handlers.NewHandler{Client: mgr.GetClient()},
    },
})
```

## Adding a New DataStore Driver

### 1. Implement Connection Interface

Create `internal/datastore/newdriver.go`:

```go
package datastore

type newDriverConnection struct {
    client *newdriver.Client
}

func newNewDriverConnection(config ConnectionConfig) (*newDriverConnection, error) {
    // Initialize client
    return &newDriverConnection{client: client}, nil
}

func (c *newDriverConnection) CreateDB(ctx context.Context, name string) error {
    // Implementation
}

func (c *newDriverConnection) DeleteDB(ctx context.Context, name string) error {
    // Implementation
}

// Implement remaining interface methods...
```

### 2. Update Factory

Edit `internal/datastore/connection.go`:

```go
func NewStorageConnection(
    ctx context.Context,
    client client.Client,
    dataStore kamajiv1alpha1.DataStore,
) (Connection, error) {
    switch dataStore.Spec.Driver {
    case kamajiv1alpha1.EtcdDriver:
        return newEtcdConnection(config)
    case kamajiv1alpha1.KineMySQLDriver:
        return newMySQLConnection(config)
    // Add new driver
    case kamajiv1alpha1.NewDriver:
        return newNewDriverConnection(config)
    }
}
```

### 3. Update API Types

Edit `api/v1alpha1/datastore_types.go`:

```go
var (
    EtcdDriver    Driver = "etcd"
    KineMySQLDriver Driver = "MySQL"
    // Add new driver
    NewDriver     Driver = "NewDriver"
)
```

## Testing

### Unit Tests

```bash
# Run all unit tests
make test

# Run specific package tests
go test ./internal/resources/... -v

# Run with coverage
go test ./... -coverprofile=cover.out
go tool cover -html=cover.out
```

### E2E Tests

```bash
# Full E2E suite
make e2e

# Run specific test
./bin/ginkgo -v -focus="should create" ./e2e
```

### Test Utilities

```go
import (
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    "sigs.k8s.io/controller-runtime/pkg/envtest"
)

var _ = BeforeSuite(func() {
    testEnv = &envtest.Environment{
        CRDDirectoryPaths: []string{
            filepath.Join("..", "charts", "kamaji", "crds"),
        },
    }
    cfg, err := testEnv.Start()
    Expect(err).NotTo(HaveOccurred())
})
```

## Debugging

### Enable Verbose Logging

```bash
# Run with debug logging
go run ./main.go manager --zap-log-level=debug

# Or set via environment
export ZAP_LOG_LEVEL=debug
```

### Debug Controller Reconciliation

Add logging in controller:

```go
log := log.FromContext(ctx)
log.V(1).Info("reconciling resource", "name", resource.GetName())
```

### Inspect Resources

```bash
# Get TCPs
kubectl get tcp -A

# Describe TCP
kubectl describe tcp tenant-1 -n tenants

# Get events
kubectl get events -n tenants --field-selector involvedObject.name=tenant-1

# Check operator logs
kubectl logs -n kamaji-system deploy/kamaji -f
```

## Release Process

### 1. Update Version

```bash
# Tag release
git tag v1.0.0
git push origin v1.0.0
```

### 2. Build and Push

```bash
VERSION=v1.0.0 KO_PUSH=true make build
```

### 3. Update Helm Chart

Edit `charts/kamaji/Chart.yaml`:

```yaml
version: 1.0.0
appVersion: v1.0.0
```

### 4. Generate Release Notes

GitHub Actions automatically generates release notes from commit messages.

## Makefile Targets

| Target | Description |
|--------|-------------|
| `make help` | Show all targets |
| `make generate` | Generate DeepCopy methods |
| `make manifests` | Generate CRDs, RBAC, webhooks |
| `make build` | Build container image |
| `make test` | Run unit tests |
| `make e2e` | Run E2E tests |
| `make golint` | Run linter |
| `make apidoc` | Generate API docs |
| `make env` | Create kind cluster |
| `make cleanup` | Delete kind cluster |
| `make run` | Run controller locally |

## Code Style

### Go Style

- Follow [Effective Go](https://golang.org/doc/effective_go)
- Use `gofmt` and `goimports`
- Run `make golint` before committing

### Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add new datastore driver
fix: handle nil pointer in webhook
docs: update API reference
test: add unit tests for migration
refactor: simplify certificate rotation
```

### License Headers

All Go files must have:

```go
// Copyright 2022 Clastix Labs
// SPDX-License-Identifier: Apache-2.0
```

## Troubleshooting

### Common Issues

**CRD validation errors:**
```bash
make manifests
kubectl apply -f charts/kamaji/crds/
```

**Webhook certificate issues:**
```bash
kubectl delete secret kamaji-serving-cert -n kamaji-system
kubectl rollout restart deploy/kamaji -n kamaji-system
```

**DataStore connection failures:**
```bash
# Check DataStore status
kubectl get datastore

# Check operator logs
kubectl logs -n kamaji-system deploy/kamaji | grep -i datastore
```

**TCP stuck in Provisioning:**
```bash
# Check events
kubectl describe tcp <name> -n <namespace>

# Check deployment
kubectl get deploy -l kamaji.clastix.io/name=<tcp-name> -n <namespace>
```
