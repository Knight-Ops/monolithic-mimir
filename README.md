# Monolithic Mimir Helm Chart

A Helm chart for deploying [Grafana Mimir](https://grafana.com/oss/mimir/) in **monolithic mode** (`-target=all`) on Kubernetes. All Mimir components (distributor, ingester, compactor, store-gateway, querier, query-frontend, ruler) run within a single StatefulSet.

## Prerequisites

- Kubernetes 1.24+
- Helm 3.x
- An S3-compatible object storage bucket (AWS S3, MinIO, etc.)
- A Kubernetes Secret containing your S3 credentials (recommended)

## Quick Start

### 1. Create an S3 credentials secret

```bash
kubectl create namespace mimir

kubectl create secret generic mimir-s3-credentials \
  --namespace mimir \
  --from-literal=AWS_ACCESS_KEY_ID=<your-access-key> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<your-secret-key>
```

### 2. Install the chart

```bash
helm install mimir ./monolithic-mimir \
  --namespace mimir \
  --set s3.endpoint=s3.us-east-1.amazonaws.com \
  --set s3.bucketName=my-mimir-blocks \
  --set s3.region=us-east-1 \
  --set s3.existingSecret=mimir-s3-credentials
```

### 3. Verify the deployment

```bash
kubectl get pods -n mimir
kubectl port-forward -n mimir svc/mimir-monolithic-mimir 8080:8080
curl http://localhost:8080/ready
```

## Configuration

All configuration is managed through `values.yaml`. The following tables describe the available parameters.

### General

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of StatefulSet replicas | `1` |
| `image.repository` | Mimir container image | `grafana/mimir` |
| `image.tag` | Image tag (defaults to chart `appVersion`) | `""` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `nameOverride` | Override the chart name | `""` |
| `fullnameOverride` | Override the full release name | `""` |

### Storage

| Parameter | Description | Default |
|-----------|-------------|---------|
| `storage.size` | Persistent volume size for `/data` | `10Gi` |
| `storage.storageClassName` | Storage class name (empty uses cluster default) | `""` |

### S3 Backend

| Parameter | Description | Default |
|-----------|-------------|---------|
| `s3.endpoint` | S3 endpoint URL | `s3.amazonaws.com` |
| `s3.bucketName` | S3 bucket name | `mimir-blocks` |
| `s3.region` | S3 region | `us-east-1` |
| `s3.insecure` | Use HTTP instead of HTTPS (for homelab S3 like MinIO/TrueNAS) | `false` |
| `s3.accessKeyId` | Inline access key (creates a Secret; prefer `existingSecret`) | `""` |
| `s3.secretAccessKey` | Inline secret key (creates a Secret; prefer `existingSecret`) | `""` |
| `s3.existingSecret` | Name of an existing Secret containing S3 credentials | `""` |
| `s3.credentials.accessKeyId.envVar` | Env var name for the access key | `AWS_ACCESS_KEY_ID` |
| `s3.credentials.accessKeyId.secretKey` | Key in the Secret for the access key | `AWS_ACCESS_KEY_ID` |
| `s3.credentials.secretAccessKey.envVar` | Env var name for the secret key | `AWS_SECRET_ACCESS_KEY` |
| `s3.credentials.secretAccessKey.secretKey` | Key in the Secret for the secret key | `AWS_SECRET_ACCESS_KEY` |

### Service

| Parameter | Description | Default |
|-----------|-------------|---------|
| `service.type` | Kubernetes Service type | `ClusterIP` |
| `service.httpPort` | HTTP API port | `8080` |
| `service.grpcPort` | gRPC port | `9095` |
| `service.memberlistPort` | Memberlist gossip port | `7946` |

### HTTPRoute (Gateway API)

| Parameter | Description | Default |
|-----------|-------------|---------|
| `httpRoute.enabled` | Enable Gateway API HTTPRoute | `false` |
| `httpRoute.annotations` | HTTPRoute annotations | `{}` |
| `httpRoute.parentRefs` | Gateway parent references | `[]` |
| `httpRoute.hostnames` | Hostnames for the route | `[]` |
| `httpRoute.rules` | Routing rules | PathPrefix `/` |

### Pod Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `resources` | CPU/memory requests and limits | `{}` |
| `nodeSelector` | Node selector labels | `{}` |
| `tolerations` | Pod tolerations | `[]` |
| `affinity` | Pod affinity rules | `{}` |
| `podAnnotations` | Additional pod annotations | `{}` |
| `podLabels` | Additional pod labels | `{}` |
| `podSecurityContext` | Pod-level security context | `{fsGroup: 10001}` |
| `securityContext` | Container-level security context | `{}` |

### Service Account

| Parameter | Description | Default |
|-----------|-------------|---------|
| `serviceAccount.create` | Create a ServiceAccount | `true` |
| `serviceAccount.automount` | Automount the SA token | `true` |
| `serviceAccount.annotations` | SA annotations (useful for IRSA) | `{}` |
| `serviceAccount.name` | SA name override | `""` |

### Mimir Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `mimir.config` | Full Mimir YAML config (rendered as a Go template) | See `values.yaml` |

## S3 Credentials

The chart supports three approaches for providing S3 credentials:

### Option 1: Existing Secret (recommended)

Create a Secret externally (via Sealed Secrets, External Secrets Operator, SOPS, etc.) and reference it:

```yaml
s3:
  existingSecret: my-s3-secret
```

If your secret uses non-default key names, map them:

```yaml
s3:
  existingSecret: my-s3-secret
  credentials:
    accessKeyId:
      envVar: AWS_ACCESS_KEY_ID
      secretKey: minio-access-key    # key name inside your Secret
    secretAccessKey:
      envVar: AWS_SECRET_ACCESS_KEY
      secretKey: minio-secret-key    # key name inside your Secret
```

### Option 2: Inline credentials

The chart creates a Secret for you (not recommended for production):

```yaml
s3:
  accessKeyId: AKIAIOSFODNN7EXAMPLE
  secretAccessKey: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

### Option 3: IAM Roles for Service Accounts (IRSA)

If running on AWS EKS with IRSA, you don't need to set any S3 credential values. Instead, annotate the ServiceAccount:

```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/mimir-s3-role
```

The Mimir config will still reference `${AWS_ACCESS_KEY_ID}` env vars, but with IRSA the AWS SDK handles authentication automatically. You may want to override `mimir.config` to remove the `access_key_id` and `secret_access_key` fields entirely in this case.

## Homelab / Local S3 Setup

If you're running a local S3-compatible backend (MinIO, TrueNAS, etc.) that doesn't have a trusted TLS certificate, enable `insecure` mode:

```yaml
s3:
  endpoint: minio.local:9000
  bucketName: mimir
  region: us-east-1
  insecure: true
  existingSecret: mimir-s3-credentials
```

The `podSecurityContext.fsGroup: 10001` is enabled by default, which ensures the Mimir container (running as UID 10001) has write permissions to the persistent volume. This is particularly important when using block storage backends like iSCSI that mount as root.

## Customizing Mimir Configuration

The `mimir.config` value contains the full Mimir YAML configuration and is rendered as a **Go template**, which means you can use Helm expressions like `{{ .Values.s3.endpoint }}` inside it.

S3 credentials are injected via environment variable expansion at runtime using Mimir's `-config.expand-env=true` flag. This keeps credentials out of the ConfigMap.

To override the config entirely:

```yaml
mimir:
  config: |
    multitenancy_enabled: true
    server:
      http_listen_port: 8080
    # ... your full config here
```

## Flux CD (GitOps) Deployment

To deploy this chart using Flux CD:

### 1. Define a GitRepository source

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: monolithic-mimir
  namespace: flux-system
spec:
  interval: 5m
  url: https://github.com/<your-org>/monolithic-mimir
  ref:
    branch: main
```

### 2. Define a HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: mimir
  namespace: mimir
spec:
  interval: 10m
  chart:
    spec:
      chart: .
      sourceRef:
        kind: GitRepository
        name: monolithic-mimir
        namespace: flux-system
  values:
    s3:
      endpoint: s3.us-east-1.amazonaws.com
      bucketName: my-mimir-blocks
      region: us-east-1
      existingSecret: mimir-s3-credentials
    storage:
      size: 50Gi
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: "2"
        memory: 4Gi
```

## Exposed Endpoints

Once deployed, the following endpoints are available on the HTTP port:

| Endpoint | Description |
|----------|-------------|
| `/ready` | Readiness check |
| `/metrics` | Prometheus metrics |
| `/api/v1/push` | Remote write endpoint |
| `/prometheus/*` | Prometheus-compatible query API |
| `/api/v1/rules` | Ruler API |

## Architecture

This chart deploys Mimir as a **StatefulSet** running all components in a single process:

- **StatefulSet** with persistent volume for TSDB data, compactor working directory, and bucket store sync
- **Service** (ClusterIP) exposing HTTP (8080), gRPC (9095), and memberlist (7946) ports
- **Headless Service** for StatefulSet pod DNS and memberlist peer discovery
- **ConfigMap** containing the rendered `mimir.yaml`
- **Secret** (optional) for S3 credentials
- **ServiceAccount** with optional annotations for IRSA
- **HTTPRoute** (optional) for Gateway API ingress

## Upgrading

When `mimir.config` changes, pods automatically restart due to a `checksum/config` annotation on the StatefulSet pod template.

To upgrade:

```bash
helm upgrade mimir ./monolithic-mimir --namespace mimir --reuse-values
```
