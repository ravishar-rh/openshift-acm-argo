# OpenShift GitOps Knowledge Base

Reference and patterns for OpenShift GitOps (Argo CD) and automated deployment with Application Sets.

---

## 1. OpenShift GitOps Overview

- **OpenShift GitOps** is the Red Hat offering of Argo CD on OpenShift, installed via the OpenShift GitOps Operator.
- **Namespace**: Controller and Argo CD apps typically run in `openshift-gitops`.
- **GitOps**: Desired state lives in Git; the cluster is continuously reconciled to match that state (declarative, auditable, rollback via Git).

### Key Components

| Component | Purpose |
|-----------|---------|
| Argo CD | Syncs Kubernetes/OpenShift resources from Git (and other sources) to the cluster |
| Application Set controller | Generates multiple Argo CD Applications from templates + generators (Git, Cluster, List, etc.) |
| Red Hat OpenShift GitOps Operator | Installs and manages Argo CD and Application Set on OpenShift |

---

## 2. Application Sets

- **ApplicationSet** = one CR that produces many **Application** resources.
- **Generators** provide the list of parameters (e.g. directories, file contents, clusters).
- **Template** uses those parameters to fill in each Application’s `spec` (repo, path, destination, etc.).

### Generator Types (summary)

| Generator | Use case |
|-----------|----------|
| **Git – directories** | One Application per directory under a path (e.g. `apps/*`) |
| **Git – files** | One Application per element in a YAML/JSON file (e.g. clusters, apps list) |
| **List** | Static list of parameters in the ApplicationSet spec |
| **Clusters** | One Application per cluster registered in Argo CD |
| **Matrix** | Combine two or more generators (e.g. apps × clusters) |

---

## 3. Git Generator – Directories

- Scans a path in a Git repo; each **directory** becomes one set of parameters.
- Parameters available in the template: `path`, `path.basename`, `path[0]`, etc.

Example: one Application per child of `apps/`:

```yaml
generators:
  - git:
      repoURL: https://github.com/org/repo
      revision: HEAD
      directories:
        - path: apps/*
```

Template can use `{{path}}` (e.g. `apps/frontend`) and `{{path.basename}}` (e.g. `frontend`).

---

## 4. Git Generator – Files

- Reads a **file** from Git (YAML/JSON/JSON Lines); each element becomes one parameter set.
- Use when the list of “what to deploy” is maintained in a single file (e.g. clusters, operators).

Example: one Application per cluster defined in `config/clusters.yaml`:

```yaml
generators:
  - git:
      repoURL: https://github.com/org/repo
      revision: HEAD
      files:
        - path: config/clusters.yaml
```

Keys in each element (e.g. `cluster`, `url`, `namespace`) are available as `{{cluster}}`, `{{url}}`, `{{namespace}}` in the template.

---

## 5. Cluster Deployment Automation (Concept)

- **Cluster provisioning** (creating new OpenShift clusters) is often done outside Argo CD (e.g. ACM, OCM, installer, Terraform).
- **GitOps role**: once a cluster exists and is registered with the hub’s Argo CD, Application Sets can:
  - Deploy the same set of apps/operators to every cluster.
  - Use a **Git file generator** with a `clusters.yaml` (or similar) that lists cluster API URLs and namespaces.
- **Flow**: Add a new cluster to `config/clusters.yaml` (and ensure it’s registered in Argo CD) → Application Set creates/updates the Application(s) for that cluster automatically.

---

## 6. Cluster Operators Automation (Concept)

- **Operators** on OpenShift are typically installed via:
  - Cluster-scoped: `Subscription` + `OperatorGroup` (and sometimes `ClusterServiceVersion`).
  - Or OLM channels and approval strategy.
- **GitOps approach**:
  - Store operator manifests (Subscription, OperatorGroup, etc.) in Git, e.g. under `operators/<operator-name>/` or a single file list.
  - Use an **Application Set** (e.g. Git directory generator for `operators/*` or Git file generator for `config/operators.yaml`) to create one Application per operator.
  - Each Application points to the path (or Helm chart) that installs that operator; sync policy can be automated.

---

## 7. Sync Policies

- **Manual**: User triggers sync (default).
- **Automated**: Argo CD syncs when Git (or source) changes; optional `prune` and `selfHeal` for drift correction.

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

---

## 8. Multi-Cluster and Destination

- **destination.server**: In-cluster use `https://kubernetes.default.svc`; for another cluster use the cluster’s API URL (as registered in Argo CD) or the Argo CD cluster name.
- **destination.namespace**: Target namespace on that cluster.
- For OpenShift GitOps on a hub, ensure each managed cluster is added (e.g. `argocd cluster add` or equivalent) so Application Sets can target them.

---

## 9. Repo Layout (This Knowledge Base Repo)

```
.
├── knowledge.md                 # This file
├── README.md                    # Repo overview and usage
├── bootstrap/                   # ApplicationSet definitions (entry point)
│   ├── acm-hub-application-set.yaml    # Git file → config/acm-hubs.yaml
│   ├── clusters-application-set.yaml   # Git file → config/clusters.yaml
│   └── operators-application-set.yaml  # Git file → config/operators.yaml
├── config/                      # Generator input (Git file generator)
│   ├── acm-hubs.yaml            # Clusters that run ACM hub (for cluster automation)
│   ├── clusters.yaml            # List of clusters for deployment
│   └── operators.yaml           # List of operators to deploy
├── acm-hub/                     # ACM operator + MultiClusterHub (hub for new clusters)
├── clusters/
│   └── base/                    # Kustomize base deployed to every cluster
└── operators/                   # One directory per operator (path in operators.yaml)
    ├── openshift-pipelines/
    ├── openshift-gitops/
    └── red-hat-codeready-workspaces/
```

- **acm-hubs.yaml** drives which cluster(s) get the ACM hub (operator + MultiClusterHub); the hub is used to automate deployment of new clusters.
- **clusters.yaml** drives which clusters get the base deployment (one Application per cluster).
- **operators.yaml** drives which operators are installed (one Application per operator; path points to **operators/** subdirs).

---

## 10. Bootstrapping OpenShift GitOps

1. Install OpenShift GitOps operator from OperatorHub.
2. Create or use the default `ArgoCD` instance in `openshift-gitops`.
3. Apply ApplicationSet(s) in `openshift-gitops` that point to this repo.
4. Argo CD creates Applications; those Applications sync from this repo (and any referenced repos) to deploy clusters’ workloads and operators.

---

## 11. References

- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Argo CD Application Set](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [OpenShift GitOps](https://docs.openshift.com/container-platform/latest/cicd/gitops/understanding-openshift-gitops.html)
