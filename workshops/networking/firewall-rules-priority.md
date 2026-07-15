# Firewall Rules: Priority & Stateful Behavior

> **Category:** Networking · **Level:** Beginner · **Duration:** ~15 min · **Cost:** Free tier

## Exam relevance

Firewall questions on the exam are really about one number: priority (lower wins), and one property: statefulness (allowed inbound traffic gets its return traffic allowed automatically, with no separate egress rule needed).

## Objective

Create two conflicting firewall rules at different priorities, prove the lower-numbered one wins, and confirm return traffic doesn't need its own rule.

## Prerequisites

- Reuse the VPC and VMs from the [Regional VPC workshop](regional-vpc-subnet.md), or create a fresh custom-mode VPC with two VMs in the same subnet

## Steps

### 1. Create a low-priority (high-number) allow rule for ICMP

```bash
gcloud compute firewall-rules create pca-allow-icmp \
  --network=pca-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=icmp \
  --source-ranges=10.0.0.0/24 \
  --priority=2000
```

### 2. Confirm ping works between VM-A and VM-B

```bash
gcloud compute ssh vm-a --zone=europe-west9-a --command="ping -c 3 <VM_B_INTERNAL_IP>"
```

Ping succeeds.

### 3. Create a higher-priority (lower-number) deny rule for the same traffic

```bash
gcloud compute firewall-rules create pca-deny-icmp \
  --network=pca-vpc \
  --direction=INGRESS \
  --action=DENY \
  --rules=icmp \
  --source-ranges=10.0.0.0/24 \
  --priority=1000
```

### 4. Retry the ping

```bash
gcloud compute ssh vm-a --zone=europe-west9-a --command="ping -c 3 <VM_B_INTERNAL_IP>"
```

Ping now fails — the rule with priority `1000` (lower number, higher precedence) wins over the one at `2000`, even though the allow rule was created first.

### 5. Prove statefulness with SSH (no egress rule needed)

```bash
gcloud compute firewall-rules create pca-allow-ssh \
  --network=pca-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=0.0.0.0/0 \
  --priority=1000

gcloud compute ssh vm-a --zone=europe-west9-a --command="echo connected"
```

The SSH session's return traffic (VM-A → your machine) flows back without any egress rule — only the inbound rule was needed, because Google Cloud firewalls are stateful by default.

## Cleanup

```bash
gcloud compute firewall-rules delete pca-allow-icmp pca-deny-icmp pca-allow-ssh --quiet
```

## Key takeaways

- Lower priority number = evaluated first = wins on conflict, regardless of creation order or whether it's an allow or deny.
- Google Cloud firewalls are distributed (enforced per-VM NIC) but defined at VPC scope, and apply globally across all regions in that VPC.
- Statefulness means one inbound allow rule is enough — the return path is tracked automatically, no matching egress rule required.