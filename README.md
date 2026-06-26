# jenkins-2026-gitops-config

> **GitOps configuration repository** for the [`jenkins-2026`](https://github.com/nubenetes/jenkins-2026) proof-of-concept.
>
> This repo is the **Git source of truth for ArgoCD**. Jenkins CI writes image tags here; ArgoCD reads them and reconciles the cluster state. You do not deploy anything manually from this repo.

> ## ⚠️ `main` is CI-writable — do NOT require pull requests on it
>
> The Jenkins **GitOps Update** pipeline stage pushes image-tag bumps **directly** to `main` (`git push origin main`). `main` is therefore protected only against **force-pushes/deletions**, **not** with *require-a-pull-request*. If you enable "Require a pull request before merging" on `main`, the CI's PAT-authenticated push is rejected (an admin PAT does **not** bypass branch protection), so **every deploy fails** at GitOps Update and the image tags freeze. This is deliberate and the **opposite** of the [`jenkins-2026`](https://github.com/nubenetes/jenkins-2026) infra repo, whose `main` is strict-GitFlow-protected (PR-from-`develop`-only) because it is human-reviewed. Here, image-tag bumps are machine-managed, not human-reviewed — so `main` must accept the CI's direct push.

## Table of Contents
- [Golden Path IDP Infrastructure](#golden-path-idp-infrastructure)
- [Relationship to `jenkins-2026`](#relationship-to-jenkins-2026)
- [Repository Layout](#repository-layout)
- [How Image Tags Are Updated](#how-image-tags-are-updated)
- [ArgoCD Applications](#argocd-applications)
  - [`microservices` ApplicationSet](#microservices-applicationset)
  - [Standalone Applications](#standalone-applications)
- [Helm Chart: `helm/microservices`](#helm-chart-helmmicroservices)
  - [Key values schema](#key-values-schema)
  - [Environments](#environments)
- [Postgres (CNPG)](#postgres-cnpg)
- [NetworkPolicies (zero-trust)](#networkpolicies-zero-trust)
- [Branch Strategy](#branch-strategy)
  - [Why only the `main` branch?](#why-only-the-main-branch)
  - [Would a `develop` branch make sense?](#would-a-develop-branch-make-sense)
- [OTel Auto-Instrumentation](#otel-auto-instrumentation)
- [Related Repositories](#related-repositories)
- [Setup & Forking Guide](#setup--forking-guide)
- [Release & Versioning Strategy](#release--versioning-strategy)
- [Git History and Privacy](#git-history-and-privacy)
- [Do Not Edit Manually](#do-not-edit-manually)

## Golden Path IDP Infrastructure

This repository defines the GitOps state for the modernized **Internal Developer Platform (IDP)** architecture on GKE.

### Decoupled Core Components
In alignment with 2026 Cloud-Native best practices, all platform infrastructure manifests are decoupled from CI build execution and managed via GitOps:
* **Elastic Karpenter Autoscaling**: Configured with dynamic `NodePool` and `GCPNodeClass` manifests under `infrastructure/karpenter/` in the main repo to handle autoscaling of ephemeral build agents on Spot instances.
* **GKE Gateway API Routing**: Secure HTTPS traffic routing for Jenkins and Headlamp is mapped under `infrastructure/gateway/` using native `Gateway`, `HTTPRoute`, and `BackendTLSPolicy` (zero-trust TLS to pods).
* **Workload-Aware scheduling & Security**: Maps K8s v1.36 `PodGroup` (Gang scheduling) and `ConstrainedImpersonation` policies for Headlamp UI users.

---

## Relationship to `jenkins-2026`

```
+--------------------------------------------------------------------+
|                nubenetes/jenkins-2026 (infra repo)                 |
|                                                                    |
|  scripts/        --- bootstrap cluster, install Jenkins/ArgoCD     |
|  jenkins/        --- JCasC, Job DSL, shared pipeline library       |
|  helm/           --- Helm charts for supporting services           |
|  argocd/         --- ApplicationSet/Application manifests          |
|  observability/  --- OTel collector, Grafana dashboards            |
+------------------------+-------------------------------------------+
                         | scripts/08.5-argocd.sh registers
                         | THIS repo as ArgoCD source
                         v
+--------------------------------------------------------------------+
|          nubenetes/jenkins-2026-gitops-config (this repo)          |
|                                                                    |
|  argocd/            --- Application / AppSet manifests (deployed   |
|                         FROM infra repo, stored here for clarity)  |
|  helm/microservices/--- Helm chart + env values files              |
|    values-stable.yaml<- Jenkins writes image tags here             |
+--------------------------------------------------------------------+
```

| Action | Who does it | Where |
|--------|------------|-------|
| Bootstrap cluster & install ArgoCD | `scripts/08.5-argocd.sh` | `jenkins-2026` |
| Register this repo in ArgoCD | `scripts/08.5-argocd.sh` | `jenkins-2026` |
| Build & push container images | Jenkins pipeline (`MicroservicesPipeline`) | `jenkins-2026/vars/` |
| **Write image tag to values file** | Jenkins (`vars/microservicesDeploy.groovy`) | **this repo** |
| Detect tag change & deploy to cluster | ArgoCD (automated sync) | cluster |
| Grafana dashboard push | `scripts/07-grafana-dashboards.sh` | `jenkins-2026` |

---

## Repository Layout

```
jenkins-2026-gitops-config/
├── argocd/
│   ├── microservices-appset.yaml   # ApplicationSet: generates microservices-stable Application
│   ├── microservices-project.yaml  # AppProject: scope for the microservices Application
│   ├── headlamp-app.yaml           # Application: Headlamp Kubernetes UI
│   ├── pgadmin-app.yaml            # Application: pgAdmin 4 Postgres UI
│   └── cnpg-app.yaml               # Application: CloudNative-PG Operator (CNPG)
└── helm/
    └── microservices/
        ├── Chart.yaml                 # Helm chart metadata
        ├── values.yaml                # Base defaults / schema documentation
        ├── values-stable.yaml         # Stable env (namespace: microservices, branch: main)
        ├── values-develop.yaml        # Dormant develop-tier values (only used if a develop track is re-enabled)
        └── templates/
            ├── deployment.yaml        # Deployment per service in .Values.services
            ├── service.yaml           # ClusterIP Service
            ├── ingress.yaml           # Ingress (enabled per platform)
            ├── route.yaml             # Gateway API HTTPRoute / OpenShift Route (per platform)
            ├── instrumentation.yaml   # OTel Instrumentation CR (auto-instruments JVM services)
            ├── postgres.yaml          # CNPG Cluster & Pooler CR per service
            ├── networkpolicies.yaml   # Zero-trust: default-deny + gateway / microservice / postgres policies
            ├── logback-configmap.yaml # ECS-JSON structured-logging config
            ├── gateway-cache-patch.yaml # gateway Hazelcast cache config
            ├── limitrange.yaml        # Default container resource limits
            ├── resourcequota.yaml     # Namespace resource cap
            └── _helpers.tpl           # Shared template helpers
```

---

## How Image Tags Are Updated

Jenkins runs the `microservicesDeploy.groovy` shared-library step on every successful build:

```mermaid
sequenceDiagram
    autonumber
    actor Jenkins as Jenkins CI Pipeline
    participant GitOps as jenkins-2026-gitops-config (Git)
    participant ArgoCD as ArgoCD Server
    participant Cluster as Kubernetes Cluster

    Jenkins->>GitOps: 1. Clone GitOps repo
    Jenkins->>GitOps: 2. Update tag in values-stable.yaml
    Jenkins->>GitOps: 3. git commit & push tag
    Jenkins->>ArgoCD: 4. Sync microservices-stable application
    ArgoCD->>Cluster: Reconcile manifests to new image tag
    Cluster-->>ArgoCD: Pods running & Healthy
    ArgoCD-->>Jenkins: Sync finished & Healthy
```

The updated `values-stable.yaml` is the **only file Jenkins ever modifies** in this repo. Everything else is managed by humans or by `scripts/08.5-argocd.sh` in the infra repo.

---

## ArgoCD Applications

All four Applications are **installed by `scripts/08.5-argocd.sh`** in the infra repo. The manifests live here so ArgoCD can self-heal them via the `microservices` AppProject.

### `microservices` ApplicationSet
Generates the stable application:

| Generated App | Namespace | Values file | Branch |
|---------------|-----------|-------------|--------|
| `microservices-stable` | `microservices` | `values-stable.yaml` | `main` |

It uses `prune: true` + `selfHeal: true`. Only the **stable** application is generated; the develop tier is **disabled by default** (the AppSet emits a `develop` element only when `microservices.developTrackEnabled` is set in the infra repo). The dormant `values-develop.yaml` stays in the chart for when that track is re-enabled — see [Branch Strategy](#branch-strategy).

### Standalone Applications

| Application | Source | Target Namespace | Notes |
|-------------|--------|-----------------|-------|
| `headlamp` | `helm/headlamp/values.yaml` (infra repo) | `headlamp` | Kubernetes UI, Google OIDC |
| `pgadmin` | `helm/pgadmin/` (infra repo) | `pgadmin` | Postgres admin UI |
| `cnpg-operator` | `cloudnative-pg/cloudnative-pg` | `cnpg-system` | CNPG, ServerSideApply |

---

## Helm Chart: `helm/microservices`

A single chart renders all services defined in `values.services.*`. Each service entry specifies its image tag and per-service config; the chart generates a `Deployment`, `Service`, `Instrumentation` CR, and `PostgresCluster` CR for each.

### Key values schema

```yaml
global:
  platform: gke          # gke | eks | aks | openshift

namespace: microservices  # overridden per-env by values-stable.yaml
env: stable               # "stable" → deployment.environment OTel attribute
registry: ghcr.io/nubenetes/jenkins-2026-microservices
imagePullSecret: ghcr-credentials

otel:
  collectorEndpoint: http://otel-collector-gateway.observability.svc.cluster.local:4317

services:
  gateway:
    type: java
    image:
      repository: gateway
      tag: main           # ← Jenkins writes a new SHA here on every build
    port: 8080
    healthPath: /management/health
    resources:
      requests: { cpu: 100m, memory: 256Mi }
      limits:   { cpu: 500m, memory: 512Mi }
    env:
      - name: SPRING_PROFILES_ACTIVE
        value: prod,api-docs
```

### Environments

| File | `env` | `namespace` | ArgoCD App |
|------|-------|-------------|-----------|
| `values-stable.yaml` | `stable` | `microservices` | `microservices-stable` |

The `env` value becomes the `deployment.environment` OTel resource attribute on every trace/metric/log emitted by deployed services, enabling environment filtering in Grafana dashboards.

---

## Postgres (CNPG)

Each service in `.Values.services` gets CNPG `Cluster` and `Pooler` CRs templated by `templates/postgres.yaml` (the template ranges over **every** service unconditionally; per-service `postgres.storageSize` / `walStorageSize` are the only optional knobs). The CloudNative-PG Operator (installed via the `cnpg-operator` Application) reconciles these CRs into:

- A highly-available **PostgreSQL 18.3** database tier — **3 instances**, zonal anti-affinity, dynamic primary promotion. The image is **pinned** explicitly (`spec.imageName`, default `ghcr.io/cloudnative-pg/postgresql:18.3-system-trixie`, overridable via `global.postgresImage`) so the DB version is reproducible; bump it deliberately
- Connection pooling managed via native PgBouncer pooler deployments
- Automated Barman Object Store backups targeting Google Cloud Storage (GCS)
- A secret `postgres-<service>-app` injected into the service pod via `SPRING_DATASOURCE_URL` or `SPRING_R2DBC_URL`

Two clusters are provisioned in total — one per service in the stable environment:

| Cluster | Namespace |
|---------|-----------|
| `postgres-gateway` | `microservices` |
| `postgres-jhipstersamplemicroservice` | `microservices` |

---

## NetworkPolicies (zero-trust)

`templates/networkpolicies.yaml` ships a default-deny posture for the `microservices`
namespace (enforced by GKE **Dataplane V2 / Cilium-eBPF** in the infra repo). Four
policies:

| Policy | Applies to | Key ingress | Key egress |
|---|---|---|---|
| `default-deny` | all pods | none | CoreDNS (`kube-system:53`) only |
| `gateway-policy` | `gateway` pod | **8080** (from the Gateway/LB) | jhipster **8081**, its Postgres **5432**, OTLP `observability` **4317/4318** |
| `microservice-policy` | `jhipstersamplemicroservice` pod | **8081** from the `gateway` pod **and** the **`tekton-ci` + `jenkins`** namespaces (so CI smoke tests can hit `/management/health`) | its Postgres **5432**, OTLP **4317/4318** |
| `postgres-policy` | `cnpg.io/cluster` pods | **5432** from the app pods, `pgadmin` ns, intra-cluster | CNPG replication + **443** |

The CI-namespace ingress on 8081 is what lets the Jenkins/Tekton smoke stage reach the
microservice under enforcement (see [`jenkins-2026` docs/501](https://github.com/nubenetes/jenkins-2026/blob/main/docs/501-PLATFORM_OPERATIONS.md#networkpolicy-matrix)).

---

## Branch Strategy

The GitOps repository uses the `main` branch to target `microservices-stable` deployments. Jenkins updates `helm/microservices/values-stable.yaml` on `main` to promote new image versions. The `develop` tier is **disabled by default** (only `microservices-stable` is generated); its `values-develop.yaml` stays dormant in the chart and is activated only when `microservices.developTrackEnabled` is set in the infra repo.

### Why only the `main` branch?

1. **Single Environment Target**: In this unified model the develop tier is disabled by default, leaving a single active target namespace (`microservices`); the develop track can be re-enabled (see below).
2. **Simplified Promotion**: The Jenkins CI pipeline writes image tags directly inside [values-stable.yaml](helm/microservices/values-stable.yaml) on the `main` branch of the GitOps repository.

### Would a `develop` branch make sense?

Yes, but **only if you restore a multi-environment deployment model** (e.g., dev/staging vs. stable namespaces):

* **Testing Infrastructure Changes**: If developers need to test Helm chart updates (e.g., resource limits, new environment variables, or sidecar additions) in a sandbox (`develop`) namespace before promoting them to stable (`main`), they would push changes to the `develop` branch of the GitOps repo first for verification.
* **Tracking Parallel Code Tracks**: If upstream repositories build from both a `develop` branch (dev builds) and a `main` branch (stable releases), Jenkins would commit dev tags to a `values-develop.yaml` on the GitOps `develop` branch (synced to a dev namespace), and stable tags to [values-stable.yaml](helm/microservices/values-stable.yaml) on the GitOps `main` branch (synced to the stable namespace).

---

## OTel Auto-Instrumentation

The `templates/instrumentation.yaml` template creates an `Instrumentation` CR (managed by the OTel Operator, installed by `scripts/03-observability.sh`). This automatically attaches the OTel Java agent to every Spring Boot service pod via a mutating webhook — no changes to application code or Docker images are required.

The agent is configured with:
- `OTEL_EXPORTER_OTLP_ENDPOINT` → the in-cluster OTel Collector gateway
- `OTEL_RESOURCE_ATTRIBUTES` → `service.name`, `service.namespace`, `deployment.environment`
- `OTEL_INSTRUMENTATION_LOGBACK_APPENDER_ENABLED=true` → injects `trace_id` into log lines for Loki correlation

---

## Related Repositories

| Repository | Role |
|-----------|------|
| [`nubenetes/jenkins-2026`](https://github.com/nubenetes/jenkins-2026) | **Infra repo** — cluster bootstrap, Jenkins, ArgoCD, Observability, shared pipeline library |
| [`nubenetes/jenkins-2026-gitops-config`](https://github.com/nubenetes/jenkins-2026-gitops-config) | **This repo** — GitOps state: Helm chart, env values, ArgoCD manifests |
| [`spring-microservices/spring-microservices-microservices`](https://github.com/spring-microservices/spring-microservices-microservices) | Upstream Spring Boot microservices source code |
| [`spring-microservices/spring-microservices-angular`](https://github.com/spring-microservices/spring-microservices-angular) | Upstream Angular gateway UI source code |

---

## Setup & Forking Guide

If you are setting up this PoC for yourself or your organization, you must fork this configuration repository along with the main [infrastructure repository](https://github.com/nubenetes/jenkins-2026).

1. **Fork the Repository**: Fork this repository (`jenkins-2026-gitops-config`) to your GitHub account/organization.
2. **Update Main Infra Configuration**: In your fork of the infra repository (`jenkins-2026`), update `config/config.yaml` to point `jenkins.selfRepoUrl` and `microservices.git.org` to your own forks.
3. **Configure Git Credentials**: Ensure you set `GIT_USERNAME` and `GIT_TOKEN` secrets in the infra repository Actions settings, as the Jenkins pipeline dynamically clones and commits updated image tags to this GitOps repository.

---

## Release & Versioning Strategy

This GitOps configuration repository is released and tagged in lockstep with the main [infrastructure repository](https://github.com/nubenetes/jenkins-2026):

* **Git Tags**: Every stable release of the infrastructure stack (e.g. `v0.9.0`) has a matching tag `v0.9.0` applied to both repositories.
* **Release Flow**:
  1. Infrastructure modifications and Helm chart configurations are tested on `develop`.
  2. A Pull Request is opened to merge `develop` into `main`.
  3. Once merged, a GitHub Release is drafted explaining the changes, and the tag is pushed to remote.

---

## Git History and Privacy

> [!NOTE]
> This repository's Git history has been fully rewritten and sanitized to remove any private Google identity email addresses and account IDs.
> When collaborating or contributing to this repository, please configure your local Git author settings to use GitHub's private email alias (e.g., `username@users.noreply.github.com`) to avoid accidentally leaking private email addresses in future commits.

---

## Do Not Edit Manually

> [!CAUTION]
> `helm/microservices/values-stable.yaml` is **continuously overwritten by Jenkins CI** on every successful build. Manual edits to `services.<name>.image.tag` will be overwritten by the next pipeline run. All other fields (resources, env vars, healthPath) are safe to edit.

For all other infrastructure changes — Jenkins config, observability stack, ArgoCD setup, Helm charts for Headlamp/pgAdmin — make changes in [`nubenetes/jenkins-2026`](https://github.com/nubenetes/jenkins-2026) and re-run the relevant script or GitHub Actions workflow.
