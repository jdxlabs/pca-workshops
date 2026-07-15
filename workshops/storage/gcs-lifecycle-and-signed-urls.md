# Cloud Storage: Lifecycle Rules & Signed URLs

> **Category:** Storage · **Level:** Beginner · **Duration:** ~15 min · **Cost:** Free tier

## Exam relevance

Two recurring exam scenarios live here: automatically moving cold data to cheaper storage classes, and granting temporary access to an object without ever creating a Google account for the requester.

## Objective

Configure a lifecycle rule that auto-transitions objects to Coldline, then generate a Signed URL that grants time-limited, credential-free access to a private object.

## Prerequisites

- A GCP project with Cloud Storage API enabled
- `gcloud` authenticated with a user or service account that has `roles/iam.serviceAccountTokenCreator` on itself (needed for signed URLs without a key file)

## Steps

### 1. Create a bucket and upload a test object

```bash
gcloud storage buckets create gs://pca-workshop-$RANDOM --location=europe-west9
echo "confidential report" > report.txt
gcloud storage cp report.txt gs://pca-workshop-<bucket-suffix>/report.txt
```

### 2. Apply a lifecycle rule: transition to Coldline after 90 days

```json
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
      "condition": {"age": 90}
    }
  ]
}
```

```bash
gcloud storage buckets update gs://pca-workshop-<bucket-suffix> --lifecycle-file=lifecycle.json
```

### 3. Confirm the object has no public access

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  https://storage.googleapis.com/pca-workshop-<bucket-suffix>/report.txt
```

You should see `403`.

### 4. Generate a Signed URL valid for 10 minutes

```bash
gcloud storage sign-url gs://pca-workshop-<bucket-suffix>/report.txt \
  --impersonate-service-account=YOUR_SA@PROJECT_ID.iam.gserviceaccount.com \
  --duration=10m
```

### 5. Access the object anonymously with the signed URL

```bash
curl -s -o /dev/null -w "%{http_code}\n" "<signed-url-from-previous-step>"
```

You should see `200` — no Google login involved. Wait 10 minutes and retry: it now fails.

## Cleanup

```bash
gcloud storage rm -r gs://pca-workshop-<bucket-suffix>
```

## Key takeaways

- Lifecycle rules are evaluated per-object and act automatically — no cron job or script needed for cold-tiering.
- A Signed URL grants scoped, time-limited access without an IAM binding or a Google identity on the requester's side.
- `--impersonate-service-account` signs the URL without ever downloading a service account key file — the modern, keyless way to do it.