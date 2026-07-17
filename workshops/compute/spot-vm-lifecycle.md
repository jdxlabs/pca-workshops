# Spot VM Lifecycle: Shutdown Scripts & Disk Persistence

> **Category:** Compute · **Level:** Beginner · **Duration:** ~15 min · **Cost:** Free tier

## Exam relevance

The exam expects you to know two specific compute tricks: how to gracefully handle a Spot VM's termination, and how to make a VM's state outlive the VM itself by decoupling the disk's lifecycle from the instance's. These are two separate proofs — don't conflate them, as it's easy to write your "survives termination" test data to the wrong disk.

## Objective

Attach a `shutdown-script` to a Spot VM and confirm it runs on termination, then prove a Persistent Disk created with `--no-auto-delete` survives instance deletion and can be reattached to a brand-new VM.

## Prerequisites

- A GCP project with Compute Engine API enabled

## Steps

### 1. Create a Persistent Disk independent of any instance

```bash
gcloud compute disks create pca-data-disk --size=10GB --zone=europe-west9-a
```

This disk is blank and unformatted — it has to be formatted and mounted before anything can be written to it.

### 2. Create a Spot VM with a shutdown script and the disk attached, without auto-delete

```bash
gcloud compute instances create pca-spot-vm \
  --zone=europe-west9-a \
  --machine-type=e2-micro \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --disk=name=pca-data-disk,device-name=pca-data-disk,auto-delete=no \
  --metadata=shutdown-script='#!/bin/bash
echo "Graceful shutdown at $(date)" >> /var/log/shutdown-script.log'
```

The `shutdown-script` writes to `/var/log` on the **boot disk** — fine for now, since we'll check it before that disk is gone. Note this is a different disk than `pca-data-disk`, which is why the two proofs below are separate.

The `device-name=pca-data-disk` part matters: without it, GCE names the `/dev/disk/by-id/google-*` link generically (`persistent-disk-1`, `persistent-disk-2`...) instead of after the disk itself, which breaks the predictable path used below.

### 3. Format and mount the data disk, and write a marker file to it

```bash
gcloud compute ssh pca-spot-vm --zone=europe-west9-a --command="
set -e
DISK=/dev/disk/by-id/google-pca-data-disk
ls -la \$DISK
sudo mkfs.ext4 -F \$DISK
sudo mkdir -p /mnt/data
sudo mount \$DISK /mnt/data
mountpoint -q /mnt/data && echo 'MOUNT OK' || (echo 'MOUNT FAILED'; exit 1)
echo 'this data must survive VM deletion' | sudo tee /mnt/data/marker.txt
"
```

Using `/dev/disk/by-id/google-pca-data-disk` instead of a raw device name (`/dev/sdb`, `/dev/sda`...) avoids ambiguity — GCE doesn't guarantee which device name a given disk gets. The `set -e` and explicit `mountpoint -q` check matter here: without them, a failed `mount` doesn't stop the script, and the following `tee` silently writes the marker file into an ordinary directory on the **boot disk** instead of onto `pca-data-disk` — giving you false confidence that persistence worked. If `ls -la $DISK` fails, run `ls /dev/disk/by-id/` to find the actual link name for your disk.

### 4. Stop the VM and confirm the shutdown script ran

```bash
gcloud compute instances stop pca-spot-vm --zone=europe-west9-a
gcloud compute instances get-serial-port-output pca-spot-vm --zone=europe-west9-a | grep "Graceful shutdown"
```

You should see the log line the script wrote — captured from the serial console before the boot disk is gone for good.

### 5. Delete the instance and confirm the data disk survives

```bash
gcloud compute instances delete pca-spot-vm --zone=europe-west9-a --quiet
gcloud compute disks list --filter="name=pca-data-disk"
```

The disk is still listed — deleting the VM did not delete it, because it was never marked for auto-delete. The boot disk, by contrast, is gone along with the VM (its `shutdown-script.log` included).

### 6. Reattach the disk to a new VM and confirm the marker file is still there

```bash
gcloud compute instances create pca-spot-vm-2 \
  --zone=europe-west9-a \
  --machine-type=e2-micro \
  --disk=name=pca-data-disk,device-name=pca-data-disk

gcloud compute ssh pca-spot-vm-2 --zone=europe-west9-a --command="
set -e
sudo mkdir -p /mnt/data
sudo mount /dev/disk/by-id/google-pca-data-disk /mnt/data
mountpoint -q /mnt/data && echo 'MOUNT OK' || (echo 'MOUNT FAILED'; exit 1)
cat /mnt/data/marker.txt
"
```

`mkfs` is not needed again — the filesystem you created in step 3 is still on the disk. Only the mount needs to be redone, since a fresh VM starts with no mounts configured.

## Cleanup

```bash
gcloud compute instances delete pca-spot-vm-2 --zone=europe-west9-a --quiet
gcloud compute disks delete pca-data-disk --zone=europe-west9-a --quiet
```

## Key takeaways

- `shutdown-script` metadata runs automatically on any normal shutdown/stop/delete (not just Spot preemption) — this is how you flush state or drain connections gracefully. Check it via the serial console log, since it typically writes to the boot disk, which won't survive deletion.
- A disk's `auto-delete` flag, not the VM's lifecycle, controls whether data survives instance deletion — but only for disks you deliberately wrote data to and mounted; attaching a disk isn't the same as using it.
- Reattaching a surviving disk to a new VM (or baking it into a Custom Image for a Managed Instance Group) is the standard way to recover state after termination. The filesystem persists; only the mount has to be redone on the new VM.
