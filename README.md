# pi-k3s-smarthome

Single-node K3s cluster on a Raspberry Pi running home automation services. Fully GitOps-managed via Flux CD — the repo is the source of truth.

All apps are accessible at `*.home.trevorleibert.com` with automatic TLS via cert-manager + Let's Encrypt.

## Apps

| App | Host | Port | TLS |
|-----|------|------|-----|
| [Dashy](https://dashy.to/) | `home.trevorleibert.com` / `dashy.home.trevorleibert.com` | 8080 | ✅ |
| [Glances](https://nicolargo.github.io/glances/) | `glances.home.trevorleibert.com` | 61208 | ✅ |
| [Home Assistant](https://www.home-assistant.io/) | `ha.home.trevorleibert.com` | 8123 | ✅ |
| [Z-Wave JS UI](https://zwave-js.github.io/zwave-js-ui/) | `zwave.home.trevorleibert.com` | 8091 | ✅ |

All apps use the [bjw-s app-template](https://github.com/bjw-s/helm-charts) Helm chart with native Gateway API `route` blocks.

## Architecture

```
GitHub (this repo)
  ↓ Flux CD (every 10 min)
K3s on Raspberry Pi (arm64, Debian)
  ├── Traefik (built-in, Gateway API provider)
  │   ├── Gateway "internal" :8000 (HTTP) + :8443 (HTTPS)
  │   │   ├── TLS via cert-manager (Let's Encrypt, Cloudflare DNS-01)
  │   │   └── Auto-renewed certificates
  │   ├── HTTPRoute: dashy, glances, ha, zwave
  │   └── GatewayClass: traefik
  ├── cert-manager
  ├── Flux controllers (kustomize, helm, source, notification)
  ├── image-automation (auto-updates container tags)
  └── DNS: *.home.trevorleibert.com → 192.168.4.20 (Cloudflare, grey cloud)
```

## Bootstrap (Fresh Pi)

Only **6 manual steps** — everything else is GitOps:

### 1. Install K3s
```bash
curl -sfL https://get.k3s.io | sh -
```
Single-node cluster with built-in Traefik (v3, Gateway API support).

### 2. Label the node
```bash
kubectl label node $(hostname) zigbee=true zwave=true
```
Required by Home Assistant (`zigbee`) and Z-Wave JS UI (`zwave`) for pod scheduling.

### 3. Create host paths
```bash
mkdir -p /root/container-data/homeassistant
mkdir -p /root/container-data/zwave-ui
```
Persistence mounts for Home Assistant config and Z-Wave JS UI store.

### 4. Create Cloudflare API token secret
```bash
kubectl create namespace cert-manager
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token=<your-token>
```
Required by cert-manager to complete DNS-01 challenges with Let's Encrypt.
Token needs **Zone:DNS:Edit** permission for `trevorleibert.com`.
Create one at https://dash.cloudflare.com/profile/api-tokens.

### 5. Bootstrap Flux
```bash
flux bootstrap github \
  --owner=tleibert \
  --repository=pi-k3s-smarthome \
  --branch=main \
  --path=./clusters/my-pi \
  --personal
```

### 6. Set up DNS (Cloudflare)
| Record | Type | Value | Proxy |
|--------|------|-------|-------|
| `home.trevorleibert.com` | A | `192.168.4.20` | Grey cloud (DNS only) |
| `*.home.trevorleibert.com` | A | `192.168.4.20` | Grey cloud (DNS only) |

### What happens next

Flux syncs and applies resources in **guaranteed order** via 3 Kustomizations with `dependsOn`:

```
gateway-crds    fetches Gateway API CRDs from upstream release URL
    ↓ dependsOn
infrastructure  HelmRepository (bjw-s, jetstack), Traefik Gateway provider,
                Gateway (HTTP + HTTPS), cert-manager, Let's Encrypt ClusterIssuer,
                Wildcard Certificate
    ↓ dependsOn
apps            dashy, glances, homeassistant, zwave (HelmReleases + HTTPRoutes)
```

cert-manager will automatically request a wildcard certificate for `*.home.trevorleibert.com`
on its first reconcile. The certificate Secret (`home-tls`) is referenced by the Gateway's
TLS listener — Traefik picks it up without restart whenever it's renewed.

Everything converges in a **single reconciliation pass** — no retry cycles.

## Repo Structure

```
.
├── apps/                    # Application HelmReleases
│   ├── dashy/               #   Dashy dashboard (includes ConfigMap)
│   ├── glances.yaml
│   ├── homeassistant.yaml
│   ├── zwave.yaml
│   └── kustomization.yaml
├── clusters/
│   └── my-pi/
│       └── flux-system/     # Flux Kustomization resources + bootstrap manifests
├── infrastructure/          # Cluster-wide resources
│   ├── gateway/             #   Gateway, Traefik HelmChartConfig
│   │   └── crds/            #   Gateway API CRDs (fetched from upstream URL)
│   ├── cert-manager/        #   cert-manager HelmRelease, ClusterIssuer, Certificate
│   ├── image-repos/         #   Image repository references
│   ├── image-automation.yaml
│   ├── sources.yaml         #   HelmRepositories (bjw-s, jetstack)
│   └── kustomization.yaml
└── docs/                    # Planning documents
```

## What's managed outside this repo

- **DNS**: Cloudflare A records for `*.home.trevorleibert.com` → LAN IP — grey cloud (DNS only)
- **TLS automation**: cert-manager handles Let's Encrypt renewal automatically (~30 days before expiry)
- **Cloudflare API token**: Created manually in step 4, stored as a Kubernetes Secret
- **Static IP**: `192.168.4.20` — configure via DHCP reservation or `/etc/network/interfaces`
- **Z-Wave USB radio**: Path `/dev/serial/by-id/usb-1a86_USB_Single_Serial_58E3038596-if00` is hardware-specific; update `apps/zwave.yaml` for a different Pi/radio
- **K3s itself**: Installed once, self-updating via built-in upgrade controller
- **mDNS / Avahi**: `.trevorpi.lan` still resolves locally if needed, but the primary domain is now `home.trevorleibert.com`

## Daily Operations

- **Image updates**: Flux image-automation scans for new container tags, commits updates to the repo
- **App config changes**: Edit the HelmRelease values in `apps/`, commit, push — Flux applies within 10 minutes
- **Drift detection**: Enabled on all HelmReleases — Flux reverts manual changes automatically
