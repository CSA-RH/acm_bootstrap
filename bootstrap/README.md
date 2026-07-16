# Initial ACM Setup

This document describes the required configuration to install do the initial ACM configuration:

---

## 1. Install in you laptop the PolicyGenerator plug-in

The PolicyGenerator is a Kustomize plugin used only by the ACM
governance policy framework. It is required here to install the
OpenShift-Gitops operator on the Hub cluster via a governance policy.

**Install PolicyGenerator locally to your laptop:**

```bash
mkdir -p ${HOME}/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator

wget -O ${HOME}/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator/PolicyGenerator \
  https://github.com/open-cluster-management-io/policy-generator-plugin/releases/download/v1.16.0/linux-amd64-PolicyGenerator

chmod +x ${HOME}/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator/PolicyGenerator
```

## 2. Clone repository

```bash
git clone https://github.com/CSA-RH/acm_bootstrap.git
cd acm_bootstrap
```

## 3. Deploy the cluster topology sets and the policy namespace

This will create the following resources on the hub cluster:

- `Namespace`: policy namespace for ACM governance
- `Role` / `ClusterRole`: RBAC for ApplicationSet PlacementDecision access
- `Group`: ArgoCD admin group
- `ManagedClusterSet`: cluster sets for prod and dev environments
- `ManagedCluster`: managed cluster definitions (prod, dev)
- `ManagedClusterSetBinding`: binds cluster sets to namespaces

**These manifests define a sample cluster topology for a blank cluster.
If your environment already has ManagedClusterSets, ManagedClusters, or
RBAC configured, use your existing setup instead. Applying these
manifests over an existing configuration may overwrite or conflict with
your current setup.**

```bash
oc apply -f bootstrap/clustergroups/
```

## 4. Install GitOps operator on the hub:

```bash
kustomize build bootstrap/gitops --enable-alpha-plugins | oc apply -f -
oc get policy gitops-operator -n acm-policies -w
```

References:
- [ACM 2.16 Governance - Policy Generator](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html-single/governance/index#policy-generator)
- [open-cluster-management-io/policy-generator-plugin (GitHub)](https://github.com/open-cluster-management-io/policy-generator-plugin)
- [ArgoCD - ApplicationSet Git Generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/)
