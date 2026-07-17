# Traffic Splitting with Istio / Cloud Service Mesh (GKE Autopilot or local k3s)

> **Category:** Service Mesh · **Level:** Intermediate · **Duration:** ~25 min · **Cost:** Free tier (GKE) or free (local)

## Exam relevance

Cloud Service Mesh (CSM) questions on the PCA exam revolve around `VirtualService` and `DestinationRule` — canary releases, weighted traffic splits, and L7 routing. CSM is Google's **managed control plane for Istio** — same CRDs, same behavior, Google just runs the control plane for you on GKE. Running open-source Istio locally on k3s teaches the exact same primitives for free.

## Objective

Deploy a multi-version microservice app on a mesh, and split traffic between two versions by weighted percentage — entirely at the application layer (L7), no NetworkPolicy involved.

## Prerequisites

- [`istioctl`](https://istio.io/latest/docs/setup/getting-started/) installed locally either way — it's used to install Istio on k3s (Option B), and to verify the control plane on both options.
- Pick **one** of the two options below — from step 2 onward everything is identical.
  - **Option A (GKE):** enabling the Fleet/Mesh feature requires `roles/gkehub.admin`-level permissions. If your sandbox project doesn't allow it, ask your ESN's platform team to enable it once — it's a one-time, project-level toggle.
  - **Option B (local):** no GCP dependency at all.

## Steps

### 1. Create your cluster and mesh

**Option A — GKE Autopilot + managed Cloud Service Mesh**

```bash
gcloud container clusters create-auto pca-cluster --region=europe-west9
gcloud container clusters get-credentials pca-cluster --region=europe-west9

gcloud services enable mesh.googleapis.com
gcloud container fleet mesh enable
gcloud container fleet memberships register pca-cluster \
  --gke-cluster=europe-west9/pca-cluster \
  --enable-workload-identity

# Turn on the managed control plane for this specific membership —
# enabling the API alone does not activate it
gcloud container fleet mesh update \
  --management=automatic \
  --memberships=pca-cluster

kubectl label namespace default istio.io/rev=asm-managed --overwrite
```

Provisioning the managed control plane can take several minutes. Check progress with:

```bash
gcloud container fleet mesh describe
```

Wait until the membership shows `state: ACTIVE` before moving on.

**Option B — local k3s + self-managed Istio**

```bash
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled --overwrite
```

Either way, the `default` namespace is now labeled for automatic Envoy sidecar injection.

### 2. Deploy the Bookinfo sample app

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/platform/kube/bookinfo.yaml
```

This deploys four microservices (`productpage`, `details`, `reviews`, `ratings`), with `reviews` having three versions:

- `v1` — no star ratings
- `v2` — black star ratings
- `v3` — red star ratings

### 3. Confirm the mesh is injecting sidecars

```bash
kubectl get pods
```

Each `reviews-*` pod should show `2/2` containers ready — your app container plus the injected Envoy sidecar.

Cross-check with the actual Istio control plane, using `istioctl` (works against both the managed CSM control plane on GKE and the self-managed one on k3s, as long as your `istioctl` version is close to the cluster's):

```bash
istioctl proxy-status
```

Every `reviews-*` and `productpage-*` proxy should show `SYNCED` — confirming Envoy has received its routing configuration from the control plane.

### 4. Apply a DestinationRule defining the subsets

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/networking/destination-rule-reviews.yaml
```

### 5. Apply a VirtualService to split traffic 90/10

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

### 6. Observe the split from inside the cluster

No need for an ingress gateway or port-forward — call `productpage` directly from a throwaway pod and count how often each `reviews` version answers:

```bash
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- \
  sh -c 'for i in $(seq 1 20); do curl -s productpage:9080/productpage | grep -o "glyphicon-star\|reviews-v[0-9]" ; done'
```

Roughly 9 out of 10 iterations show no stars (`v1`); about 1 in 10 shows the red-star markup (`v3`).

## Cleanup

```bash
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl delete -f reviews-90-10.yaml
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/networking/destination-rule-reviews.yaml
```

Then tear down whichever cluster you used:

```bash
# Option A — GKE Autopilot
gcloud container fleet memberships unregister pca-cluster --gke-cluster=europe-west9/pca-cluster --quiet
gcloud container clusters delete pca-cluster --region=europe-west9 --quiet

# Option B — local k3s
/usr/local/bin/k3s-uninstall.sh
```

## Key takeaways

- `DestinationRule` defines *which versions exist* (subsets); `VirtualService` defines *how traffic is routed* between them.
- This is L7 routing based on percentages and application semantics — fundamentally different from a NetworkPolicy's binary allow/deny at L3/L4.
- On the exam, "canary rollout", "weighted traffic split", or "A/B test at the routing layer" all point to Cloud Service Mesh, not NetworkPolicies.
- Cloud Service Mesh **is** Istio (Envoy sidecars + `VirtualService`/`DestinationRule` CRDs) with Google managing the control plane — the CRDs and concepts transfer directly between the managed (GKE) and self-hosted (local) versions.
