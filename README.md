# ACM Bootstrap

Common prerequisites for ACM-managed GitOps deployments. These resources must
be applied on the hub cluster before creating ApplicationSets or governance
policies.

## Structure

```
acm_bootstrap/
├── applications/    # GitOps application prerequisites (ManagedClusterSetBinding, Placement, GitOpsCluster)
└── governance/      # Governance/policy prerequisites (placeholder)
```

## Applications

Bootstrap resources required before deploying applications via ArgoCD
ApplicationSets (push or pull model):

- **ManagedClusterSetBinding**: binds cluster sets to the `openshift-gitops` namespace.
- **Placement**: selects clusters for GitOpsCluster registration and app targeting.
- **GitOpsCluster**: registers managed clusters in ArgoCD as cluster secrets.

See [applications/README.md](applications/README.md) for step-by-step instructions.

## Governance

_(placeholder: governance bootstrap resources will be added here)_

---

# ACM Bootstrap: GitOps Application Prerequisites

Before deploying applications via ArgoCD ApplicationSets, the following
resources must be created on the hub cluster. These are common to **both**
push and pull delivery models.

## Step 1: Bind ManagedClusterSets to the openshift-gitops namespace

> **Manual step**: corresponds to `bootstrap/00-managedclustersetbinding.yaml`

The `Placement` in `openshift-gitops` can only see clusters from sets that are
bound to that namespace.

```bash
cat <<'EOF' | oc apply -f -
---
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: global
  namespace: openshift-gitops
spec:
  clusterSet: global
EOF
```

Verify:

```bash
oc get managedclustersetbinding -n openshift-gitops
```

---

## Step 2: Create the Placements

> **Manual step**: corresponds to `bootstrap/01-placement-gitops.yaml` and
> `bootstrap/01-placement-todo-app.yaml`

Two Placements are used:

- **`clusters-for-gitops`**: used by GitOpsCluster to register all clusters
  in the set as ArgoCD cluster secrets. No predicates (selects all).
- **`todo-app-placement`**: used by the ApplicationSet to select target
  clusters for the application. Filters by `environment: prod`.

```bash
cat <<'EOF' | oc apply -f -
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: clusters-for-gitops
  namespace: openshift-gitops
spec:
  tolerations:
    - key: cluster.open-cluster-management.io/unreachable
      operator: Exists
    - key: cluster.open-cluster-management.io/unavailable
      operator: Exists
  clusterSets:
    - global
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: todo-app-placement
  namespace: openshift-gitops
spec:
  numberOfClusters: 2
  clusterSets:
    - global
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: environment
              operator: In
              values:
                - prod
EOF
```

Verify the clusters are selected:

```bash
oc get placementdecision -n openshift-gitops \
  -l cluster.open-cluster-management.io/placement=todo-app-placement \
  -o jsonpath='{.items[*].status.decisions[*].clusterName}'
```

---

## Step 3: Create the GitOpsCluster

> **Manual step**: corresponds to `bootstrap/02-gitopscluster.yaml`

`GitOpsCluster` tells ACM to register each cluster selected by the
`clusters-for-gitops` Placement as an ArgoCD cluster secret. This is what
allows ArgoCD to deploy to managed clusters.

When this resource is created, ACM:
1. Creates an ArgoCD cluster **Secret** for each managed cluster selected by
   the Placement (containing the cluster name, API server URL, and credentials).
2. Creates the **`acm-placement` ConfigMap** that tells the
   `clusterDecisionResource` generator how to parse `PlacementDecision`
   resources.

```bash
cat <<'EOF' | oc apply -f -
---
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
  name: gitops-clusters
  namespace: openshift-gitops
spec:
  argoServer:
    cluster: local-cluster
    argoNamespace: openshift-gitops
  placementRef:
    kind: Placement
    apiVersion: cluster.open-cluster-management.io/v1beta1
    name: clusters-for-gitops
    namespace: openshift-gitops
EOF
```

Verify ArgoCD cluster secrets are created:

```bash
oc get secrets -n openshift-gitops -l argocd.argoproj.io/secret-type=cluster
```

Verify the `acm-placement` ConfigMap exists:

```bash
oc get configmap acm-placement -n openshift-gitops -o yaml
```
