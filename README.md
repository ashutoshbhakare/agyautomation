# hello-server on GKE — Antigravity CLI Demo

Example files accompanying the blog post *"Antigravity CLI: An AI Deploy Agent for `gcloud` and `kubectl`"*. This repo deploys a sample `hello-server` app to GKE and configures autoscaling on it, driven partly by [Antigravity CLI](https://antigravity.google) (`agy`).

## Contents

| File | Purpose |
|---|---|
| `deployment.yaml` | Deployment + Service for `hello-server` |
| `hpa.yaml` | HorizontalPodAutoscaler for `hello-server` (2–10 replicas, 70% CPU target) |
| `configure-hpa.sh` | Non-interactive script that applies the HPA via `agy -p`, suitable for CI |

## Prerequisites

- A Google Cloud project with billing enabled
- `gcloud` installed and authenticated (`gcloud auth login`)
- A GKE cluster (Autopilot or Standard)
- `kubectl` and the GKE auth plugin (`gcloud components install gke-gcloud-auth-plugin`)
- [Antigravity CLI](https://antigravity.google) installed (`curl -fsSL https://antigravity.google/cli/install.sh | bash`)

## Setup

1. **Get cluster credentials**
   ```bash
   gcloud container clusters get-credentials hello-cluster \
     --region us-central1 \
     --project PROJECT_ID
   ```

2. **Build and push the app image** (bring your own app, or swap in any image)
   ```bash
   gcloud builds submit --tag gcr.io/PROJECT_ID/hello-server:latest
   ```
   Update the `image:` field in `deployment.yaml` to match your project ID.

3. **Deploy the app**
   ```bash
   kubectl create namespace production --dry-run=client -o yaml | kubectl apply -f -
   kubectl apply -f deployment.yaml
   ```

4. **Configure autoscaling**

   Either apply the manifest directly:
   ```bash
   kubectl apply -f hpa.yaml
   ```

   Or let Antigravity CLI do it non-interactively (see script for required env vars):
   ```bash
   export PROJECT_ID=your-project-id
   export CLUSTER_NAME=hello-cluster
   export REGION=us-central1
   export NAMESPACE=production
   export DEPLOYMENT=hello-server

   ./configure-hpa.sh
   ```

5. **Verify**
   ```bash
   kubectl get deployment hello-server -n production
   kubectl get hpa hello-server -n production
   kubectl get pods -n production
   ```

## Notes

- `deployment.yaml` sets CPU `requests`/`limits` on the container — this is required for CPU-based HPA scaling to work at all; without a `requests.cpu` value, the HPA has no baseline to calculate utilization against.
- `configure-hpa.sh` runs with `--permission-mode always-proceed`, which skips Antigravity's default approval prompt. Only use that flag in a script you've reviewed and trust — for interactive/manual use, stick with the default `request-review` mode so you can see each command before it runs.
- Replace `PROJECT_ID` and `gcr.io/PROJECT_ID/hello-server:latest` with your actual project ID and image before applying.

## Cleanup

```bash
kubectl delete -f hpa.yaml
kubectl delete -f deployment.yaml
```

## License

MIT — use freely for your own blog post, demo, or learning project.
