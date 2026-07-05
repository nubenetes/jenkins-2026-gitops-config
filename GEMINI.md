# Gemini Developer Assistant Guide - jenkins-2026-gitops-config

Welcome! This repository holds the **GitOps target configurations** reconciled by ArgoCD in the JHipster-based microservices architecture. It contains Kubernetes deployment patterns, Helm chart configuration parameters, and application mappings.

---

## 🏗️ Repository Architecture

The project is structured as follows:

```
├── argocd/
│   ├── headlamp-app.yaml        # ArgoCD Application definition for Headlamp UI
│   ├── pgadmin-app.yaml         # ArgoCD Application definition for pgAdmin 4
│   ├── cnpg-app.yaml            # ArgoCD Application definition for Postgres Operator
│   ├── microservices-project.yaml # ArgoCD AppProject restricting sync namespaces
│   └── microservices-appset.yaml  # ArgoCD ApplicationSet generating microservices app instances
└── helm/
    └── microservices/           # Helm files for the microservice deployments
        ├── Chart.yaml
        ├── templates/
        ├── values.yaml          # Base values for microservices Helm deployment
        ├── values-stable.yaml   # Stable environment-specific overrides (namespace: microservices)
        └── values-develop.yaml  # Dormant develop-tier overrides (namespace: microservices-develop)
```

---

## 🚀 GitOps Promotion Flow

1. **CI Trigger**: the active CI engine (Jenkins by default; Tekton, GitHub Actions/ARC or Argo Workflows via the infra repo's `ci.engine`) compiles code, builds container images, and pushes them to GitHub Packages (`ghcr.io/nubenetes/jenkins-2026-microservices/...`).
2. **Tag Promotion**: the engine's GitOps step (Jenkins `microservicesDeploy.groovy`, Tekton `gitops-deploy`, the GHA workflow's GitOps-bump step, or the Argo Workflows `gitops-deploy` step) writes the newly built image tags to [`helm/microservices/values-stable.yaml`](helm/microservices/values-stable.yaml), commits, and pushes.
3. **ArgoCD Sync**: ArgoCD detects the change in this repository, automatically syncs the live Kubernetes cluster to pull the new images, and applies any updated Kubernetes resources.

---

## 💡 Troubleshooting and Optimization Tips

1. **Develop tier disabled by default**: Only [`values-stable.yaml`](helm/microservices/values-stable.yaml) / the `microservices` namespace is active; the AppSet generates just `microservices-stable`. [`values-develop.yaml`](helm/microservices/values-develop.yaml) remains **dormant** in the chart and is activated only when `microservices.developTrackEnabled` is set in the infra repo — don't assume a develop deployment exists unless that flag is on.
2. **Checking Application Status**:
   ```bash
   kubectl get applications -n argocd
   ```
   Check if applications are `Synced` and `Healthy`.
3. **ApplicationSet Key Paths**: The ApplicationSet ([`microservices-appset.yaml`](argocd/microservices-appset.yaml)) loops over **environments** in its list generator (`stable` always; `develop` only when the develop track is enabled), pointing each generated Application at `helm/microservices` with the matching `values-<env>.yaml`. The **services** are defined inside those values files, not in the generator.
