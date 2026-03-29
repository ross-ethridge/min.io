# min.io

My personal MinIO configuration files for running a MinIO tenant in Kubernetes, backed by DirectPV storage.

These are not a general-purpose deployment — they reflect my own setup. Use them as a reference or starting point.

| File | Purpose |
| --- | --- |
| `demucs-minio.yaml` | Helm values for the `demucs-minio` tenant |
| `direct-pv.yaml` | StorageClass for DirectPV |

---

## Prerequisites

### 1. Install the MinIO Operator

The [MinIO Operator](https://github.com/minio/operator) manages MinIO tenants as Kubernetes custom resources.

```bash
helm repo add minio https://operator.min.io
helm repo update
helm install minio-operator minio/operator -n minio-operator --create-namespace
```

Verify:

```bash
kubectl get pods -n minio-operator
```

### 2. Install DirectPV

[DirectPV](https://github.com/minio/directpv) is a CSI driver that provisions storage directly from raw block devices on your nodes. MinIO recommends it for performance.

Download the `kubectl-directpv` binary from the [DirectPV releases page](https://github.com/minio/directpv/releases), replacing `${release}` with the version you want (e.g. `4.0.9`):

```bash
curl -fLo kubectl-directpv \
  https://github.com/minio/directpv/releases/download/v${release}/kubectl-directpv_${release}_linux_amd64
chmod +x kubectl-directpv
mv kubectl-directpv /usr/local/bin/
```

Install the DirectPV controller into the cluster:

```bash
kubectl-directpv install
```

Verify:

```bash
kubectl get pods -n directpv
```

#### Discover and initialize drives

DirectPV needs to know which drives it can use. Run discovery (optionally scoped to a node):

```bash
kubectl-directpv discover --nodes <node-name>
```

This generates a `drives.yaml` file listing available block devices. Review it, then initialize — `--dangerous` is required as this will format the drives with XFS and erase all existing data:

```bash
kubectl-directpv init drives.yaml --dangerous
```

Verify drives are `Ready`:

```bash
kubectl-directpv list drives
```

**Note:** The StorageClass must include `parameters: fsType: xfs` or provisioning will fail with an "unsupported filesystem type" error — see `direct-pv.yaml`.

### 3. Apply the StorageClass

```bash
kubectl apply -f direct-pv.yaml
```

---

## Deploying a tenant

### 1. Create the namespace

```bash
kubectl create namespace <tenant-namespace>
```

### 2. Create the credentials secret

```bash
cat > /tmp/config.env << 'EOF'
export MINIO_ROOT_USER=<your-access-key>
export MINIO_ROOT_PASSWORD=<your-secret-key>
EOF

kubectl create secret generic <tenant-name> \
  --from-file=config.env=/tmp/config.env \
  -n <tenant-namespace>
```

### 3. Deploy the tenant

```bash
helm install <tenant-name> minio/tenant \
  -f demucs-minio.yaml \
  -n <tenant-namespace>
```

Verify:

```bash
kubectl get all -n <tenant-namespace>
```

The tenant pod should reach `2/2 Running`.

### 4. Access the console

The console is not exposed publicly. Port-forward to access it:

```bash
kubectl port-forward svc/<tenant-name>-console 9090:9090 -n <tenant-namespace>
```

Then browse to `http://localhost:9090` and log in with your `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`.

---

## References

- [MinIO Operator](https://github.com/minio/operator)
- [DirectPV](https://github.com/minio/directpv)
- [MinIO](https://github.com/minio/minio)
