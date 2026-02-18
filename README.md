# OpenShift GitOps – Cluster & Operator Automation

This repo uses **Argo CD Application Sets** with the **Git Generator** to automate:

1. **ACM hub** – Deploy Red Hat Advanced Cluster Management (ACM) on a cluster to use it as the hub for automating deployment of new clusters.
2. **Cluster deployment** – Deploy the same base resources to multiple clusters.
3. **Cluster operators** – Install and manage OpenShift operators from Git in an automated fashion.

## Knowledge Base

See **[knowledge.md](./knowledge.md)** for OpenShift GitOps concepts, Application Sets, and Git Generator patterns.

## Repo Layout

```
.
├── knowledge.md              # OpenShift GitOps knowledge base
├── bootstrap/                # ApplicationSet manifests (apply these once)
│   ├── acm-hub-application-set.yaml
│   ├── clusters-application-set.yaml
│   └── operators-application-set.yaml
├── config/                   # Generator input (drives what gets deployed)
│   ├── acm-hubs.yaml         # Clusters that run ACM hub → one Application per hub
│   ├── clusters.yaml         # List of clusters → one Application per cluster
│   └── operators.yaml        # List of operators → one Application per operator
├── acm-hub/                  # ACM operator + MultiClusterHub (for hub cluster)
├── clusters/
│   └── base/                 # Kustomize base deployed to every cluster
└── operators/                # One directory per operator (Subscription, OperatorGroup, etc.)
    ├── openshift-pipelines/
    ├── openshift-gitops/
    └── red-hat-codeready-workspaces/
```

## How It Works

### ACM hub (Git file generator)

- **Generator**: `config/acm-hubs.yaml` – each entry (hub name, cluster URL, namespace) becomes one Application.
- **ApplicationSet**: `bootstrap/acm-hub-application-set.yaml` deploys the ACM operator and MultiClusterHub to each listed cluster.
- **Purpose**: The hub cluster runs ACM and can later be used to automate creation/import of new clusters (provisioning, policies, governance). Add or change hub targets by editing `config/acm-hubs.yaml`.

### Cluster deployment (Git file generator)

- **Generator**: `config/clusters.yaml` – each entry (cluster name, API URL, namespace) becomes one set of parameters.
- **ApplicationSet**: `bootstrap/clusters-application-set.yaml` creates one Application per cluster.
- **Template**: Each Application syncs `clusters/base` to that cluster’s namespace. Add new clusters by editing `config/clusters.yaml`; no need to create Applications by hand.

### Cluster operators (Git file generator)

- **Generator**: `config/operators.yaml` – each entry (name, path, namespace, channel) becomes one set of parameters.
- **ApplicationSet**: `bootstrap/operators-application-set.yaml` creates one Application per operator.
- **Template**: Each Application syncs the operator’s path (e.g. `operators/openshift-pipelines`) to the cluster where OpenShift GitOps runs. Add or remove operators by editing `config/operators.yaml` and adding the corresponding manifests under `operators/<name>/`.

## Bootstrap (One-Time)

1. **Install OpenShift GitOps** (Argo CD) on your cluster and ensure the Application Set controller is running.

2. **Point ApplicationSets at this repo**  
   Edit `repoURL` in both files under `bootstrap/` to your Git repo URL (e.g. your fork or org clone of this repo).

3. **Apply the ApplicationSets** (from this repo root):

   ```bash
   oc apply -f bootstrap/acm-hub-application-set.yaml -n openshift-gitops
   oc apply -f bootstrap/clusters-application-set.yaml -n openshift-gitops
   oc apply -f bootstrap/operators-application-set.yaml -n openshift-gitops
   ```

4. **Register clusters**:  
   Each cluster in `config/clusters.yaml` and `config/acm-hubs.yaml` must be registered in Argo CD (e.g. `argocd cluster add` or via ACM). Use the same API URL as in the config files.

5. **Configure config files**: Set real cluster API URLs and namespaces in `config/acm-hubs.yaml`, `config/clusters.yaml`, and `config/operators.yaml`. Update `repoURL` in each ApplicationSet under `bootstrap/` to your Git repo.

## Adding an ACM hub (for cluster automation)

1. Ensure the target cluster is registered in Argo CD.
2. Add an entry to `config/acm-hubs.yaml` with `hub_name`, `url`, and `namespace` (use `open-cluster-management`).
3. Commit and push; the Application Set will create the Application and deploy ACM (operator + MultiClusterHub) to that cluster. Once the hub is ready, you can use ACM to create or import new clusters.

## Adding a New Cluster

1. Register the cluster with Argo CD.
2. Add an entry to `config/clusters.yaml` with `cluster`, `url`, and `namespace`.
3. Commit and push; the Application Set will create the new Application and sync `clusters/base` to that cluster.

## Adding a New Operator

1. Create a directory under `operators/<operator-name>/` with at least:
   - `namespace.yaml`
   - `operator-group.yaml`
   - `subscription.yaml`
   (and optionally `kustomization.yaml` if you use Kustomize.)
2. Add an entry to `config/operators.yaml` with `name`, `path` (e.g. `operators/<operator-name>`), `namespace`, and `channel`.
3. Commit and push; the Application Set will create the new Application and sync that path.

## Requirements

- OpenShift cluster(s) with OpenShift GitOps (Argo CD + Application Set) installed.
- This repo (or a fork) pushed to a Git server that Argo CD can reach.
- For multi-cluster: each target cluster registered in Argo CD.
