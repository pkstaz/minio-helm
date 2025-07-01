# MinIO Helm Chart for OpenShift

A Helm chart for deploying MinIO object storage on OpenShift 4.18+ with automatic bucket configuration and Data Science Pipeline integration.

## Features

- ✅ **OpenShift Compatible**: Works with OpenShift Security Context Constraints (SCC)
- ✅ **Automatic Bucket Setup**: Creates and configures buckets via post-install Job
- ✅ **Data Science Integration**: Webhook notifications for OpenShift AI Pipelines
- ✅ **Flexible Access Policies**: Support for upload, download, public, and custom policies
- ✅ **Versioning Support**: Enable object versioning per bucket
- ✅ **Auto-generated Routes**: OpenShift Routes with automatic hostname generation
- ✅ **TLS Support**: Edge termination for secure access

## Prerequisites

- OpenShift 4.18+
- Helm 3.x
- Persistent storage available in the cluster

## Installation

### Basic Installation

```bash
# Clone the repository
git clone https://github.com/pkstaz/minio-helm.git
cd minio-helm

# Install with default values
helm install minio .
```

### Installation with Custom Values

```bash
# Install with custom configuration
helm install minio . -f values-example.yaml

# Or using --set flags
helm install minio . \
  --set auth.rootUser=admin \
  --set auth.rootPassword=SecurePass123 \
  --set persistence.size=50Gi
```

## Configuration

### Basic Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | MinIO image repository | `quay.io/minio/minio` |
| `image.tag` | MinIO image tag | `latest` |
| `auth.rootUser` | MinIO root username | `minio` |
| `auth.rootPassword` | MinIO root password | `minio123` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.size` | Storage size | `10Gi` |
| `route.enabled` | Enable OpenShift Route | `true` |
| `console.enabled` | Enable MinIO Console | `true` |

### Bucket Configuration

Enable automatic bucket creation and configuration:

```yaml
buckets:
  enabled: true
  list:
    - name: "my-bucket"
      policy: "upload"        # none, download, upload, public, custom
      versioning: true        # Enable object versioning
      notifications:
        enabled: true
        events: ["put"]       # put, delete, get
        webhook:
          endpoint: "http://webhook-service:8080/webhook"
          authToken: "secret-token"
        pipeline:
          name: "data-pipeline"
          namespace: "openshift-ai"
```

### Access Policies

- **`none`**: Private access only (default)
- **`download`**: Public read access
- **`upload`**: Public write access  
- **`public`**: Full public access
- **`custom`**: Use custom policy JSON

### Security Context

The chart automatically configures OpenShift-compatible security contexts:

```yaml
securityContext:
  runAsNonRoot: true
  # OpenShift assigns UIDs automatically within allowed ranges
```

## Usage Examples

### Example 1: Basic MinIO

```yaml
# values-basic.yaml
auth:
  rootUser: "admin"
  rootPassword: "MySecurePassword123"

persistence:
  size: 20Gi

route:
  enabled: true
  tls:
    enabled: true
```

### Example 2: Data Science Setup

```yaml
# values-datascience.yaml
auth:
  rootUser: "dataops"
  rootPassword: "DataSciencePass123"

buckets:
  enabled: true
  list:
    - name: "raw-data"
      policy: "upload"
      versioning: true
      notifications:
        enabled: true
        events: ["put"]
        webhook:
          endpoint: "http://pipeline-webhook.openshift-ai.svc.cluster.local:8080/minio"
        pipeline:
          name: "data-ingestion"
          namespace: "openshift-ai"
    
    - name: "processed-data"
      policy: "download"
      versioning: true
    
    - name: "public-datasets"
      policy: "public"
      versioning: false
```

## Accessing MinIO

After installation, MinIO will be available through OpenShift Routes:

```bash
# Get route URLs
oc get routes

# Example URLs:
# API: https://minio-api-myproject.apps.cluster.local
# Console: https://minio-console-myproject.apps.cluster.local
```

## Data Science Pipeline Integration

This chart supports automatic webhook notifications to OpenShift AI Data Science Pipelines:

1. **Upload file** → MinIO bucket
2. **MinIO sends webhook** → Your webhook handler
3. **Webhook handler** → Triggers OpenShift AI Pipeline
4. **Pipeline processes** → The uploaded data

### Webhook Payload Example

```json
{
  "EventName": "s3:ObjectCreated:Put",
  "Key": "data/file.csv",
  "Records": [{
    "s3": {
      "bucket": {"name": "raw-data"},
      "object": {"key": "data/file.csv"}
    }
  }]
}
```

## Troubleshooting

### Bucket Setup Job Issues

If the automatic bucket setup fails, check the job logs:

```bash
kubectl logs job/minio-bucket-setup
```

If the job fails to connect, you'll see manual configuration instructions:

```
========================================
MANUAL CONFIGURATION REQUIRED
========================================
Connect to MinIO using:
  mc alias set minio http://minio-service:9000 admin <password>

Required buckets to create:
  • my-bucket
    - Policy: upload
    - Versioning: enabled
    - Notifications: put events
    - Webhook endpoint: http://webhook-service:8080/webhook
    - Target pipeline: data-pipeline (openshift-ai)
========================================
```

### Common Issues

1. **SCC Issues**: The chart automatically handles OpenShift Security Context Constraints
2. **Storage Issues**: Ensure your cluster has available persistent storage
3. **Route Issues**: OpenShift automatically generates hostnames if not specified
4. **Webhook Issues**: Ensure your webhook endpoint is accessible from the MinIO pod

### Manual Bucket Configuration

If automatic setup fails, use MinIO Client:

```bash
# Port forward to access MinIO
kubectl port-forward svc/minio 9000:9000

# Configure MinIO client
mc alias set minio http://localhost:9000 admin <password>

# Create bucket
mc mb minio/my-bucket

# Set policy
mc anonymous set upload minio/my-bucket

# Enable versioning
mc version enable minio/my-bucket
```

## Upgrading

```bash
# Upgrade with new values
helm upgrade minio . -f values.yaml

# The post-upgrade hook will reconfigure buckets if needed
```

## Uninstalling

```bash
# Remove the MinIO deployment
helm uninstall minio

# Note: PVCs are not automatically deleted
kubectl delete pvc minio-pvc
```

## Contributing

This chart is maintained by Carlos Estay (cestay@redhat.com).

- Repository: https://github.com/pkstaz/minio-helm
- Issues: https://github.com/pkstaz/minio-helm/issues

## License

This Helm chart is provided under the MIT License.

## Version History

- **0.1.0**: Initial release with OpenShift support and Data Science Pipeline integration