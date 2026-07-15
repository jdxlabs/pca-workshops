# PCA Workshops

Hands-on workshops for the Google Cloud **Professional Cloud Architect (PCA)** certification. Each workshop is a self-contained Markdown file with an objective, step-by-step commands, cleanup, and exam takeaways.

Most workshops run on the GCP free tier, and some run fully locally (no cloud cost at all).

## Index

### Networking

| Workshop | Level | Cost |
|---|---|---|
| [Regional VPC & Subnets](workshops/networking/regional-vpc-subnet.md) | Beginner | Free tier |

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
| [GKE Workload Identity: KSA ↔ GSA Binding](workshops/security/gke-workload-identity.md) | Intermediate | Free tier |
| [Visualizing Network Policies with Cilium & Hubble](workshops/security/cilium-network-policies.md) | Intermediate | Free (local) |

### Service Mesh

| Workshop | Level | Cost |
|---|---|---|
| [Traffic Splitting with Istio (Cloud Service Mesh, Local Edition)](workshops/service-mesh/istio-traffic-splitting.md) | Intermediate | Free (local) |

## Adding a new workshop

1. Pick (or create) a category folder under `workshops/`.
2. Copy the structure of an existing workshop: **Exam relevance → Objective → Prerequisites → Steps → Cleanup → Key takeaways**.
3. Add one row to the matching table above.
