# SRE Signals: Liveness/Readiness Probes & Uptime Checks

> **Category:** Observability & SRE · **Level:** Intermediate · **Duration:** ~20 min · **Cost:** Free tier (Cloud Run/Monitoring) + GKE credit or free (local k3s)

## Exam relevance

The exam draws a sharp line between *internal* self-healing signals (probes, handled by Kubernetes) and *external* black-box monitoring (Uptime Checks + Alerting, handled by Cloud Monitoring). This workshop exercises both halves. The probe part is plain Kubernetes — it runs the same on GKE or on a local cluster.

## Objective

Trigger a pod restart via a failing liveness probe, then configure an external Uptime Check with an Alerting policy on a real public endpoint.

## Prerequisites

- A GCP project with the Cloud Run API and Cloud Monitoring enabled (needed regardless of where your Kubernetes pod runs)
- Pick **one** cluster option below for the probe part

## Steps

### 1. Create your cluster

**Option A — GKE Autopilot**

```bash
gcloud container clusters create-auto pca-cluster --region=europe-west9
gcloud container clusters get-credentials pca-cluster --region=europe-west9
```

**Option B — local k3s**

```bash
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

### 2. Deploy a pod with a deliberately failing liveness probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pca-probe-test
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "touch /tmp/healthy && sleep 3600"]
    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      initialDelaySeconds: 5
      periodSeconds: 5
    readinessProbe:
      exec:
        command: ["sleep", "20"]
      initialDelaySeconds: 0
      periodSeconds: 5
```

```bash
kubectl apply -f probe-test.yaml
kubectl get pod pca-probe-test -w
```

Note the pod is **not** marked `Ready` for the first ~20 seconds — the readiness probe delays traffic until the app has "warmed up".

### 3. Break the liveness probe and watch Kubernetes self-heal

```bash
kubectl exec pca-probe-test -- rm /tmp/healthy
kubectl get events --field-selector involvedObject.name=pca-probe-test
```

Within seconds, the liveness probe starts failing and Kubernetes restarts the container automatically — no human intervention.

### 4. Deploy a throwaway public endpoint to monitor

```bash
gcloud run deploy pca-uptime-target \
  --image=us-docker.pkg.dev/cloudrun/container/hello \
  --region=europe-west9 \
  --allow-unauthenticated
```

### 5. Create an Uptime Check on that endpoint

```bash
gcloud monitoring uptime create pca-uptime-check \
  --resource-type=uptime-url \
  --hostname=$(gcloud run services describe pca-uptime-target --region=europe-west9 --format='value(status.url)' | sed 's|https://||') \
  --path=/ \
  --protocol=https \
  --period=5
```

### 6. Create an Alerting Policy tied to that check, with an email notification channel

```bash
gcloud alpha monitoring channels create \
  --display-name="pca-email" \
  --type=email \
  --channel-labels=email_address=YOUR_EMAIL

gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="pca-uptime-alert" \
  --condition-display-name="Uptime check failing" \
  --condition-filter='metric.type="monitoring.googleapis.com/uptime_check/check_passed" AND resource.type="uptime_url"' \
  --condition-threshold-value=1 \
  --condition-threshold-comparison=COMPARISON_LT
```

### 7. Trigger the alert

```bash
gcloud run services delete pca-uptime-target --region=europe-west9 --quiet
```

Wait a few minutes and check your inbox — the Uptime Check fails and the Alerting Policy notifies you, purely from the outside, with no knowledge of internal pod health.

## Cleanup

```bash
kubectl delete pod pca-probe-test
gcloud monitoring uptime delete pca-uptime-check --quiet
gcloud alpha monitoring policies list  # then delete the created policy by ID
```

(`pca-uptime-target` was already deleted in step 7 to trigger the alert.)

Then tear down whichever cluster you used:

```bash
# Option A — GKE Autopilot
gcloud container clusters delete pca-cluster --region=europe-west9 --quiet

# Option B — local k3s
/usr/local/bin/k3s-uninstall.sh
```

## Key takeaways

- Liveness and Readiness probes are internal, Kubernetes-native signals: one restarts broken containers, the other withholds traffic until a pod is actually ready.
- An Uptime Check is an external, "outside-in" signal — it doesn't know or care why a service is down, only that it is.
- SLIs are built from exactly these two signal types combined: internal health drives self-healing, external checks drive the SLO/SLA conversation with users.
