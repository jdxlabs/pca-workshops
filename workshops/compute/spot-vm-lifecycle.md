# Spot VM Lifecycle: Shutdown Scripts & Disk Persistence

> **Category:** Compute · **Level:** Beginner · **Duration:** ~15 min · **Cost:** Free tier

## Exam relevance

The exam expects you to know two specific compute tricks: how to gracefully handle a Spot VM's termination, and how to make a VM's state outlive the VM itself by decoupling the disk's lifecycle from the instance's.

## Objective

Attach a `shutdown-script` to a Spot VM, then prove a Persistent Disk created with `--no-auto-delete` survives instance deletion and can be reattached to a brand-new VM.

## Prerequisites

- A GCP project with Compute Engine API enabled

## Steps

### 1. Create a Persistent Disk independent of any instance

```bash
gcloud compute disks create pca-data-disk --size=10GB --zone=europe-west9-a
```

### 2. Create a Spot VM with a shutdown script and the disk attached, without auto-delete

```bash
gcloud compute instances create pca-spot-vm \
  --zone=europe-west9-a \
  --machine-type=e2-micro \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --disk=name=pca-data-disk,auto-delete=no \
  --metadata=shutdown-script='#!/bin/bash
echo "Graceful shutdown at $(date)" >> /tmp/shutdown.log'
```

### 3. Delete the instance and confirm the disk survives

```bash
gcloud compute instances delete pca-spot-vm --zone=europe-west9-a --quiet
gcloud compute disks list --filter="name=pca-data-disk"
```

The disk is still listed — deleting the VM did not delete it, because it was never marked for auto-delete.

### 4. Reattach the disk to a new VM

```bash
gcloud compute instances create pca-spot-vm-2 \
  --zone=europe-west9-a \
  --machine-type=e2-micro \
  --disk=name=pca-data-disk
```

Any data previously written to that disk is available on the new VM.

## Cleanup

```bash
gcloud compute instances delete pca-spot-vm-2 --zone=europe-west9-a --quiet
gcloud compute disks delete pca-data-disk --zone=europe-west9-a --quiet
```

## Key takeaways

- `shutdown-script` metadata runs automatically when a Spot/preemptible VM receives its termination notice — this is how you flush state or drain connections gracefully.
- A disk's `auto-delete` flag, not the VM's lifecycle, controls whether data survives instance deletion.
- Reattaching a surviving disk to a new VM (or baking it into a Custom Image for a Managed Instance Group) is the standard way to recover state after termination.