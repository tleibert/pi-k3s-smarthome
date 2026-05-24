# pi-k3s-smarthome

Single-node K3s cluster on a Raspberry Pi running home automation services. Fully GitOps-managed via Flux CD — the repo is the source of truth.

All apps are accessible over HTTPS with automatic TLS (cert-manager + Let's Encrypt) and automatic DNS (external-dns + Cloudflare).

## Apps

| App | Host | Port | TLS |
|-----|------|------|-----|
| [Dashy](https://dashy.to/) | `dashy.trevorleibert.com` | 8080 | ✅ |
| [Glances](https://nicolargo.github.io/glances/) | `glances.trevorleibert.com` | 61208 | ✅ |
| [Home Assistant](https://www.home-assistant.io/) | `ha.trevorleibert.com` | 8123 | ✅ |
| [Z-Wave JS UI](https://zwave-js.github.io/zwave-js-ui/) | `zwave.trevorleibert.com` | 8091 | ✅ |

All apps use the [bjw-s app-template](https://github.com/bjw-s/helm-charts) Helm chart with native Gateway API `route` blocks.

## Architecture

```
GitHub (this repo)
  ↓ Flux CD (every 10 min)
K3s on Raspberry Pi (arm64, Debian)
  ├── Traefik (built-in, Gateway API provider)
  │   ├── Gateway "internal" :8000 (HTTP→HTTPS 301 redirect) + :8443 (HTTPS)
  │   │   ├── TLS via cert-manager (Let's Encrypt, Cloudflare DNS-01)
  │   │   └── Auto-renewed certificates
  │   ├── HTTPRoute: https-redirect (port 8000) + dashy, glances, ha, zwave (port 8443)
  │   └── GatewayClass: traefik
  ├── cert-manager (Let's Encrypt TLS via Cloudflare DNS-01)
  ├── external-dns (watches HTTPRoutes, creates Cloudflare A records)
  ├── Flux controllers (kustomize, helm, source, notification)
  ├── image-automation (auto-updates container tags)
  └── DNS: A records created automatically by external-dns → 192.168.4.20
```

## Bootstrap (Fresh Pi)

Only **5 manual steps** — everything else is GitOps:

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
Required by cert-manager (DNS-01 challenges) and external-dns (creating A records).
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

### What happens next (automatic)

external-dns watches your HTTPRoutes and creates the corresponding Cloudflare A records
automatically. No manual DNS setup needed — add a new app with a hostname and the record
appears within ~1 minute.

### What happens next

Flux syncs and applies resources in **guaranteed order** via 5 Kustomizations with `dependsOn`:

```
                        gateway-crds
                              │
             ┌────────────────┼────────────────┐
             ↓                ↓                ↓
    cert-manager-install   external-dns   infrastructure
             │                               │
             ↓                               ↓
       cert-manager                        apps
```

**Parallel tracks after `gateway-crds`:**
- **cert-manager-install** → installs cert-manager Helm chart (registers CRDs)
- **cert-manager** → creates ClusterIssuer + Certificate (SANs for all 4 hostnames)
- **external-dns** → installs external-dns, starts watching HTTPRoutes
- **infrastructure** → HelmRepos, Traefik Gateway provider, Gateway (HTTP→HTTPS redirect + TLS)
- **apps** → waits for infrastructure, then installs app HelmReleases + HTTPRoutes
- **DNS records** → external-dns sees HTTPRoutes and creates A records in Cloudflare

All apps are **HTTPS-only** — the HTTP listener on port 8000 serves a 301 redirect to HTTPS.

The certificate Secret (`home-tls`) is created by cert-manager after Let's Encrypt issues the
cert (DNS-01 challenge via Cloudflare). Traefik picks it up without restart and auto-renews
~30 days before expiry.

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
│   ├── cert-manager/        #   cert-manager resources
│   │   ├── install/         #     Namespace + HelmRelease (pinned to v1.17.1)
│   │   ├── cluster-issuer.yaml
│   │   └── certificate.yaml
│   ├── external-dns/        #   external-dns HelmRelease (Cloudflare provider)
│   ├── image-repos/         #   Image repository references
│   ├── image-automation.yaml
│   ├── sources.yaml         #   HelmRepositories (bjw-s, jetstack, kubernetes-sigs)
│   └── kustomization.yaml
└── docs/                    # Planning documents
```

## What's managed outside this repo

- **DNS**: A records are created automatically by external-dns from HTTPRoute hostnames. Manual records only needed if you want to pre-create them before external-dns starts.
- **TLS automation**: cert-manager handles Let's Encrypt renewal automatically (~30 days before expiry)
- **Cloudflare API token**: Created manually in step 4, stored as a Kubernetes Secret
- **Static IP**: `192.168.4.20` — configure via DHCP reservation or `/etc/network/interfaces`
- **Z-Wave USB radio**: Path `/dev/serial/by-id/usb-1a86_USB_Single_Serial_58E3038596-if00` is hardware-specific; update `apps/zwave.yaml` for a different Pi/radio
- **K3s itself**: Installed once, self-updating via built-in upgrade controller
- **mDNS / Avahi**: `.trevorpi.lan` still resolves locally if needed, but the primary domains are now `*.trevorleibert.com`

## Daily Operations

- **Image updates**: Flux image-automation scans for new container tags, commits updates to the repo
- **App config changes**: Edit the HelmRelease values in `apps/`, commit, push — Flux applies within 10 minutes
- **Drift detection**: Enabled on all HelmReleases — Flux reverts manual changes automatically
