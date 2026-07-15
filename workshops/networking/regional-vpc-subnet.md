# Regional VPC & Subnets

> **Category:** Networking · **Level:** Beginner · **Duration:** ~20 min · **Cost:** Free tier

## Exam relevance

GCP subnets are **regional**, not zonal — a single subnet spans every zone in its region. This is a classic PCA exam trap for anyone coming from AWS, where subnets are zonal. This workshop proves the behavior hands-on.

## Objective

Show that two VMs in *different zones* of the *same region*, attached to the *same subnet*, can communicate directly — no router, no VPN, no peering required.

## Prerequisites

- A GCP project with billing enabled (free tier is enough)
- `gcloud` CLI authenticated, or access to Cloud Shell

## Steps

### 1. Create a custom-mode VPC

```bash
gcloud compute networks create pca-vpc --subnet-mode=custom
```

### 2. Create a single regional subnet

```bash
gcloud compute networks subnets create pca-subnet \
  --network=pca-vpc \
  --region=europe-west9 \
  --range=10.0.0.0/24
```

### 3. Allow internal ICMP/SSH traffic

```bash
gcloud compute firewall-rules create pca-allow-internal \
  --network=pca-vpc \
  --allow=icmp,tcp:22 \
  --source-ranges=10.0.0.0/24
```

### 4. Deploy VM-A and VM-B in different zones, same subnet

```bash
gcloud compute instances create vm-a \
  --zone=europe-west9-a \
  --subnet=pca-subnet \
  --machine-type=e2-micro

gcloud compute instances create vm-b \
  --zone=europe-west9-b \
  --subnet=pca-subnet \
  --machine-type=e2-micro
```

### 5. Validate connectivity

```bash
# Get VM-B's internal IP
gcloud compute instances describe vm-b \
  --zone=europe-west9-b \
  --format='get(networkInterfaces[0].networkIP)'

# SSH into VM-A and ping VM-B's internal IP
gcloud compute ssh vm-a --zone=europe-west9-a --command="ping -c 3 <VM_B_INTERNAL_IP>"
```

The ping succeeds — both VMs are on the same subnet even though they live in different zones.

## Cleanup

```bash
gcloud compute instances delete vm-a vm-b --zone=europe-west9-a --quiet
gcloud compute instances delete vm-b --zone=europe-west9-b --quiet
gcloud compute firewall-rules delete pca-allow-internal --quiet
gcloud compute networks subnets delete pca-subnet --region=europe-west9 --quiet
gcloud compute networks delete pca-vpc --quiet
```

## Key takeaways

- A GCP subnet's scope is **regional**, covering all zones in that region automatically.
- High availability across zones doesn't require multiple subnets — one subnet is enough.
- This is the opposite of AWS, where each subnet is pinned to a single AZ.
