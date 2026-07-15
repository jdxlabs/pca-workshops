# Traffic Splitting with Istio (Cloud Service Mesh, Local Edition)

> **Category:** Service Mesh · **Level:** Intermediate · **Duration:** ~25 min · **Cost:** Free (fully local, no cloud resources)

## Exam relevance

Cloud Service Mesh (CSM) questions on the PCA exam revolve around `VirtualService` and `DestinationRule` concepts — canary releases, weighted traffic splits, and L7 routing. Istio is CSM's open-source twin, so running it locally teaches the exact same primitives for free.

## Objective

Deploy a multi-version microservice app and split traffic between two versions by weighted percentage, entirely at the application layer (L7) — no NetworkPolicy involved.

## Prerequisites

- A Linux host or VM to install [k3s](https://k3s.io/) on
- `istioctl` installed

## Steps

### 1. Create a local cluster

```bash
curl -sfL https://get.k3s.io | sh -

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

### 2. Install Istio with the demo profile

```bash
istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled
```

### 3. Deploy the Bookinfo sample app

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/platform/kube/bookinfo.yaml
```

This deploys four microservices (`productpage`, `details`, `reviews`, `ratings`), with `reviews` having three versions:

- `v1` — no star ratings
- `v2` — black star ratings
- `v3` — red star ratings

### 4. Expose the app via the Istio ingress gateway

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```

Open `http://localhost:8080/productpage` and refresh a few times — ratings currently look random/inconsistent because Kubernetes load-balances across all three `reviews` versions equally.

### 5. Apply a DestinationRule defining the subsets

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/networking/destination-rule-reviews.yaml
```

### 6. Apply a VirtualService to split traffic 90/10

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v3
      weight: 10
```

```bash
kubectl apply -f reviews-90-10.yaml
```

### 7. Observe the result

Refresh `http://localhost:8080/productpage` about 20 times. Roughly 9 out of 10 refreshes show no stars (`v1`); about 1 in 10 shows red stars (`v3`).

## Cleanup

```bash
/usr/local/bin/k3s-uninstall.sh
```

## Key takeaways

- `DestinationRule` defines *which versions exist* (subsets); `VirtualService` defines *how traffic is routed* between them.
- This is L7 routing based on percentages and application semantics — fundamentally different from a NetworkPolicy's binary allow/deny at L3/L4.
- On the exam, "canary rollout", "weighted traffic split", or "A/B test at the routing layer" all point to Cloud Service Mesh, not NetworkPolicies.