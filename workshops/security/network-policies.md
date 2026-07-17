# Kubernetes Network Policies (GKE Autopilot or local k3s)

> **Category:** Security · **Level:** Beginner · **Duration:** ~15 min · **Cost:** Free tier (GKE) or free (local)

## Exam relevance

The exam expects you to know that NetworkPolicies are an L3/L4 firewall *inside* a cluster (pod-to-pod), and that GKE enforces them via **Dataplane V2** — GKE's default networking engine, itself built on Cilium/eBPF. On GKE Autopilot, Cilium is fully managed and its CLI is inaccessible (no node access, no privileged DaemonSets) — the `NetworkPolicy` YAML is all you get to touch. Running Cilium yourself locally lets you go one level deeper and actually drive it with the real `cilium`/`hubble` CLIs.

## Objective

Confirm two pods can talk freely by default, then apply a NetworkPolicy that blocks traffic and re-open it with a scoped label selector — while watching the real enforcement engine underneath.

## Prerequisites

Pick **one** of the two options below — everything after step 1 is identical either way.

- **Option B (local) only:** [`cilium` CLI](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli) and [`hubble` CLI](https://docs.cilium.io/en/stable/gettingstarted/hubble_setup/) installed locally.

## Steps

### 1. Create your cluster

**Option A — GKE Autopilot**

```bash
gcloud container clusters create-auto pca-cluster --region=europe-west9
gcloud container clusters get-credentials pca-cluster --region=europe-west9
```

Autopilot enforces NetworkPolicies through Dataplane V2 automatically — nothing to install, and no CLI access to the underlying Cilium agent (it lives in a Google-managed `kube-system` you can't exec into).

**Option B — local k3s + real Cilium**

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --flannel-backend=none \
  --disable-network-policy \
  --disable=traefik

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

cilium install
cilium status --wait
```

This installs the same Cilium engine GKE runs under Dataplane V2 — except here you have full CLI access to it.

### 2. Deploy a backend and a frontend pod

```bash
kubectl create deployment backend --image=hashicorp/http-echo -- -text="hello from backend"
kubectl expose deployment backend --port=5678
kubectl run frontend --image=curlimages/curl --command -- sleep 3600
```

### 3. Confirm traffic flows before any policy

```bash
kubectl exec -it frontend -- curl -s --max-time 3 backend:5678
```

You should see `hello from backend` — by default, every pod can reach every other pod.

### 4. Apply a NetworkPolicy that blocks all ingress to the backend

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

### 5. Retry the curl

```bash
kubectl exec -it frontend -- curl -s --max-time 3 backend:5678
```

The request now times out — Cilium is enforcing the policy at the packet level, silently dropping traffic to `backend` (whether it's the GKE-managed instance under Dataplane V2, or your own local one).

**Option B only — watch it live with Hubble:**

```bash
cilium hubble enable
cilium hubble port-forward &
hubble observe --pod frontend --follow
```

Re-run the `curl` from above in another terminal: Hubble shows the flow as **DROPPED** in real time — something GKE Autopilot never lets you see directly.

### 6. Scope the policy down instead of blocking everything

Replace the empty `ingress: []` with a rule that only allows a specific label:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-only-trusted
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: trusted
```

```bash
kubectl delete networkpolicy deny-frontend-to-backend
kubectl apply -f allow-only-trusted.yaml
kubectl label pod frontend role=trusted
kubectl exec -it frontend -- curl -s --max-time 3 backend:5678
```

Traffic is allowed again — this time because `frontend` carries the `role: trusted` label the policy selects for.

## Cleanup

```bash
kubectl delete networkpolicy allow-only-trusted
kubectl delete deployment backend
kubectl delete pod frontend
kubectl delete service backend
```

Then tear down whichever cluster you used:

```bash
# Option A — GKE Autopilot
gcloud container clusters delete pca-cluster --region=europe-west9 --quiet

# Option B — local k3s (also stop the Hubble port-forward if you left it running)
kill %1 2>/dev/null
/usr/local/bin/k3s-uninstall.sh
```

## Key takeaways

- NetworkPolicies operate at L3/L4 (IPs, ports) — they know nothing about HTTP paths or traffic percentages; that's the Service Mesh's job.
- GKE enforces NetworkPolicies through **Dataplane V2**, which is Cilium under the hood — this is why the exam bullet "GKE Dataplane V2, eBPF, high performance L3/L4" points straight to Cilium.
- On GKE Autopilot, Cilium is entirely Google-managed and its CLI is off-limits; running it yourself locally (Option B) is the only way to actually drive `cilium`/`hubble` commands and watch drops happen live.
- An empty `ingress: []` list means "deny all ingress"; adding `from` selectors narrows it back down to exactly the traffic you want.
