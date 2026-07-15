# Terraform Core Workflow: Init, Plan, Apply

> **Category:** DevOps & IaC · **Level:** Beginner · **Duration:** ~15 min · **Cost:** Free tier

## Exam relevance

The exam tests the exact sequencing of the Terraform workflow (`init` → `plan` → `apply`) and what a version constraint does during `init`. This is the fastest way to internalize both.

## Objective

Provision a single GCS bucket through the full Terraform workflow, observe a plan diff after a change, then tear it down with `destroy`.

## Prerequisites

- [Terraform CLI](https://developer.hashicorp.com/terraform/install) installed locally
- `gcloud` authenticated with Application Default Credentials (`gcloud auth application-default login`)

## Steps

### 1. Author the configuration

```hcl
# main.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = "YOUR_PROJECT_ID"
  region  = "europe-west9"
}

resource "google_storage_bucket" "pca_bucket" {
  name          = "pca-tf-workshop-YOUR_SUFFIX"
  location      = "EU"
  force_destroy = true
}
```

### 2. Init — download the provider matching the version constraint

```bash
terraform init
```

Terraform resolves and downloads the latest provider version satisfying `~> 5.0` — not necessarily the newest overall.

### 3. Plan — preview the change

```bash
terraform plan
```

Terraform shows exactly one resource to be created — nothing is touched yet.

### 4. Apply — create the resource

```bash
terraform apply
```

Confirm with `yes`. The bucket now exists.

### 5. Change and re-plan

Add a label to the resource block:

```hcl
resource "google_storage_bucket" "pca_bucket" {
  name          = "pca-tf-workshop-YOUR_SUFFIX"
  location      = "EU"
  force_destroy = true
  labels = {
    owner = "pca-workshop"
  }
}
```

```bash
terraform plan
```

The plan shows an in-place update — Terraform diffs against its state file, not against a fresh read of reality.

## Cleanup

```bash
terraform destroy
```

## Key takeaways

- The workflow order is fixed: `init` (providers) → `plan` (preview) → `apply` (execute) — the exam tests this sequence directly.
- A version constraint like `~> 5.0` is resolved at `init` time to the latest matching version, not pinned to an exact release.
- `plan` always diffs against Terraform's own state file, which is why drift (manual changes made outside Terraform) can surprise you.
