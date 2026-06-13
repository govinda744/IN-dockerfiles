# Invoice Ninja Helm Chart

This chart deploys the Debian Invoice Ninja stack based on `debian/docker-compose.yml`:
- `app` (php-fpm + supervisor)
- `nginx`
- `mysql`
- `redis`

## Install

```bash
helm install invoiceninja ./charts/invoiceninja \
  -n invoiceninja \
  --create-namespace \
  --set secret.appKey='base64:YOUR_APP_KEY' \
  --set env.appUrl='http://invoiceninja.local'
```

## Upgrade

```bash
helm upgrade invoiceninja ./charts/invoiceninja \
  -n invoiceninja \
  --set secret.appKey='base64:YOUR_APP_KEY' \
  --set env.appUrl='http://invoiceninja.local'
```

## Access

```bash
kubectl -n invoiceninja port-forward svc/invoiceninja-nginx 8080:80
```

Open http://127.0.0.1:8080

## Persistence

Persistence is disabled by default so the chart works on clusters without a default StorageClass.

To enable persistence, set:
- `persistence.appPublic.enabled=true`
- `persistence.appStorage.enabled=true`
- `persistence.mysqlData.enabled=true`
- `persistence.redisData.enabled=true`

Optionally set `storageClassName` for each claim.

## Pod Security Admission

This chart's security contexts are designed to pass Kubernetes Pod Security Admission (PSA) `baseline:latest` policy by default, with violations of the stricter `restricted:latest` policy logged as warnings.

### Understanding the defaults

The `app` container necessarily runs as root (supervisord) and adds capabilities (CHOWN, SETUID, SETGID, DAC_OVERRIDE, FOWNER) for file operations. These settings:
- **Pass baseline enforcement** (allowed to run as root; non-default capabilities permitted)
- **Violate restricted policy** (restricted requires `runAsNonRoot=true` and forbids these capabilities)

The `nginx` init container (`seed-public-dir`) also sets `runAsNonRoot: false` because the upstream image needs root privileges to seed the public directory.

### Cluster PSA defaults

Whether these violations cause warnings or failures depends on your cluster's PSA configuration:

- **Baseline enforced, restricted warned** (default Talos config):
  - Pods are admitted; warnings appear in kubectl output and API server logs
  - No manual namespace labeling required
  - This chart deploys without errors

- **Baseline enforced, restricted enforced**:
  - Pods are rejected and cannot be deployed
  - Either:
    - Label the namespace to exempt it: `kubectl label namespace <name> pod-security.kubernetes.io/enforce=baseline`
    - Or modify cluster-wide PSA defaults to use baseline enforcement only (not recommended)
    - Or change container images to support non-root execution and remove added capabilities (PRs welcome!)
