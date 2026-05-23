# pi-k3s-smarthome

Single-node K3s cluster on a Raspberry Pi running home automation services. Fully GitOps-managed via Flux CD — the repo is the source of truth.

## Apps

| App | Host | Port | Type |
|-----|------|------|------|
| [Dashy](https://dashy.to/) | `trevorpi.lan` / `dashy.trevorpi.lan` | 8080 | Dashboard |
| [Glances](https://nicolargo.github.io/glances/) | `glances.trevorpi.lan` | 61208 | System monitoring |
| [Home Assistant](https://www.home-assistant.io/) | `home.trevorpi.lan` | 8123 | Home automation |
| [Z-Wave JS UI](https://zwave-js.github.io/zwave-js-ui/) | `zwave.trevorpi.lan` | 8091 | Z-Wave controller |

All apps use the [bjw-s app-template](https://github.com/bjw-s/helm-charts) Helm chart with native Gateway API `route` blocks.

## Architecture

```
GitHub (this repo)
  ↓ Flux CD (every 10 min)
K3s on Raspberry Pi (arm64, Debian)
  ├── Traefik (built-in, Gateway API provider)
  │   ├── Gateway "internal" :8000  ← K3s maps 80→8000 via iptables
  │   ├── HTTPRoute: dashy, glances, homeassistant, zwave
  │   └── GatewayClass: traefik
  ├── Flux controllers (kustomize, helm, source, notification)
  ├── image-automation (auto-updates container tags)
  └── mDNS (Avahi) → .trevorpi.lan resolution
```

## Bootstrap (Fresh Pi)

Only **4 manual steps** — everything else is GitOps:

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

### 4. Bootstrap Flux
```bash
flux bootstrap github \
  --owner=tleibert \
  --repository=pi-k3s-smarthome \
  --branch=main \
  --path=./clusters/my-pi \
  --personal
```

### What happens next

Flux syncs and applies resources in **guaranteed order** via 3 Kustomizations with `dependsOn`:

```
gateway-crds    fetches Gateway API CRDs from upstream release URL
    ↓ dependsOn
infrastructure  HelmRepository (bjw-s), Traefik Gateway provider, GatewayClass, Gateway
    ↓ dependsOn
apps            dashy, glances, homeassistant, zwave (HelmReleases + HTTPRoutes)
```

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
│   ├── gateway/             #   GatewayClass, Gateway, Traefik HelmChartConfig
│   │   └── crds/            #   Gateway API CRDs (fetched from upstream URL)
│   ├── image-repos/         #   Image repository references
│   ├── image-automation.yaml
│   ├── sources.yaml         #   HelmRepository (bjw-s)
│   └── kustomization.yaml
└── docs/                    # Planning documents
```

## What's managed outside this repo

- **mDNS / Avahi**: `.trevorpi.lan` resolution — zero config on Debian, hostname must be `trevorpi`
- **Static IP**: `192.168.4.20` — configure via DHCP reservation or `/etc/network/interfaces`
- **Z-Wave USB radio**: Path `/dev/serial/by-id/usb-1a86_USB_Single_Serial_58E3038596-if00` is hardware-specific; update `apps/zwave.yaml` for a different Pi/radio
- **K3s itself**: Installed once, self-updating via built-in upgrade controller

## Daily Operations

- **Image updates**: Flux image-automation scans for new container tags, commits updates to the repo
- **App config changes**: Edit the HelmRelease values in `apps/`, commit, push — Flux applies within 10 minutes
- **Drift detection**: Enabled on all HelmReleases — Flux reverts manual changes automatically
