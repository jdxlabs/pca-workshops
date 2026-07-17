# GKE Workload Identity: KSA ↔ GSA Binding

> **Category:** Security · **Level:** Intermediate · **Duration:** ~30 min · **Cost:** Free tier (GKE Autopilot monthly credit covers a small test cluster)

## Exam relevance

Workload Identity is the boundary between Kubernetes RBAC and Google Cloud IAM — arguably the single most exam-relevant security mechanism for GKE. Expect scenario questions that test whether you know how a Kubernetes Service Account (KSA) securely "becomes" a Google Service Account (GSA) without any downloaded JSON key.

## Objective

Bind a Kubernetes Service Account to a Google Service Account via Workload Identity, then prove a pod can call a Google Cloud API using that identity with **zero exported credentials**.

> This workshop is **GKE-only** — unlike the other K8s workshops in this repo, it has no local k3s equivalent. Workload Identity works by having the pod talk to Google's metadata server to exchange its Kubernetes identity for a short-lived GCP token; a local k3s cluster has no such metadata server, so there's nothing to bind to. Simulating it with a mounted key file would defeat the entire point of the exercise (no exported credentials).

## Prerequisites

- A GCP project with the GKE API enabled
- A Cloud Storage bucket to test read access against

## Steps

### 1. Create a GKE Autopilot cluster

```bash
gcloud components install gke-gcloud-auth-plugin
gcloud components install kubectl

gcloud container clusters create-auto pca-cluster --region=europe-west9
gcloud container clusters get-credentials pca-cluster --region=europe-west9
```

Autopilot enables Workload Identity by default — no extra flag needed.

### 2. Create the Google Service Account (GSA)

```bash
gcloud iam service-accounts create mon-gsa-test
```

Grant it a minimal role, e.g. read-only on a specific bucket:

```bash
gcloud storage buckets add-iam-policy-binding gs://YOUR_BUCKET \
  --member=serviceAccount:mon-gsa-test@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer
```

### 3. Create the Kubernetes Service Account (KSA)

```bash
kubectl create namespace test-ns
kubectl create serviceaccount mon-ksa-test -n test-ns
```

### 4. Bind KSA to GSA (Workload Identity)

```bash
gcloud iam service-accounts add-iam-policy-binding \
  mon-gsa-test@PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[test-ns/mon-ksa-test]"
```

### 5. Annotate the KSA

```bash
kubectl annotate serviceaccount mon-ksa-test -n test-ns \
  iam.gke.io/gcp-service-account=mon-gsa-test@PROJECT_ID.iam.gserviceaccount.com
```

### 6. Deploy a test pod using the KSA

```bash
kubectl run wi-test --image=google/cloud-sdk:slim -n test-ns \
  --overrides='{"spec": {"serviceAccountName": "mon-ksa-test"}}' \
  --command -- sleep 3600
```

### 7. Verify the borrowed identity, no key file involved

```bash
kubectl exec -it wi-test -n test-ns -- gcloud storage ls gs://YOUR_BUCKET
```

The listing succeeds — the pod authenticated as `mon-gsa-test`, entirely through metadata-server token exchange.

## Cleanup

```bash
kubectl delete namespace test-ns
gcloud iam service-accounts delete mon-gsa-test@PROJECT_ID.iam.gserviceaccount.com --quiet
gcloud container clusters delete pca-cluster --region=europe-west9 --quiet
```

## Key takeaways

- Workload Identity replaces service account key files — a major attack surface — with short-lived, automatically rotated tokens.
- The binding is two-directional: an IAM policy binding (`roles/iam.workloadIdentityUser`) on the GSA side, and an annotation on the KSA side.
- On the exam, "no service account keys" + "GKE" + "least privilege" is a strong signal to reach for Workload Identity.
