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

The chart does not set a `securityContext`, and the image runs as root and
listens on port 443. In a namespace that enforces the Pod Security Standard
`restricted`, the webhook pod is rejected; it runs under `baseline`. To run
it under `restricted` today, the pod needs (for example via a post-renderer
or a patched copy of the chart):

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 65532
    runAsGroup: 65532
    seccompProfile:
      type: RuntimeDefault
    sysctls:
      - name: net.ipv4.ip_unprivileged_port_start   # bind 443 without root
        value: "0"
  containers:
    - name: helm
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

With a read-only root filesystem, also mount an `emptyDir` at `/tmp`.

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
