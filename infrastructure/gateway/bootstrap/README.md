# Bootstrap Prerequisites

These are the **only manual steps** needed on a fresh Raspberry Pi. Everything else is GitOps-managed via Flux.

## 1. OS + K3s

Install K3s on a Debian-based arm64 Pi:
```bash
curl -sfL https://get.k3s.io | sh -
```
This gives you a single-node cluster with built-in Traefik.

## 2. Node Labels

Required by homeassistant (`zigbee=true`) and zwave (`zwave=true`) for pod scheduling:
```bash
kubectl label node $(hostname) zigbee=true zwave=true
```

## 3. Host Paths

Create directories for persistence mounts:
```bash
mkdir -p /root/container-data/homeassistant
mkdir -p /root/container-data/zwave-ui
```

> The zwave USB device path (`/dev/serial/by-id/usb-1a86_USB_Single_Serial_58E3038596-if00`) is hardware-specific and may need updating in `apps/zwave.yaml` for a different Pi/radio.

## 4. Flux Bootstrap

```bash
flux bootstrap github \
  --owner=tleibert \
  --repository=pi-k3s-smarthome \
  --branch=main \
  --path=./clusters/my-pi \
  --personal
```

This installs Flux controllers and creates a deploy key in the GitHub repo.

## After Flux Syncs

Flux automatically applies (in order of the kustomization):
1. **Gateway API CRDs** — `infrastructure/gateway/crds/`
2. **Traefik HelmChartConfig** — enables the Gateway API provider in K3s Traefik
3. **GatewayClass + Gateway** — the Traefik Gateway API listener
4. **All 4 apps** — dashy, glances, homeassistant, zwave (HelmReleases create deployments, services, and HTTPRoutes)

> Note: On the very first reconciliation, Gateway API objects may briefly fail while CRDs register. Flux retries every 10 minutes — everything converges within 1-2 cycles.

## What's NOT in this repo

- **mDNS / Avahi**: `.trevorpi.lan` resolution is handled by Avahi on the Pi (zero config on Debian). The Pi must have hostname `trevorpi`.
- **Static IP**: The Pi uses `192.168.4.20` (seen in Gateway status). Configure via `/etc/network/interfaces` or DHCP reservation.
- **ZWave USB radio**: The ZWave JS UI pod needs `/dev/serial/by-id/usb-1a86_USB_Single_Serial_58E3038596-if00` to exist (a physical Z-Wave USB stick).
