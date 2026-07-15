# PCA Workshops

Hands-on workshops for the Google Cloud **Professional Cloud Architect (PCA)** certification. Each workshop is a self-contained Markdown file with an objective, step-by-step commands, cleanup, and exam takeaways.

Most workshops run on the GCP free tier, and some run fully locally (no cloud cost at all).

## Index

### Compute

| Workshop | Level | Cost |
|---|---|---|
| [Spot VM Lifecycle: Shutdown Scripts & Disk Persistence](workshops/compute/spot-vm-lifecycle.md) | Beginner | Free tier |
| [Cloud Run: Revisions, Traffic Splitting & Scale-to-Zero](workshops/compute/cloud-run-traffic-splitting.md) | Beginner | Free tier |

### Networking

| Workshop | Level | Cost |
|---|---|---|
| [Regional VPC & Subnets](workshops/networking/regional-vpc-subnet.md) | Beginner | Free tier |
| [Firewall Rules: Priority & Stateful Behavior](workshops/networking/firewall-rules-priority.md) | Beginner | Free tier |

### Storage

| Workshop | Level | Cost |
|---|---|---|
| [Cloud Storage: Lifecycle Rules & Signed URLs](workshops/storage/gcs-lifecycle-and-signed-urls.md) | Beginner | Free tier |

### Data & Analytics

| Workshop | Level | Cost |
|---|---|---|
| [BigQuery Cost Control with Partitioning](workshops/data-analytics/bigquery-partitioning.md) | Beginner | Free tier |

### AI & Agents

| Workshop | Level | Cost |
|---|---|---|
| [Your First Support Agent (Gemini Enterprise Agent Platform)](workshops/ai/gemini-enterprise-support-agent.md) | Beginner | Free tier |

### Security

| Workshop | Level | Cost |
|---|---|---|
| [IAM Least Privilege: Scoping a Service Account to One Bucket](workshops/security/least-privilege-service-accounts.md) | Beginner | Free tier |
| [GKE Workload Identity: KSA ↔ GSA Binding](workshops/security/gke-workload-identity.md) | Intermediate | Free tier |
| [Visualizing Network Policies with Cilium & Hubble](workshops/security/cilium-network-policies.md) | Intermediate | Free (local) |

### Service Mesh

| Workshop | Level | Cost |
|---|---|---|
| [Traffic Splitting with Istio (Cloud Service Mesh, Local Edition)](workshops/service-mesh/istio-traffic-splitting.md) | Intermediate | Free (local) |

### Observability & SRE

| Workshop | Level | Cost |
|---|---|---|
| [SRE Signals: Liveness/Readiness Probes & Uptime Checks](workshops/observability/probes-and-uptime-checks.md) | Intermediate | Free (local) + Free tier |

### DevOps & IaC

| Workshop | Level | Cost |
|---|---|---|
| [Terraform Core Workflow: Init, Plan, Apply](workshops/devops/terraform-basics.md) | Beginner | Free tier |
| [CI/CD with Cloud Build: `cloudbuild.yaml` to Cloud Run](workshops/devops/cloudbuild-cicd.md) | Beginner | Free tier |

## Adding a new workshop

1. Pick (or create) a category folder under `workshops/`.
2. Copy the structure of an existing workshop: **Exam relevance → Objective → Prerequisites → Steps → Cleanup → Key takeaways**.
3. Add one row to the matching table above.
