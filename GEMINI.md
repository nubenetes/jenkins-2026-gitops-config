# Gemini Developer Assistant Guide - jenkins-2026-gitops-config

Welcome! This repository holds the **GitOps target configurations** reconciled by ArgoCD in the JHipster-based microservices architecture. It contains Kubernetes deployment patterns, Helm chart configuration parameters, and application mappings.

---

## 🏗️ Repository Architecture

The project is structured as follows:

```
├── argocd/
│   ├── headlamp-app.yaml        # ArgoCD Application definition for Headlamp UI
│   ├── pgadmin-app.yaml         # ArgoCD Application definition for pgAdmin 4
│   ├── pgo-app.yaml             # ArgoCD Application definition for Postgres Operator
│   ├── microservices-project.yaml # ArgoCD AppProject restricting sync namespaces
│   └── microservices-appset.yaml  # ArgoCD ApplicationSet generating microservices app instances
└── helm/
    └── microservices/           # Helm files for the microservice deployments
        ├── Chart.yaml
        ├── templates/
        ├── values.yaml          # Base values for microservices Helm deployment
        └── values-stable.yaml   # Stable environment-specific overrides (namespace: microservices)
```

---

## 🚀 GitOps Promotion Flow

1. **Jenkins Trigger**: Jenkins pipelines compile code, build container images, and push them to GitHub Packages (`ghcr.io/nubenetes/jenkins-2026-microservices/...`).
2. **Tag Promotion**: The Jenkins deployment stage (`microservicesDeploy.groovy`) writes the newly built image tags to `helm/microservices/values-stable.yaml` in this repository, commits, and pushes them.
3. **ArgoCD Sync**: ArgoCD detects the change in this repository, automatically syncs the live Kubernetes cluster to pull the new images, and applies any updated Kubernetes resources.

---

## 💡 Troubleshooting and Optimization Tips

1. **Pruned Sandbox**: The develop/sandbox environments have been completely pruned. All active deployments now go exclusively to the `microservices` namespace using `values-stable.yaml`. Do not try to restore or deploy files under a develop track.
2. **Checking Application Status**:
   ```bash
   kubectl get applications -n argocd
   ```
   Check if applications are `Synced` and `Healthy`.
3. **ApplicationSet Key Paths**: The ApplicationSet (`microservices-appset.yaml`) loops over the services defined in its generator, targeting the stable branch and namespace.
