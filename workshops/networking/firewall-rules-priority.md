# Firewall Rules: Priority & Stateful Behavior

> **Category:** Networking · **Level:** Beginner · **Duration:** ~15 min · **Cost:** Free tier

## Exam relevance

Firewall questions on the exam are really about one number: priority (lower wins), and one property: statefulness (allowed inbound traffic gets its return traffic allowed automatically, with no separate egress rule needed).

## Objective

Create two conflicting firewall rules at different priorities, prove the lower-numbered one wins, and confirm return traffic doesn't need its own rule.

## Prerequisites

- A GCP project with Compute Engine API enabled

## Steps

### 1. Create a custom-mode VPC with two VMs in the same subnet

```bash
gcloud compute networks create pca-vpc --subnet-mode=custom
gcloud compute networks subnets create pca-subnet \
  --network=pca-vpc --region=europe-west9 --range=10.0.0.0/24

gcloud compute instances create vm-a \
  --zone=europe-west9-a --subnet=pca-subnet --machine-type=e2-micro
gcloud compute instances create vm-b \
  --zone=europe-west9-a --subnet=pca-subnet --machine-type=e2-micro
```

### 2. Allow SSH so you can actually reach the VMs

```bash
gcloud compute firewall-rules create pca-allow-ssh \
  --network=pca-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=0.0.0.0/0 \
  --priority=1000
```

Without this, `gcloud compute ssh` below has nothing to connect through — Google Cloud denies all ingress by default.

### 3. Create a low-priority (high-number) allow rule for ICMP

```bash
gcloud compute firewall-rules create pca-allow-icmp \
  --network=pca-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=icmp \
  --source-ranges=10.0.0.0/24 \
  --priority=2000
```

### 4. Confirm ping works between VM-A and VM-B

```bash
gcloud compute ssh vm-a --zone=europe-west9-a --command="ping -c 3 <VM_B_INTERNAL_IP>"
```

Ping succeeds.

### 5. Create a higher-priority (lower-number) deny rule for the same traffic

```bash
gcloud compute firewall-rules create pca-deny-icmp \
  --network=pca-vpc \
  --direction=INGRESS \
  --action=DENY \
  --rules=icmp \
  --source-ranges=10.0.0.0/24 \
  --priority=1000
```

### 6. Retry the ping

```bash
gcloud compute ssh vm-a --zone=europe-west9-a --command="ping -c 3 <VM_B_INTERNAL_IP>"
```

Ping now fails — the rule with priority `1000` (lower number, higher precedence) wins over the one at `2000`, even though the allow rule was created first.

### 7. Prove statefulness with the existing SSH rule

```bash
gcloud compute ssh vm-a --zone=europe-west9-a --command="echo connected"
```

This SSH session's return traffic (VM-A → your machine) flows back without any egress rule — the single inbound rule from step 2 is enough, because Google Cloud firewalls are stateful by default.

## Cleanup

```bash
gcloud compute firewall-rules delete pca-allow-ssh pca-allow-icmp pca-deny-icmp --quiet
gcloud compute instances delete vm-a vm-b --zone=europe-west9-a --quiet
gcloud compute networks subnets delete pca-subnet --region=europe-west9 --quiet
gcloud compute networks delete pca-vpc --quiet
```

## Key takeaways

- Lower priority number = evaluated first = wins on conflict, regardless of creation order or whether it's an allow or deny.
- Google Cloud firewalls are distributed (enforced per-VM NIC) but defined at VPC scope, and apply globally across all regions in that VPC.
- Statefulness means one inbound allow rule is enough — the return path is tracked automatically, no matching egress rule required.
