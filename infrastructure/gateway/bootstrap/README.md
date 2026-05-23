# Bootstrap Prerequisites for Gateway API on K3s

These are **one-time cluster-admin actions** that must be done before Flux can manage Gateway API resources.
They live in the repo for documentation and reproducibility — not applied by Flux GitOps.

## 1. Gateway API CRDs

CRDs are already installed on the cluster (as of 2026-05-23).

To install/reinstall:
```bash
# Standard channel (GatewayClass, Gateway, HTTPRoute)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

# Or experimental (includes validation webhook)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/experimental-install.yaml
```

Verify:
```bash
kubectl get crd | grep gateway.networking.k8s.io
```

## 2. Enable Traefik Gateway API Provider

K3s deploys Traefik via a HelmChart in `kube-system`. To enable the Gateway API provider, apply the HelmChartConfig:

```bash
kubectl apply -f infrastructure/gateway/bootstrap/traefik-helmchartconfig.yaml
```

This adds `kubernetesGateway` provider alongside the existing `kubernetesIngress` and `kubernetesCRD` providers. Both Ingress and Gateway API work in parallel during migration.

> **K3s auto-apply note**: For persistence across node reboots, also place this file at
> `/var/lib/rancher/k3s/server/manifests/traefik-helmchartconfig.yaml` on the node.

Verify:
```bash
kubectl -n kube-system logs -l app.kubernetes.io/name=traefik | grep -i "kubernetesgateway\|gateway.*provider"
kubectl -n kube-system get deployment traefik -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | grep gateway
```

## 3. Verify Gateway API Readiness

```bash
kubectl get gatewayclass,gateway -A
kubectl get httproute -A
```

## Cleanup of Half-Baked Install

If any Gateway API objects were manually created before this GitOps migration:
```bash
# Check for manually-created objects
kubectl get gatewayclass,gateway,httproute -A

# Delete any that conflict with GitOps-managed resources
kubectl delete gatewayclass <name>
kubectl delete gateway <name> -n <namespace>
kubectl delete httproute <name> -n <namespace>
```
