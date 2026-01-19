# Kamaji DataStore Guide

> AI-Generated Documentation | Scan Date: 2026-01-19

## Overview

Kamaji supports multiple storage backends for TenantControlPlane state data. The storage layer is abstracted through a unified `Connection` interface, allowing seamless switching between drivers.

## Supported Drivers

| Driver | Backend | Protocol | Multi-Tenancy |
|--------|---------|----------|---------------|
| **etcd** | Native etcd | gRPC | Key prefix + RBAC |
| **MySQL** | MySQL/MariaDB | SQL (via Kine) | Database per tenant |
| **PostgreSQL** | PostgreSQL | SQL (via Kine) | Schema per tenant |
| **NATS** | NATS JetStream | NATS (via Kine) | Bucket per tenant |

## Architecture

### Native etcd

```
┌─────────────────────────┐
│   TenantControlPlane    │
│        Pod              │
│                         │
│  ┌───────────────────┐  │
│  │   kube-apiserver  │  │
│  │                   │  │
│  │  etcd client      │──┼──────► etcd cluster
│  │  (native)         │  │        (external)
│  └───────────────────┘  │
└─────────────────────────┘
```

### Kine-based (MySQL, PostgreSQL, NATS)

```
┌─────────────────────────────────────────┐
│        TenantControlPlane Pod           │
│                                         │
│  ┌───────────────────┐  ┌────────────┐  │
│  │   kube-apiserver  │  │    Kine    │  │
│  │                   │  │  sidecar   │  │
│  │  etcd client      │──┤            │──┼──► Database
│  │  (to Kine)        │  │  etcd →    │  │    (external)
│  │                   │  │  SQL/NATS  │  │
│  └───────────────────┘  └────────────┘  │
│           │                   ▲         │
│           └───────────────────┘         │
│           Unix Domain Socket            │
└─────────────────────────────────────────┘
```

## etcd Configuration

### Requirements

- etcd v3.4+ cluster
- TLS certificates for authentication
- RBAC enabled

### DataStore Example

```yaml
apiVersion: kamaji.clastix.io/v1alpha1
kind: DataStore
metadata:
  name: etcd-default
spec:
  driver: etcd
  endpoints:
    - etcd-0.etcd.kamaji-system.svc:2379
    - etcd-1.etcd.kamaji-system.svc:2379
    - etcd-2.etcd.kamaji-system.svc:2379
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

### Multi-Tenancy Model

Each TenantControlPlane gets:
- Dedicated etcd user
- Dedicated etcd role
- Key prefix: `/{namespace}_{name}/`

```
/tenant-ns_tenant-1/registry/pods/...
/tenant-ns_tenant-1/registry/services/...
/tenant-ns_tenant-2/registry/pods/...
/tenant-ns_tenant-2/registry/services/...
```

### Helm Installation (kamaji-etcd)

```bash
helm repo add clastix https://clastix.github.io/charts
helm upgrade --install etcd-default clastix/kamaji-etcd \
  --namespace etcd-system \
  --create-namespace \
  --set datastore.enabled=true
```

---

## MySQL Configuration

### Requirements

- MySQL 5.7+ or MariaDB 10.3+
- TLS optional but recommended
- User with CREATE DATABASE, CREATE USER, GRANT privileges

### DataStore Example

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
      secretReference:
        name: mysql-admin
        namespace: kamaji-system
        keyPath: username
    password:
      secretReference:
        name: mysql-admin
        namespace: kamaji-system
        keyPath: password
  tlsConfig:
    certificateAuthority:
      certificate:
        secretReference:
          name: mysql-ca
          namespace: kamaji-system
          keyPath: ca.crt
```

### Multi-Tenancy Model

Each TenantControlPlane gets:
- Dedicated database: `{namespace}_{name}`
- Dedicated user: `{namespace}_{name}`
- GRANT ALL on database

```sql
CREATE DATABASE tenant_ns_tenant_1;
CREATE USER 'tenant_ns_tenant_1'@'%' IDENTIFIED BY '...';
GRANT ALL PRIVILEGES ON tenant_ns_tenant_1.* TO 'tenant_ns_tenant_1'@'%';
```

### Kine Connection String

```
mysql://{user}:{password}@tcp({host}:{port})/{database}?tls=true
```

### Helm Installation (MariaDB)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install mariadb bitnami/mariadb \
  --namespace mysql-system \
  --create-namespace \
  --set auth.rootPassword=secretpassword \
  --set primary.persistence.size=10Gi
```

---

## PostgreSQL Configuration

### Requirements

- PostgreSQL 11+
- TLS optional but recommended
- User with CREATEDB, CREATEROLE privileges

### DataStore Example

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
        name: postgres-admin
        namespace: kamaji-system
        keyPath: username
    password:
      secretReference:
        name: postgres-admin
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

### Multi-Tenancy Model

Each TenantControlPlane gets:
- Dedicated database: `{namespace}_{name}`
- Dedicated role: `{namespace}_{name}`
- CONNECT, CREATE privileges on database

```sql
CREATE ROLE tenant_ns_tenant_1 WITH LOGIN PASSWORD '...';
CREATE DATABASE tenant_ns_tenant_1 OWNER tenant_ns_tenant_1;
GRANT CONNECT ON DATABASE tenant_ns_tenant_1 TO tenant_ns_tenant_1;
```

### Kine Connection String

```
postgres://{user}:{password}@{host}:{port}/{database}?sslmode=require
```

### Helm Installation (PostgreSQL)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install postgresql bitnami/postgresql \
  --namespace postgres-system \
  --create-namespace \
  --set auth.postgresPassword=secretpassword \
  --set primary.persistence.size=10Gi
```

---

## NATS Configuration

### Requirements

- NATS 2.x with JetStream enabled
- TLS optional

### DataStore Example

```yaml
apiVersion: kamaji.clastix.io/v1alpha1
kind: DataStore
metadata:
  name: nats-default
spec:
  driver: NATS
  endpoints:
    - nats.nats-system.svc:4222
  basicAuth:
    username:
      content: dXNlcg==  # base64: user
    password:
      secretReference:
        name: nats-credentials
        namespace: kamaji-system
        keyPath: password
```

### Multi-Tenancy Model

Each TenantControlPlane gets:
- Dedicated KeyValue bucket: `{namespace}_{name}`

NATS has simplified user management - the same credentials access all buckets.

### Kine Connection String

```
nats://{user}:{password}@{host}:{port}?bucket={bucket}
```

### Helm Installation (NATS)

```bash
helm repo add nats https://nats-io.github.io/k8s/helm/charts/
helm upgrade --install nats nats/nats \
  --namespace nats-system \
  --create-namespace \
  --set config.jetstream.enabled=true \
  --set config.jetstream.fileStore.pvc.size=10Gi
```

---

## DataStore Migration

Kamaji supports migrating TenantControlPlanes between DataStores of the **same driver type**.

### Supported Migrations

| From | To | Supported |
|------|-----|-----------|
| etcd → etcd | Yes |
| MySQL → MySQL | Yes |
| PostgreSQL → PostgreSQL | Yes |
| NATS → NATS | Yes |
| etcd → MySQL | No |
| MySQL → PostgreSQL | No |

### Migration Process

1. Update TCP spec with new DataStore reference
2. Kamaji detects change and enters migration mode
3. ValidatingWebhook blocks writes to old data
4. Migration job copies data to new DataStore
5. TCP switches to new DataStore
6. Webhook removed, normal operation resumes

### Migration Example

```yaml
# Before
apiVersion: kamaji.clastix.io/v1alpha1
kind: TenantControlPlane
metadata:
  name: tenant-1
spec:
  dataStore: etcd-bronze  # Original

# After
apiVersion: kamaji.clastix.io/v1alpha1
kind: TenantControlPlane
metadata:
  name: tenant-1
spec:
  dataStore: etcd-gold  # New target
```

### Monitor Migration

```bash
# Check TCP status
kubectl get tcp tenant-1 -o jsonpath='{.status.kubernetes.version.status}'
# Output: Migrating

# Watch migration job
kubectl get jobs -n kamaji-system -l kamaji.clastix.io/component=migrate

# Check logs
kubectl logs -n kamaji-system job/migrate-tenant-1
```

---

## DataStore Overrides

Route specific Kubernetes resources to different DataStores.

### Use Case

- Store Events in a separate, faster datastore
- Isolate Secrets in a dedicated secure datastore

### Configuration

```yaml
apiVersion: kamaji.clastix.io/v1alpha1
kind: TenantControlPlane
metadata:
  name: tenant-1
spec:
  dataStore: etcd-default
  dataStoreOverrides:
    - resource: events
      dataStore: etcd-events
    - resource: secrets
      dataStore: etcd-secrets
```

### Limitations

- Only etcd driver supported for overrides
- Resource must be a valid Kubernetes resource type

---

## Best Practices

### High Availability

**etcd:**
- Deploy 3+ node cluster
- Use anti-affinity for node distribution
- Enable periodic snapshots

**MySQL/PostgreSQL:**
- Use managed database service or replication
- Configure connection pooling
- Enable automated backups

**NATS:**
- Deploy 3+ node cluster
- Use JetStream with replication

### Security

1. **Always use TLS** for production
2. **Rotate credentials** periodically
3. **Limit network access** via NetworkPolicies
4. **Encrypt at rest** where supported

### Performance

| Driver | Recommended For |
|--------|----------------|
| etcd | Best performance, native protocol |
| PostgreSQL | Mature, excellent tooling |
| MySQL | Good compatibility |
| NATS | Lightweight, simple operations |

### Resource Sizing

| TCPs | etcd Memory | etcd CPU | DB Storage |
|------|-------------|----------|------------|
| 1-10 | 512Mi | 0.5 | 5Gi |
| 10-50 | 1Gi | 1 | 20Gi |
| 50-100 | 2Gi | 2 | 50Gi |
| 100+ | 4Gi+ | 4+ | 100Gi+ |

---

## Troubleshooting

### Connection Failures

```bash
# Check DataStore status
kubectl get datastore etcd-default -o yaml

# Test connectivity
kubectl run test --rm -it --image=busybox -- nc -zv etcd.kamaji-system.svc 2379

# Check certificates
kubectl get secret etcd-certs -n kamaji-system -o jsonpath='{.data.ca\.crt}' | base64 -d | openssl x509 -text
```

### Authentication Errors

```bash
# Verify credentials
kubectl get secret mysql-admin -n kamaji-system -o jsonpath='{.data.username}' | base64 -d

# Test with client
mysql -h mysql.database.svc -u admin -p
```

### Performance Issues

```bash
# Check etcd metrics
kubectl exec -it etcd-0 -n etcd-system -- etcdctl endpoint status --cluster

# Check database slow queries
kubectl logs -n mysql-system deploy/mariadb | grep -i slow
```

### Data Recovery

**etcd snapshot:**
```bash
kubectl exec -it etcd-0 -n etcd-system -- etcdctl snapshot save /tmp/backup.db
kubectl cp etcd-system/etcd-0:/tmp/backup.db ./backup.db
```

**MySQL dump:**
```bash
kubectl exec -it mariadb-0 -n mysql-system -- mysqldump -u root -p tenant_ns_tenant_1 > backup.sql
```

**PostgreSQL dump:**
```bash
kubectl exec -it postgresql-0 -n postgres-system -- pg_dump -U postgres tenant_ns_tenant_1 > backup.sql
```
