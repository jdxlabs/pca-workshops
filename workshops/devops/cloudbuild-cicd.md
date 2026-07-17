# CI/CD with Cloud Build: `cloudbuild.yaml` to Cloud Run

> **Category:** DevOps & IaC · **Level:** Beginner · **Duration:** ~15 min · **Cost:** Free tier

## Exam relevance

The exam expects you to know that Cloud Build pipelines are declared entirely in `cloudbuild.yaml`, and that triggers (push, schedule, event) are what turn a manual pipeline into a real CI/CD flow.

## Objective

Define a build pipeline in `cloudbuild.yaml` that builds a container image and deploys it straight to Cloud Run, run it manually, then wire it to a Git push trigger.

## Prerequisites

- A GCP project with Cloud Build, Artifact Registry, and Cloud Run APIs enabled

## Steps

### 1. Write a minimal app and its `Dockerfile`

Cloud Run needs an HTTP server listening on `$PORT` (defaults to `8080`). Create `app.py`:

```python
from flask import Flask
import os

app = Flask(__name__)

@app.route("/")
def hello():
    return "hello from pca-app"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

And `Dockerfile` next to it:

```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir flask
COPY app.py .
CMD ["python", "app.py"]
```

### 2. Write `cloudbuild.yaml`

```yaml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'europe-west9-docker.pkg.dev/$PROJECT_ID/pca-repo/app', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'europe-west9-docker.pkg.dev/$PROJECT_ID/pca-repo/app']
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - run
      - deploy
      - pca-app
      - --image=europe-west9-docker.pkg.dev/$PROJECT_ID/pca-repo/app
      - --region=europe-west9
      - --allow-unauthenticated
images:
  - 'europe-west9-docker.pkg.dev/$PROJECT_ID/pca-repo/app'
```

`$PROJECT_ID` is a [built-in Cloud Build substitution](https://cloud.google.com/build/docs/configuring-builds/substitute-variable-values) — Cloud Build resolves it automatically to whichever project the build runs in. No need to hardcode or `sed`-replace your project ID.

### 3. Create the Artifact Registry repository

```bash
gcloud artifacts repositories create pca-repo \
  --repository-format=docker \
  --location=europe-west9
```

### 4. Run the pipeline manually first

```bash
gcloud builds submit --config=cloudbuild.yaml .
```

Watch the three steps execute in order: build, push, deploy. Confirm the service responds:

```bash
gcloud run services describe pca-app --region=europe-west9 --format='value(status.url)'
```

### 5. Wire it to a push trigger (if your sandbox permits repo connections)

```bash
gcloud builds triggers create github \
  --repo-name=YOUR_REPO \
  --repo-owner=YOUR_GH_USER \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml
```

If your sandbox project doesn't allow connecting an external repository, this step is informational — the manual `gcloud builds submit` in step 3 already proves the pipeline itself.

### 6. Push a commit and watch the trigger fire

```bash
git commit --allow-empty -m "trigger cloud build"
git push
```

Check the Cloud Build history in the console — a new build starts automatically and redeploys `pca-app`.

## Cleanup

```bash
gcloud run services delete pca-app --region=europe-west9 --quiet
gcloud artifacts repositories delete pca-repo --location=europe-west9 --quiet
```

## Key takeaways

- `cloudbuild.yaml` is the entire pipeline definition — steps run sequentially, each in its own container.
- A trigger is just an event source (push, PR, schedule, Pub/Sub message) bound to a build config — the pipeline itself doesn't change.
- Running `gcloud builds submit` manually is the same execution path a trigger uses; it's the fastest way to debug a pipeline before wiring automation.