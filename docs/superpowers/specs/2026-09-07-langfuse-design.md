# Langfuse Deployment Design

## Overview

Deploy Langfuse (LLM observability platform) on the homelab K8s cluster using the official Helm chart, wired to existing CNPG Postgres, ClickHouse (new, in `shared-data`), Redis, and Garage S3. Accessible at `langfuse.naadroid.dev` (HTTPRoute) and `langfuse` (Tailscale Ingress).

## Architecture

Three new ArgoCD apps in sync-wave order:

| Wave | App | Type | Namespace |
|------|-----|------|-----------|
| 1 | `clickhouse-operator` | Helm (oci) | `clickhouse-operator` |
| 2 | `clickhouse` | Raw manifests | `shared-data` |
| 7 | `langfuse` | Multi-source Helm | `langfuse` |

Langfuse web + worker connect to:
- **Postgres**: `postgres-rw.shared-data.svc.cluster.local:5432` (DB: `langfuse`, user: `admin`)
- **ClickHouse**: `clickhouse.shared-data.svc.cluster.local:8123` (HTTP) / `9000` (native)
- **Redis**: `redis.shared-data.svc.cluster.local:6379` (no auth)
- **Garage S3**: `http://garage.garage.svc.cluster.local:3900` (bucket: `langfuse`, path-style)

## File Structure

```
clusters/home/
├── root/apps/
│   ├── clickhouse-operator.yaml
│   ├── clickhouse.yaml
│   └── langfuse.yaml
│
├── infrastructure/
│   └── langfuse/
│       └── values.yaml
│
└── apps/
    ├── clickhouse/
    │   ├── cluster.yaml     # ClickHouseCluster CR
    │   └── keeper.yaml      # KeeperCluster CR
    ├── postgres/
    │   └── langfuse-database.yaml
    └── langfuse/
        ├── namespace.yaml
        ├── waypoint.yaml
        ├── secret-sealed.yaml
        ├── httproute.yaml
        └── tailscale-ingress.yaml
```

## ArgoCD Applications

### clickhouse-operator (`root/apps/clickhouse-operator.yaml`)
- Chart: `oci://ghcr.io/clickhouse/clickhouse-operator-helm` v0.0.5
- Namespace: `clickhouse-operator`
- Sync-wave: `"1"` (alongside other operators)
- No custom values needed

### clickhouse (`root/apps/clickhouse.yaml`)
- Raw directory: `clusters/home/apps/clickhouse`
- Destination namespace: `shared-data`
- Sync-wave: `"2"` (after operator CRDs are established)
- Depends on clickhouse-operator being ready

### langfuse (`root/apps/langfuse.yaml`)
- Multi-source:
  1. `https://langfuse.github.io/langfuse-k8s` chart `langfuse` (pin to stable release)
  2. Homelab repo `clusters/home/infrastructure/langfuse/values.yaml` (ref: `values`)
  3. Homelab repo `clusters/home/apps/langfuse` (extra manifests)
- Destination namespace: `langfuse`
- Sync-wave: `"7"` (same as affine)

## ClickHouse Manifests (`apps/clickhouse/`)

### cluster.yaml — ClickHouseCluster CR
- Single replica (homelab scale)
- Longhorn PVC: `longhorn-bulk` storage class, 50Gi
- User `default` with password from a Secret `clickhouse-secret` in `shared-data`

### keeper.yaml — KeeperCluster CR
- Single replica
- Longhorn PVC: `longhorn-bulk`, 5Gi
- Required for ClickHouse coordination even on single-node

## CNPG Database CR (`apps/postgres/langfuse-database.yaml`)
- `spec.name: langfuse`
- `spec.owner: admin`
- `spec.cluster.name: postgres`
- No extensions needed (Langfuse doesn't require pgvector)

## Langfuse Namespace (`apps/langfuse/`)

### namespace.yaml
- Labels: `istio.io/dataplane-mode: ambient`, `istio.io/use-waypoint: waypoint`

### waypoint.yaml
- Same pattern as affine: `gatewayClassName: istio-waypoint`, `istio.io/waypoint-for: all`

### secret-sealed.yaml
Single SealedSecret `langfuse-secrets` in `langfuse` namespace with keys:

| Key | Description |
|-----|-------------|
| `DB_PASSWORD` | CNPG admin password (same value used by other apps) |
| `CLICKHOUSE_PASSWORD` | ClickHouse default user password |
| `S3_ACCESS_KEY_ID` | Garage bucket access key |
| `S3_SECRET_ACCESS_KEY` | Garage bucket secret key |
| `NEXTAUTH_SECRET` | Random 32-byte hex (`openssl rand -hex 32`) |
| `SALT` | Random 32-byte hex (`openssl rand -hex 32`) |
| `ENCRYPTION_KEY` | 256-bit hex (`openssl rand -hex 32`) — Langfuse v3 requirement |

A matching `clickhouse-secret` Secret in `shared-data` must be sealed separately (referenced by the ClickHouseCluster CR).

### httproute.yaml
- hostname: `langfuse.naadroid.dev`
- parentRef: `homelab-gateway` in `istio-ingress`
- backend: `langfuse` service port `3000`

### tailscale-ingress.yaml
- `ingressClassName: tailscale`
- host: `langfuse`
- backend port: `3000`

## Helm Values (`infrastructure/langfuse/values.yaml`)

```yaml
langfuse:
  nextauth:
    url: "https://langfuse.naadroid.dev"
    secret:
      secretKeyRef:
        name: langfuse-secrets
        key: NEXTAUTH_SECRET
  salt:
    secretKeyRef:
      name: langfuse-secrets
      key: SALT
  encryptionKey:
    secretKeyRef:
      name: langfuse-secrets
      key: ENCRYPTION_KEY

postgresql:
  deploy: false
  host: "postgres-rw.shared-data.svc.cluster.local"
  port: 5432
  auth:
    username: "admin"
    database: "langfuse"
    existingSecret: "langfuse-secrets"
    secretKeys:
      userPasswordKey: "DB_PASSWORD"

clickhouse:
  deploy: false
  host: "clickhouse.shared-data.svc.cluster.local"
  auth:
    username: "default"
    existingSecret: "langfuse-secrets"
    existingSecretKey: "CLICKHOUSE_PASSWORD"

redis:
  deploy: false
  host: "redis.shared-data.svc.cluster.local"
  port: 6379

s3:
  deploy: false
  bucket: "langfuse"
  region: "garage"
  endpoint: "http://garage.garage.svc.cluster.local:3900"
  forcePathStyle: true
  accessKeyId:
    secretKeyRef:
      name: langfuse-secrets
      key: S3_ACCESS_KEY_ID
  secretAccessKey:
    secretKeyRef:
      name: langfuse-secrets
      key: S3_SECRET_ACCESS_KEY
```

## Manual Pre-Deploy Steps

These must be done before the ArgoCD apps sync:

1. **Garage bucket**: Create bucket `langfuse` and an access key pair via Garage admin API (`kubectl exec` into a Garage pod). Save the key ID and secret for sealing.

2. **Seal secrets**:
   ```bash
   # Generate new values
   openssl rand -hex 32  # for NEXTAUTH_SECRET, SALT, ENCRYPTION_KEY

   # Seal with kubeseal (namespace langfuse)
   kubectl create secret generic langfuse-secrets \
     --namespace langfuse --dry-run=client -o yaml \
     --from-literal=DB_PASSWORD=... \
     --from-literal=CLICKHOUSE_PASSWORD=... \
     --from-literal=S3_ACCESS_KEY_ID=... \
     --from-literal=S3_SECRET_ACCESS_KEY=... \
     --from-literal=NEXTAUTH_SECRET=... \
     --from-literal=SALT=... \
     --from-literal=ENCRYPTION_KEY=... \
   | kubeseal --format yaml > clusters/home/apps/langfuse/secret-sealed.yaml

   # Seal clickhouse-secret (namespace shared-data)
   kubectl create secret generic clickhouse-secret \
     --namespace shared-data --dry-run=client -o yaml \
     --from-literal=password=... \
   | kubeseal --format yaml > clusters/home/apps/clickhouse/secret-sealed.yaml
   ```

3. **Commit and push** — ArgoCD picks up all three apps automatically.

## Resource Sizing

| Component | CPU req | Memory req | Storage |
|-----------|---------|------------|---------|
| ClickHouseCluster | 500m | 1Gi | 50Gi Longhorn |
| KeeperCluster | 200m | 256Mi | 5Gi Longhorn |
| Langfuse web | 250m | 512Mi | — |
| Langfuse worker | 250m | 512Mi | — |
