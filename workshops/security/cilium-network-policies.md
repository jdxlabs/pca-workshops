# Visualizing Network Policies with Cilium & Hubble

> **Category:** Security · **Level:** Intermediate · **Duration:** ~20 min · **Cost:** Free (fully local, no cloud resources)

## Exam relevance

The PCA exam expects you to distinguish Kubernetes **NetworkPolicies** (L3/L4 firewalling between pods) from application-layer traffic control (covered in the Service Mesh workshop). This lab makes that boundary visible instead of theoretical.

## Objective

Run a local Kubernetes cluster with Cilium as the CNI, then watch — in real time via Hubble — a NetworkPolicy block traffic between two pods.

## Prerequisites

- [Kind](https://kind.sigs.k8s.io/) or Minikube installed locally
- `kubectl`, `cilium` CLI, and `hubble` CLI installed

## Steps

### 1. Create a Kind cluster without the default CNI

```bash
kind create cluster --config=- <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
EOF
```

### 2. Install Cilium as the CNI

```bash
cilium install
cilium status --wait
```

### 3. Enable and open Hubble (traffic observability)

```bash
cilium hubble enable
cilium hubble port-forward &
hubble observe --follow
```

Leave this running in a separate terminal — it streams every allowed/dropped packet.

### 4. Deploy a frontend and backend pod

```bash
kubectl create deployment backend --image=hashicorp/http-echo -- -text="hello from backend"
kubectl expose deployment backend --port=5678
kubectl run frontend --image=curlimages/curl --command -- sleep 3600
```

### 5. Confirm traffic flows before any policy

```bash
kubectl exec -it frontend -- curl -s backend:5678
```

You should see `hello from backend`, and a corresponding **allowed** flow in the Hubble output.

### 6. Apply a NetworkPolicy that blocks frontend → backend

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress: []
```

```bash
kubectl apply -f deny-frontend-to-backend.yaml
```

### 7. Retry the curl and watch Hubble

```bash
kubectl exec -it frontend -- curl -s --max-time 3 backend:5678
```

The request times out, and Hubble shows the flow as **DROPPED** in real time.

## Cleanup

```bash
kind delete cluster
```

## Key takeaways

- NetworkPolicies operate at L3/L4 — they know about IPs and ports, not HTTP paths or traffic percentages.
- Hubble turns an abstract policy into an observable, real-time decision — invaluable for both learning and production debugging.
- Contrast this with the Service Mesh workshop, where routing decisions happen at L7.
