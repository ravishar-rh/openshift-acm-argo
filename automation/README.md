# Automation

Ansible playbooks that deploy or manage resources on the cluster you are currently connected to (using `oc` context).

## Deploy OpenShift GitOps operator

Installs the OpenShift GitOps (Argo CD) operator on the current cluster using the manifests in `operators/openshift-gitops/`.

**Prerequisites**

- Ansible
- `oc` CLI installed and logged in to the target cluster (`oc login`)

**Usage**

```bash
# From repo root – will prompt to confirm before deploying
ansible-playbook automation/deploy-openshift-gitops.yaml

# Skip confirmation (e.g. CI or automation)
ansible-playbook automation/deploy-openshift-gitops.yaml -e "confirm_deploy=yes"
```

The playbook will:

1. Check that `oc` is available.
2. Show the current cluster (server URL, context, user).
3. **Check if OpenShift GitOps is already deployed** (subscription in `openshift-gitops` namespace). If so, it does nothing and exits.
4. Optionally pause for confirmation (unless `confirm_deploy=yes`).
5. Apply the operator manifests (Namespace, OperatorGroup, Subscription).
6. Wait for the subscription to have an installed CSV, then for the CSV to succeed (best effort).

**Verify**

```bash
oc get pods -n openshift-gitops
oc get route openshift-gitops-server -n openshift-gitops
```
