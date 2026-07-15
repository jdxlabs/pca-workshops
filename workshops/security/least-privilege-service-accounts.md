# IAM Least Privilege: Scoping a Service Account to One Bucket

> **Category:** Security · **Level:** Beginner · **Duration:** ~15 min · **Cost:** Free tier

## Exam relevance

"Least privilege" is a phrase the exam uses constantly but rarely proves. This workshop makes the boundary concrete: a service account that can touch exactly one resource, and gets a `403` on everything else.

## Objective

Attach a dedicated service account with a single, narrow IAM binding to a VM, and confirm it can access one bucket but is denied on another.

## Prerequisites

- A GCP project with Compute Engine and Cloud Storage APIs enabled

## Steps

### 1. Create two buckets

```bash
gcloud storage buckets create gs://pca-allowed-$RANDOM --location=europe-west9
gcloud storage buckets create gs://pca-forbidden-$RANDOM --location=europe-west9
```

### 2. Create a dedicated service account

```bash
gcloud iam service-accounts create pca-scoped-sa
```

### 3. Grant read access on the allowed bucket only

```bash
gcloud storage buckets add-iam-policy-binding gs://pca-allowed-<suffix> \
  --member="serviceAccount:pca-scoped-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role=roles/storage.objectViewer
```

Note that no binding is created on the forbidden bucket — this is the entire point.

### 4. Create a VM running as this service account

```bash
gcloud compute instances create pca-scoped-vm \
  --zone=europe-west9-a \
  --machine-type=e2-micro \
  --service-account=pca-scoped-sa@PROJECT_ID.iam.gserviceaccount.com \
  --scopes=cloud-platform
```

### 5. From inside the VM, test both buckets

```bash
gcloud compute ssh pca-scoped-vm --zone=europe-west9-a --command="gcloud storage ls gs://pca-allowed-<suffix>"
gcloud compute ssh pca-scoped-vm --zone=europe-west9-a --command="gcloud storage ls gs://pca-forbidden-<suffix>"
```

The first command succeeds; the second fails with a `403 Forbidden` — IAM permissions are additive, and the service account simply has none on that bucket.

## Cleanup

```bash
gcloud compute instances delete pca-scoped-vm --zone=europe-west9-a --quiet
gcloud storage rm -r gs://pca-allowed-<suffix>
gcloud storage rm -r gs://pca-forbidden-<suffix>
gcloud iam service-accounts delete pca-scoped-sa@PROJECT_ID.iam.gserviceaccount.com --quiet
```

## Key takeaways

- IAM permissions are strictly additive — there is no "deny by default on this specific resource" shortcut; you just never grant it.
- `--scopes=cloud-platform` on the VM only exposes what IAM already allows for its service account — the scope isn't the security boundary, the IAM binding is.
- A `403 Forbidden` response is the signature of an IAM authorization failure, as opposed to a `401` (no/invalid credentials at all).