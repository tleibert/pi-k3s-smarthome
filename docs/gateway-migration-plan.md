# Ingress → Gateway API Migration Plan

## Current State

| App | Host | Port | Ingress Controller |
|-----|------|------|-------------------|
| dashy | `dashy.trevorpi.lan` | 8080 | Traefik (k3s built-in) |
| glances | `glances.trevorpi.lan` | 61208 | Traefik (k3s built-in) |
| homeassistant | `home.trevorpi.lan` | 8123 | Traefik (k3s built-in) |
| zwave | `zwave.trevorpi.lan` | 8091 | Traefik (k3s built-in) |

- **GitOps**: Flux CD, repo → `./clusters/my-pi`
- **Helm chart**: bjw-s `app-template` v5.0.1 for all apps
- **Ingress method**: `ingress.main.enabled: true` in HelmRelease values
- **K3s Traefik**: built-in, handles standard `Ingress` resources automatically

## Target State

- Standard Kubernetes `Ingress` resources → replaced by Gateway API (`Gateway` + `HTTPRoute`)
- All Gateway API resources managed via Flux GitOps
- No `kubectl apply` or manual resource creation
- Traefik continues as data plane; now triggered via Gateway API instead of Ingress

## Architecture Decision

**Keep Traefik as the data plane.** K3s ships with Traefik, and Traefik v3 (shipped in k3s ≥ v1.31) supports Gateway API natively. This avoids introducing a second ingress layer (like Envoy Gateway or Istio) just for API conformance.

Traefik's Gateway API provider translates `Gateway` + `HTTPRoute` into its internal routing config — same as it does for `Ingress` today.

## Chart Native `route` Support (Deep Dive)

The bjw-s `app-template` chart (v5.0.1, the exact version this cluster uses) includes the bjw-s `common` library chart (also v5.0.1). That library has **built-in Gateway API support** via a `route` block in `values.yaml` — parallel to the existing `ingress` block.

### How It Works

Instead of writing raw `HTTPRoute` YAML manifests as separate files, you declare routes directly in the HelmRelease `values:` section. The common library's `_route.tpl` template renders the appropriate Gateway API object (`HTTPRoute`, `GRPCRoute`, `TCPRoute`, `TLSRoute`, or `UDPRoute`).

### `route` Block Schema (from `values.yaml`)

```yaml
route:
  main:
    enabled: true                    # false to disable
    annotations: {}                  # annotations on the HTTPRoute
    labels: {}                       # labels on the HTTPRoute
    kind: HTTPRoute                  # one of: HTTPRoute, GRPCRoute, TCPRoute, TLSRoute, UDPRoute (default: HTTPRoute)
    hostnames:                       # Host header matching (Helm templates supported)
      - "myapp.example.com"
    parentRefs:                      # Which Gateway this route attaches to
      - group: gateway.networking.k8s.io  # default
        kind: Gateway                     # default
        name: internal                    # REQUIRED — must exist
        namespace: default                # optional, omit for same namespace
        sectionName: http                 # optional, matches a Gateway listener name
        port: 80                          # optional
    rules:                           # Routing rules
      - backendRefs:                 # If empty, defaults to primary service (main controller, http port)
          - identifier: main         # references a service.controller identifier
            port: http               # matches a service port name
          - name: some-other-svc     # explicit Service name (overrides identifier lookup)
            port: 9000              # explicit port number
```

### API Version Auto-Detection

The template auto-detects the installed Gateway API CRD version:

| CRD Available | `apiVersion` Used |
|---------------|------------------|
| `gateway.networking.k8s.io/v1` | `gateway.networking.k8s.io/v1` |
| `gateway.networking.k8s.io/v1beta1` | `gateway.networking.k8s.io/v1beta1` |
| Neither (fallback) | `gateway.networking.k8s.io/v1alpha2` |

### `backendRefs` Resolution

When `backendRefs` is **omitted or empty**, the template defaults to the **primary service** — the Service matching the first controller with an `http` port. This means:

```yaml
route:
  main:
    enabled: true
    hostnames:
      - "dashy.trevorpi.lan"
    parentRefs:
      - name: internal
    # No rules — auto-routes to the primary service on port http
```

When explicit, you use either `identifier` (references `service.<id>.controller`) or `name` + `port`:

```yaml
rules:
  - backendRefs:
      - identifier: main
        port: http
      - name: some-external-svc
        port: 9090
```

### Ingress vs Route: Equivalence

| Ingress Block | Equivalent Route Block |
|--------------|----------------------|
| `ingress.main.hosts[0].host: "dashy.trevorpi.lan"` | `route.main.hostnames: ["dashy.trevorpi.lan"]` |
| `ingress.main.hosts[0].paths[0].path: /` | `route.main.rules[0]` (defaults to Prefix `/` via primary service) |
| `ingress.main.hosts[0].paths[0].pathType: Prefix` | `route.main.rules[0]` (default behavior) |
| `ingress.main.hosts[0].paths[0].service.identifier: main` | `route.main.rules[0].backendRefs[0].identifier: main` (auto-defaulted) |
| `ingress.main.hosts[0].paths[0].service.port: http` | `route.main.rules[0].backendRefs[0].port: http` (auto-defaulted) |

### Per-App Before/After

**dashy (current `ingress` block):**
```yaml
ingress:
  main:
    enabled: true
    hosts:
      - host: ""
        paths:
          - path: /
            pathType: Prefix
      - host: "dashy.trevorpi.lan"
        paths:
          - path: /
            pathType: Prefix
```

**dashy (native `route` block — replaces ingress entirely):**
```yaml
ingress:
  main:
    enabled: false
route:
  main:
    enabled: true
    hostnames:
      - "dashy.trevorpi.lan"
    parentRefs:
      - name: internal
    # No rules needed — auto-defaults to primary service on http port

# Note: the empty-host entry drops off because Gateway API HTTPRoutes
# don't have a direct equivalent. If needed, add a second hostname ""
# or handle via a catch-all rule.
```

**homeassistant:**
```yaml
# Before
    ingress:
      main:
        enabled: true
        hosts:
          - host: home.trevorpi.lan
            paths:
              - path: /
                pathType: Prefix

# After
    ingress:
      main:
        enabled: false
    route:
      main:
        enabled: true
        hostnames:
          - "home.trevorpi.lan"
        parentRefs:
          - name: internal
```

**zwave (uses explicit service port `http` — same as default):**
```yaml
# Before
    ingress:
      main:
        enabled: true
        hosts:
          - host: zwave.trevorpi.lan
            paths:
              - path: /
                pathType: Prefix
                service:
                  port: http

# After
    ingress:
      main:
        enabled: false
    route:
      main:
        enabled: true
        hostnames:
          - "zwave.trevorpi.lan"
        parentRefs:
          - name: internal
```

**glances:**
```yaml
# Before
    ingress:
      main:
        enabled: true
        hosts:
          - host: glances.trevorpi.lan
            paths:
              - path: /
                pathType: Prefix

# After
    ingress:
      main:
        enabled: false
    route:
      main:
        enabled: true
        hostnames:
          - "glances.trevorpi.lan"
        parentRefs:
          - name: internal
```

### Two Approaches: Native Route vs Standalone HTTPRoute

| Factor | Native `route` Block | Standalone `HTTPRoute` YAML |
|--------|---------------------|---------------------------|
| **Location** | Inside `values:` of the HelmRelease | Separate file in `apps/` |
| **Lifecycle** | Tied to HelmRelease (upgrade/rollback together) | Independent resource |
| **Service discovery** | Auto-resolves by `identifier` (references controller) | Must hardcode Service name + port |
| **API version** | Auto-detected from cluster CRDs | Must specify in YAML |
| **Simplicity** | Less YAML, lives in same file as app config | More files, more kustomization entries |
| **Portability** | Tied to bjw-s chart conventions | Vanilla K8s, any controller |

**Recommendation: Use the native `route` block.** It's less code, colocated with the app config, and auto-resolves services. The standalone HTTPRoute approach is the escape hatch if you ever migrate off the bjw-s chart.

## Prerequisites (Cluster Bootstrap)

These are **one-time cluster-admin actions** that are NOT GitOps-managed (they install CRDs into the cluster, not into a namespace):

### 1. Install Gateway API CRDs

```bash
# Option A: Install standard channel CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

# Option B (preferred): Only experimental channel has the CRDs + validation webhook
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/experimental-install.yaml
```

> CRD installation is a bootstrap action. These are cluster-scoped definitions, not application resources. They must exist before any Gateway API objects can be created. We do NOT put these in GitOps (they'd create a chicken-and-egg problem where Flux needs GW API CRDs to apply GW API resources).

### 2. Configure K3s Traefik for Gateway API

K3s configures Traefik via a ConfigMap or Helm values. On k3s, Traefik is deployed as a HelmChart CR in `kube-system`. Enable the Gateway API provider:

**Option A — Patch the existing HelmChartConfig** (if it exists):
```yaml
# /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    providers:
      kubernetesGateway:
        enabled: true
      kubernetesIngress:
        enabled: true   # keep enabled during migration
    experimental:
      kubernetesGateway:
        enabled: true
```

**Option B — K3s server config** (`/etc/rancher/k3s/config.yaml`):
```yaml
k3s:
  traefik:
    extraArgs:
      - "--providers.kubernetesgateway=true"
      - "--experimental.kubernetesgateway=true"
```

> On newer Traefik v3 releases, `kubernetesGateway` may be GA and not require the `experimental` flag. Check `traefik version`.

### 3. Verify

```bash
kubectl get crd | grep gateway.networking.k8s.io
kubectl -n kube-system logs -l app.kubernetes.io/name=traefik | grep -i gateway
```

## GitOps-Managed Resources

All resources below go in the git repo and are applied by Flux.

### New Directory Structure

```
infrastructure/
├── gateway/
│   ├── kustomization.yaml
│   ├── gateway-class.yaml
│   └── gateway.yaml
├── ... existing files ...
apps/
├── dashy/
│   ├── dashy.yaml            # updated: disable ingress
│   ├── httproute.yaml         # NEW
│   ├── ...
├── glances.yaml              # updated: disable ingress
├── glances-httproute.yaml    # NEW
├── homeassistant.yaml        # updated: disable ingress
├── homeassistant-httproute.yaml  # NEW
├── zwave.yaml                # updated: disable ingress
├── zwave-httproute.yaml      # NEW
├── kustomization.yaml        # updated: include httproute files
└── ...
```

### infrastructure/gateway/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gateway-class.yaml
  - gateway.yaml
```

### infrastructure/gateway/gateway-class.yaml

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: traefik
spec:
  controllerName: traefik.io/gateway-controller
```

### infrastructure/gateway/gateway.yaml

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: internal
  namespace: default
spec:
  gatewayClassName: traefik
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
```

> If apps span multiple namespaces, change `from: Same` to `from: All` or use `from: Selector` with labels. For this cluster, everything is in `default`.

### Per-App HTTPRoute (example: dashy)

**apps/dashy/httproute.yaml:**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: dashy
  namespace: default
spec:
  parentRefs:
    - name: internal
      namespace: default
  hostnames:
    - "dashy.trevorpi.lan"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: dashy
          port: 8080
```

> The `backendRefs[0].name` is the Service name. With the bjw-s chart and `service.main.controller: main`, the Service is named `<release-name>` (same as the HelmRelease name) — e.g. `dashy`, `glances`, `homeassistant`, `zwave`.

### Update HelmRelease Values to Disable Ingress

In each app's `dashy.yaml` / `glances.yaml` / etc., change:

```yaml
# Before
    ingress:
      main:
        enabled: true
        hosts:
        ...

# After
    ingress:
      main:
        enabled: false
```

> Optionally, the entire `ingress:` block can be removed or set to `enabled: false`. Keeping it with `enabled: false` is safest — no accidental re-enabling.

### Update apps/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - dashy/
  - glances.yaml
  - glances-httproute.yaml
  - homeassistant.yaml
  - homeassistant-httproute.yaml
  - zwave.yaml
  - zwave-httproute.yaml
```

### Wire Gateway Into infrastructure/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - image-repos
  - image-automation.yaml
  - sources.yaml
  - gateway/
```

## Migration Sequence (Step-by-Step)

### Phase 1: Prepare (No Downtime)

1. **Install Gateway API CRDs** on the cluster (manual — see Prerequisites).
2. **Enable Traefik Gateway API provider** (manual — see Prerequisites).
3. **Verify Traefik logs** — confirm it sees `kubernetesgateway` provider.
4. **Create Git branch** `feat/gateway-api` for all file changes.

### Phase 2: Add Gateway Resources (No Downtime)

1. Create `infrastructure/gateway/` with `kustomization.yaml`, `gateway-class.yaml`, `gateway.yaml`.
2. Wire `infrastructure/kustomization.yaml` to include `gateway/`.
3. Commit → push → Flux reconciles.
4. **Verify**: `kubectl get gatewayclass,gateway -A`
5. Traefik now listens on port 80 via Gateway API **in addition to** the existing Ingress listener. No conflict (same port, same controller, different API objects).

### Phase 3: Swap Ingress → Route in HelmRelease Values (No Downtime)

For each app (one commit = one app, or all at once — your choice):
1. Replace the `ingress:` block with a `route:` block in the HelmRelease values.
2. Commit → push → Flux reconciles.
3. HelmRelease upgrade removes the `Ingress` object and creates the `HTTPRoute` object.
4. **Verify**: `kubectl get httproute -A` and test via `curl -H "Host: <host>" http://<node-ip>`

> The Helm chart replaces the Ingress with an HTTPRoute in a single reconciliation. Traefik picks up the new route immediately. Brief blip possible during the swap, but since both resources target the same backend, any in-flight requests complete normally.

### Phase 4: Cleanup (Optional)

1. After all apps migrated: remove any lingering empty `ingress:` blocks.
2. If desired, disable Traefik's `kubernetesIngress` provider in K3s Traefik config.

### Phase 5: Clean Up (Optional)

1. Remove `ingress:` blocks from HelmRelease values.
2. After all apps migrated: disable Traefik's `kubernetesIngress` provider (K3s Traefik config).
3. No legacy `Ingress` objects remain in the cluster.

## Rollback Plan

If Gateway API has issues:
1. Re-enable `ingress.main.enabled: true` in HelmRelease values (revert the Phase 4 change).
2. Traefik Ingress routes return immediately.
3. HTTPRoutes can stay (they just won't be primary).

## Risks & Considerations

| Risk | Mitigation |
|------|-----------|
| **Gateway API CRD version mismatch** | Pin a specific CRD version. Test on a dev cluster first. |
| **Traefik Gateway API experimental** | Check Traefik v3 release notes — Gateway API may be GA. Fall back to Ingress if unstable. |
| **DNS / mDNS** | Hosts use `.trevorpi.lan` — likely mDNS (Avahi/Bonjour) or local DNS. HTTPRoutes use `hostnames` matching — same as Ingress `hosts`, so no DNS changes needed. |
| **Cert-manager / TLS** | Not currently in use (all plain HTTP). If TLS is added later, Gateway API uses `listeners[].tls` with cert-manager annotations on Gateway. |
| **Flux dependency ordering** | GatewayClass, Gateway, HTTPRoute → Flux applies all in one reconcile. If HTTPRoute applies before Gateway exists, it will retry. No ordering needed (HTTPRoute references Gateway by name — Kubernetes validation handles missing refs gracefully). |
| **BJW-S chart `route` support** | Already present in `common` v5.0.1 (the version used). No chart upgrade needed. The `route` block is production-ready — same template as `ingress` block, different output kind. |

## Future Improvements

- **TLS**: Add `cert-manager` + Let's Encrypt via DNS-01 challenge for internal `.lan` certs (or HTTP-01 if publicly resolvable).
- **Multiple listeners**: Add HTTPS (443) listener with TLS configuration.
- **gRPC routes**: If apps add gRPC, use `GRPCRoute` instead of `HTTPRoute`.
- **Traffic splitting**: Use `backendRefs[].weight` for canary deployments.
