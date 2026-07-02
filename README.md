# geo-mlops-infra

GitOps infra repo for [geo-mlops](https://github.com/rsmrtk/geo-mlops):
Helm charts + ArgoCD Application manifests. This repo has no application
code — it's what ArgoCD watches and syncs.

## Why a separate repo

The app repo's CI never runs `kubectl apply`. On every push to `main` it
builds images, then commits the new image tag into this repo
(`envs/prod/values.yaml`) using a bot token. ArgoCD (`automated: prune,
selfHeal`) picks up the commit and reconciles the cluster. Deploys are a
Git commit, not a pipeline step with cluster credentials.

```
geo-mlops (app CI) → bumps envs/prod/values.yaml here → ArgoCD syncs
```

## Layout

```
argocd/apps.yaml       ArgoCD Application defs — postgres, geocoder, ml-svc, mlflow
charts/geocoder/       Go API — Deployment, Service, Ingress
charts/ml-svc/         FastAPI model server — Deployment, Service
charts/mlflow/         MLflow tracking server, GCS-backed artifact store
charts/postgres/       Geocode cache DB
charts/monitoring/     Prometheus/Grafana values
envs/prod/values.yaml  Env-specific overrides (image tags, ingress host, sizing)
```

Each `charts/<name>` is a standalone Helm chart; `envs/<env>/values.yaml`
is layered on top per ArgoCD `Application` (`helm.valueFiles`). Adding a
new environment means adding `envs/<env>/values.yaml` and a matching set
of `Application` manifests — no chart changes.

## Components

- **postgres** — S2-cell-keyed geocode cache used by `geocoder`
- **geocoder** — Go HTTP API, public via Ingress
- **ml-svc** — FastAPI model server, loads from MLflow Registry (`champion` alias)
- **mlflow** — tracking server + model registry; GCS artifact storage if
  configured, otherwise falls back to a local PVC

All four sync with `prune: true, selfHeal: true` — manual cluster drift gets
reverted automatically.

## Deploying to your own cluster

Nothing here is applied by hand — ArgoCD reconciles from Git. No IPs,
hostnames, passwords, or bucket names are committed to this repo; you set
your own before the first sync.

1. **Fork/clone this repo** and point `argocd/apps.yaml`'s `repoURL` at
   your fork.

2. **Set the required values** in `envs/prod/values.yaml` (or a new
   `envs/<env>/values.yaml`):

   ```yaml
   postgres:
     auth:
       password: "" # required — leave blank here, set via --set or a sealed secret
   geocoder:
     ingress:
       host: "" # required — e.g. geo.<your-cluster-ip>.nip.io, or your own domain
   mlflow:
     gcs:
       bucket: "" # optional — e.g. gs://<your-project>-mlflow-artifacts
                  # leave empty to use local PVC storage instead (no GCP needed)
   ```

   Charts fail fast with a clear error if a required value (`postgres.auth.password`,
   `geocoder.ingress.host`) is missing — nothing silently deploys half-configured.

3. **Provide secrets out-of-band**, never in Git:
   - `postgres.auth.password` — pass with `--set` or via a sealed-secrets /
     external-secrets pipeline
   - `mlflow.gcs.secretName` — if using GCS, create the Secret yourself:
     `kubectl create secret generic mlflow-gcs-key -n geo-mlops --from-file=key.json=<path-to-service-account-key>`
   - `grafana.adminPassword` (monitoring chart) — pass with
     `--set grafana.adminPassword=<your-password>` when installing the
     upstream `kube-prometheus-stack` chart

4. **Point ArgoCD at your fork** (`kubectl apply -f argocd/apps.yaml`) and
   let it sync.

To ship a new app version afterwards, push to your `geo-mlops` fork's
`main` — CI builds, tags, and updates `envs/prod/values.yaml` in this repo.
To change infra (resources, ingress host, chart config), edit the relevant
chart or `envs/prod/values.yaml` directly and merge — ArgoCD does the rest.

## Stack

Helm · ArgoCD · Kubernetes · MLflow · GCS (optional)
