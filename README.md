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
- **mlflow** — tracking server + model registry, artifacts in GCS
  (`gs://geo-mlops-artifacts-rsmrtk`)

All four sync with `prune: true, selfHeal: true` — manual cluster drift gets
reverted automatically.

## Deploying

Nothing here is applied by hand. To ship a new app version, push to
`geo-mlops` `main` — CI builds, tags, and updates `envs/prod/values.yaml`
in this repo. To change infra (resources, ingress host, chart config),
edit the relevant chart or `envs/prod/values.yaml` directly and merge —
ArgoCD does the rest.

## Stack

Helm · ArgoCD · Kubernetes · MLflow · GCS
