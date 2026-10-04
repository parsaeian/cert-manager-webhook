# Sotoon Cert Manager Webhook

Cert manager issuer webhook for Sotoon DNS, forked from [Here](https://github.com/cert-manager/webhook-example).

## Installation

### Public (GitHub Container Registry)

```bash
helm install cert-manager-webhook oci://ghcr.io/sotoon/cert-manager-webhook/helm \
  --namespace cert-manager \
  --version v1.3.21 \
  --set image.repository=ghcr.io/sotoon/cert-manager-webhook \
  --set image.tag=v1.7.3
```

The image is `ghcr.io/sotoon/cert-manager-webhook` (git tag). The chart is a separate GHCR package at `ghcr.io/sotoon/cert-manager-webhook/helm` (Helm chart version).

Currently published on GHCR:

| Package | Tag |
| --- | --- |
| Chart `oci://ghcr.io/sotoon/cert-manager-webhook/helm` | `v1.3.21` (app version `v1.7.3`) |
| Image `ghcr.io/sotoon/cert-manager-webhook` | `v1.7.3` |

Both tags include the leading `v` (`--version 1.3.21` or `image.tag=1.7.3` are not found).
Set `image.repository` and `image.tag` explicitly: the chart's default image
(`registry.platform.sotoon.ir/delivery/cert-manager-webhook:v1.6.2`) is only
reachable from Sotoon's internal network.

### Pod Security

The chart (source in [`deploy/cert-manager-webhook`](./deploy/cert-manager-webhook),
version `v1.3.22`) sets a security context that satisfies the Pod Security
Standard `restricted`, so the webhook runs in the same namespace as
cert-manager even when that namespace enforces `restricted`:

| Value | Default |
| --- | --- |
| `podSecurityContext` | `runAsNonRoot: true`, `runAsUser/runAsGroup: 65532`, `seccompProfile: RuntimeDefault`, sysctl `net.ipv4.ip_unprivileged_port_start=0` (binds 443 without root; affects only the pod's own network namespace) |
| `securityContext` | `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]` |

An `emptyDir` is mounted at `/tmp` so the read-only root filesystem is safe.
To get the previous behaviour (no security context), set both to `null`
(setting them to `{}` keeps the defaults, because Helm merges maps):

```bash
helm install cert-manager-webhook ./deploy/cert-manager-webhook \
  --namespace cert-manager \
  --set podSecurityContext=null \
  --set securityContext=null
```

The chart's default image is now the public `ghcr.io/sotoon/cert-manager-webhook`
with the chart's `appVersion` (`v1.7.3`).

Until `v1.3.22` is published to GHCR, install it from this repository:

```bash
helm install cert-manager-webhook ./deploy/cert-manager-webhook --namespace cert-manager
```

### Sotoon internal registry

Install by running below command:

```terminal
helm install cert-manager-webhook https://admin.registry.platform.ske.sotoon.ir/repository/helm-hosted/cert-manager-webhook \
  --namespace cert-manager \
  --version 1.3.20
```

make sure you have access to admin.registry.platform.ske.sotoon.ir

## Example

### In-Cluster Mode

```yaml
# Create an issuer
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt
  namespace: delivery-test
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: "info@sotoon.ir"
    privateKeySecretRef:
      name: test-private-key
    solvers:
      - dns01:
          webhook:
            groupName: "acme.sotoon.ir"
            solverName: sotoon
            config:
              namespace: delivery-test
              inCluster: true
---
# Create a Certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-sub-acutulus-ir
  namespace: delivery-test
spec:
  dnsNames:
    - "test.sub.acutulus.ir"
    - "*.test.sub.acutulus.ir"
  issuerRef:
    name: letsencrypt
  secretName: test-sub-acutulus-ir-tls
```

### When InCluster is false

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: <name>
spec:
  acme:
    email: <their-own-email>
    preferredChain: ""
    privateKeySecretRef:
      name: <name of secret>
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - dns01:
          webhook:
            groupName: acme.sotoon.ir
            solverName: sotoon
            config:
              apiTokenSecretRef:
                key: <key>
                name: <apiTokenSecretRef>
              endpoint: https://api.sotoon.ir/delivery/v1
              inCluster: false
              namespace: <namespace in cdn>

```

### Air-Gapped Environments

For air-gapped Kubernetes clusters with access only to specific internal DNS resolvers, enable `usePublicResolver`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: <name>
spec:
  acme:
    email: <their-own-email>
    preferredChain: ""
    privateKeySecretRef:
      name: <name of secret>
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - dns01:
          webhook:
            groupName: acme.sotoon.ir
            solverName: sotoon
            config:
              apiTokenSecretRef:
                key: <key>
                name: <apiTokenSecretRef>
              endpoint: https://api.sotoon.ir/delivery/v1
              inCluster: false
              namespace: <namespace in cdn>
              usePublicResolver: true  # Query configured nameservers directly
```

Then configure the DNS resolver via Helm values:

```yaml
# values.yaml
nameServers:
  - 10.0.0.5
  - 10.0.0.6
```

Or via Helm install command:

```bash
helm install cert-manager-webhook \
  https://admin.registry.platform.ske.sotoon.ir/repository/helm-hosted/cert-manager-webhook \
  --namespace cert-manager \
  --version 1.3.20 \
  --set "nameServers[0]=10.0.0.5" \
  --set "nameServers[1]=10.0.0.6"
```

**Note:** When `usePublicResolver` is not set (defaults to `false`), the webhook recursively queries from root nameservers for backward compatibility.

For more information [read this doc](./docs/cert-manager-webhook.md).
