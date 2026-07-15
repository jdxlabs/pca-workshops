# SRE Signals: Liveness/Readiness Probes & Uptime Checks

> **Category:** Observability & SRE · **Level:** Intermediate · **Duration:** ~20 min · **Cost:** Free (local) + free tier (GCP)

## Exam relevance

The exam draws a sharp line between *internal* self-healing signals (probes, handled by Kubernetes) and *external* black-box monitoring (Uptime Checks + Alerting, handled by Cloud Monitoring). This workshop exercises both halves.

## Objective

Trigger a pod restart via a failing liveness probe on a local k3s cluster, then configure an external Uptime Check with an Alerting policy on a real public endpoint.

## Prerequisites

- A local [k3s](https://k3s.io/) cluster (see the [Network Policies workshop](../security/cilium-network-policies.md) for install steps) or any Kubernetes cluster
- A GCP project with Cloud Monitoring enabled and a public HTTP endpoint (e.g. the Cloud Run service from the [traffic splitting workshop](../compute/cloud-run-traffic-splitting.md))

## Steps

### 1. Deploy a pod with a deliberately failing liveness probe

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

### 2. Break the liveness probe and watch Kubernetes self-heal

```bash
kubectl exec pca-probe-test -- rm /tmp/healthy
kubectl get events --field-selector involvedObject.name=pca-probe-test
```

Within seconds, the liveness probe starts failing and Kubernetes restarts the container automatically — no human intervention.

### 3. Create an Uptime Check on a public endpoint

```bash
gcloud monitoring uptime create pca-uptime-check \
  --resource-type=uptime-url \
  --hostname=YOUR_CLOUD_RUN_URL_HOST \
  --path=/ \
  --protocol=https \
  --period=5
```

### 4. Create an Alerting Policy tied to that check, with an email notification channel

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

### 5. Trigger the alert

```bash
gcloud run services delete pca-app --region=europe-west9 --quiet
```

Wait a few minutes and check your inbox — the Uptime Check fails and the Alerting Policy notifies you, purely from the outside, with no knowledge of internal pod health.

## Cleanup

```bash
kubectl delete pod pca-probe-test
gcloud monitoring uptime delete pca-uptime-check --quiet
gcloud alpha monitoring policies list  # then delete the created policy by ID
```

## Key takeaways

- Liveness and Readiness probes are internal, Kubernetes-native signals: one restarts broken containers, the other withholds traffic until a pod is actually ready.
- An Uptime Check is an external, "outside-in" signal — it doesn't know or care why a service is down, only that it is.
- SLIs are built from exactly these two signal types combined: internal health drives self-healing, external checks drive the SLO/SLA conversation with users.
