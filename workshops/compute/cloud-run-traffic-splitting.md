# Cloud Run: Revisions, Traffic Splitting & Scale-to-Zero

> **Category:** Compute · **Level:** Beginner · **Duration:** ~20 min · **Cost:** Free tier

## Exam relevance

Cloud Run questions hinge on a few precise facts: revisions are immutable, traffic is split by weight between revisions automatically, and idle services scale to zero and cost nothing. This workshop exercises all three in one pass.

## Objective

Deploy two immutable revisions of the same service, split traffic between them by percentage, and observe scale-to-zero behavior once traffic stops.

## Prerequisites

- A GCP project with Cloud Run API enabled

## Steps

### 1. Deploy the first revision (v1)

```bash
gcloud run deploy pca-app \
  --image=us-docker.pkg.dev/cloudrun/container/hello \
  --region=europe-west9 \
  --allow-unauthenticated
```

### 2. Deploy a second revision (v2) without shifting traffic yet

```bash
gcloud run deploy pca-app \
  --image=us-docker.pkg.dev/cloudrun/container/hello \
  --region=europe-west9 \
  --tag=v2rev \
  --no-traffic
```

### 3. List revisions to confirm both exist immutably

```bash
gcloud run revisions list --service=pca-app --region=europe-west9
```

Each deployment created a new, separate revision — neither was mutated in place.

### 4. Split traffic 90/10 between the two revisions

```bash
gcloud run services update-traffic pca-app \
  --region=europe-west9 \
  --to-revisions=LATEST=10,pca-app-00001-abc=90
```

(Replace `pca-app-00001-abc` with your actual first revision name from step 3.)

### 5. Confirm the split and observe scale-to-zero

```bash
for i in $(seq 1 20); do curl -s -o /dev/null -w "%{http_code}\n" $(gcloud run services describe pca-app --region=europe-west9 --format='value(status.url)'); done
```

Wait ~15 minutes without sending requests, then check the Cloud Run console metrics: "Container instance count" drops to 0, and billing stops accruing.

## Cleanup

```bash
gcloud run services delete pca-app --region=europe-west9 --quiet
```

## Key takeaways

- Every deployment creates a new, immutable revision — you can always roll back by shifting traffic to a previous one.
- Traffic splitting between revisions is weight-based and native to Cloud Run — no service mesh required.
- Scale-to-zero is the default idle behavior; you pay only while requests are being served.