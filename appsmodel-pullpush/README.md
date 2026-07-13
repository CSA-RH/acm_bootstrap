# Integrating OpenShift GitOps and ACM with Managed Clusters

Common prerequisites for ACM-managed GitOps deployments. These resources must
be applied on the hub cluster before creating ApplicationSets or governance
policies.

> **Disclaimer:** This repository is for learning and testing purposes only.
> All configurations should be thoroughly tested in a non-production
> environment before applying to any production cluster.

---

## Deployment Models Overview

ACM supports three ways to deploy applications via ArgoCD ApplicationSets.
All three use the same `ApplicationSet` CRD with a `clusterDecisionResource`
generator, but differ in **how** and **where** the ArgoCD reconciliation
happens.

| | Push | Pull (ArgoCD Controller) | Pull (ArgoCD Agent) |
|---|---|---|---|
| **ArgoCD on hub** | Full instance (deploys to spokes) | ApplicationSet controller only | Full instance (principal) |
| **ArgoCD on spokes** | None | Full instance per spoke | Lightweight agent per spoke |
| **Deployment mechanism** | Hub pushes directly via cluster credentials | `ManifestWork` propagates Applications to spokes | gRPC sync between principal and agent |
| **Status feedback** | Real-time via hub ArgoCD | Periodic via `Multicluster1pplicationSetReport` | Real-time via gRPC (full ArgoCD UI integration) |
| **Network requirements** | Hub must reach spoke API servers | Spoke-to-Git repo access | Load Balancer for gRPC endpoint |
| **GitOps add-on** | Not required | `gitopsAddon.enabled: true` | `gitopsAddon.enabled: true` + `argoCDAgent.enabled: true` |
| **Scalability** | Limited by hub controller resources | Better (reconciliation distributed) | Better (reconciliation distributed) |

> **Source**: [Red Hat ACM 2.15 GitOps documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.15/html-single/gitops/index)

---

## Configuring Managed Clusters for OpenShift GitOps / ArgoCD

To configure and link OpenShift GitOps in ACM, we can register a set of one or more managed clusters to an instance of Argo CD or the OpenShift GitOps operator.

After registering, we can deploy applications to those clusters using ApplicationSets managing from the ACM Hub Applications. Then, we can set up a continuous GitOps environment to automate application consistency across clusters in development, staging, and production environments.

## 1. Clone repository

```bash
cd /tmp
git clone https://github.com/CSA-RH/acm_bootstrap.git
cd acm_bootstrap
```

### 2. ManagedClusterSet

First, we need to create managed cluster sets and add managed clusters to those managed cluster sets:
We will use in the examples the Global ClusterSet, wich is created automatically in ACM installation time.

### 3. ManagedClusterSetBinding

Create managed cluster set binding to the namespace where Argo CD or OpenShift GitOps is deployed.
Our ManagedClusterSetBinding object assigns ManagedClusterSet to the particular namespace so that Placements can discover clusters from that set. Our target namespace is openshift-gitops.

Apply resource:

```bash
oc apply -f appsmodel-pullpush/base/00-managedclustersetbinding.yaml
```

Example:
```yaml
# applications/base/00-managedclustersetbinding.yaml
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: global
  namespace: openshift-gitops
spec:
  clusterSet: global
```

### 4. Placement

In the namespace that is used in managed cluster set binding, create a placement custom resource to select a set of managed clusters to register to an ArgoCD or OpenShift GitOps operator instance. It can filter by the cluster sets or other predicates like, for example, labels. In our scenario, placement filters all the managed clusters, which have the `vendor: OpenShift` label.

Apply resource:

```bash
oc apply -f appsmodel-pullpush/base/01-placement-gitops.yaml
```

Example:
```yaml
# applications/base/01-placement-gitops.yaml
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
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            vendor: OpenShift
```

NOTE: Only OpenShift clusters are registered to an Argo CD or OpenShift GitOps operator instance, not other Kubernetes clusters.

Verify:

```bash
oc get managedclustersetbinding -n openshift-gitops
oc get placement clusters-for-gitops -n openshift-gitops
oc get placementdecision -n openshift-gitops \
  -l cluster.open-cluster-management.io/placement=clusters-for-gitops \
  -o jsonpath='{.items[*].status.decisions[*].clusterName}'
```

---

### 5. Configure GitOpsCluster

As of ACM 2.17, ACM supports three ways to deploy applications via ArgoCD ApplicationSets. Each of these models require a diffent GitOpsCluster configuration.
- Model 1: Push model
- Model 2: Pull model argocd controller
- Model 3: Pull model agent 

Apply the configuration according on the model your using.

#### Model 1: Push

The hub ArgoCD instance deploys application resources directly to managed
clusters using cluster credentials. This is the default model and the simplest
to set up.

Apply resource:
```bash
oc apply -f appsmodel-pullpush/overlay/push/02-gitopscluster.yaml
```

**How it works**:
1. `GitOpsCluster` registers managed clusters as ArgoCD cluster secrets on
   the hub.
2. ArgoCD ApplicationSet on the hub generates one `Application` per target
   cluster.
3. The hub ArgoCD Application controller pushes manifests directly to each
   managed cluster's API server.

Example:

```yaml
# applications/overlay/push/02-gitopscluster.yaml
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
```

This enables the Argo CD instance to deploy applications to any of those ACM Hub managed clusters.

As we can see from the previous example the placementRef.name is defined as `clusters-for-gitops`, and is specified as target clusters for the GitOps instance that is installed in argoNamespace: openshift-gitops.

On the other hand, the argoServer.cluster specification requires the local-cluster value, because we will be using the OpenShift GitOps deployed in the OpenShift cluster that is also where the ACM Hub is installed.

---

#### Model 2: Pull (ArgoCD Controller)

Each managed cluster runs a full OpenShift GitOps (ArgoCD) instance. The hub
ApplicationSet generates Applications that are propagated to spokes via
`ManifestWork`. Each spoke's ArgoCD controller reconciles locally.

**How it works**:
1. `GitOpsCluster` with `gitopsAddon.enabled: true` installs the OpenShift
   GitOps operator and ArgoCD instance on each selected managed cluster.
2. ArgoCD ApplicationSet on the hub generates Applications with pull-model
   annotations.
3. The **Propagation controller** wraps each Application in a `ManifestWork`
   and sends it to the corresponding managed cluster.
4. The local ArgoCD on the spoke reconciles the Application, pulling manifests
   from the Git repository.
5. Status is aggregated back via `Multicluster1pplicationSetReport`.

**GitOpsCluster** (with `gitopsAddon`):

```yaml
# applications/overlay/pull-argocontroller/02-gitopscluster.yaml
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
  name: gitops-clusters
  namespace: openshift-gitops
spec:
  argoServer:
    argoNamespace: openshift-gitops
  placementRef:
    kind: Placement
    apiVersion: cluster.open-cluster-management.io/v1beta1
    name: clusters-for-gitops
    namespace: openshift-gitops
  gitopsAddon:
    enabled: true
```

Apply:

```bash
oc apply -f appsmodel-pullpush/overlay/pull-argocontroller/.......
```

**ApplicationSet annotations** (required in the template):

```yaml
template:
  metadata:
    annotations:
      apps.open-cluster-management.io/ocm-managed-cluster: "{{name}}"
      argocd.argoproj.io/skip-reconcile: "true"
    labels:
      apps.open-cluster-management.io/pull-to-ocm-managed-cluster: "true"
  spec:
    destination:
      server: https://kubernetes.default.svc    # local deployment on spoke
```

> **Important**: `argocd.argoproj.io/skip-reconcile` prevents the hub ArgoCD
> from reconciling these Applications. Only the spoke's ArgoCD processes them.

---

#### Model 3: Pull (ArgoCD Agent)

Each managed cluster runs a lightweight ArgoCD agent that communicates with a
principal component on the hub via gRPC. This provides the full ArgoCD UI
experience (live status, health, resource tree) without requiring direct
network access from hub to spoke API servers.

**How it works**:
1. `GitOpsCluster` with `gitopsAddon.enabled: true` and
   `argoCDAgent.enabled: true` deploys the ArgoCD agent on each selected
   managed cluster and the agent principal on the hub.
2. The agent principal manages certificate rotation and secure gRPC channels.
3. Applications created on the hub are synchronized to spokes via gRPC (not
   `ManifestWork`).
4. Full application status (health, sync, resource tree) is synchronized back
   to the hub in real-time.

**GitOpsCluster** (with `argoCDAgent`):

```yaml
# applications/overlay/pull-agent/02-gitopscluster.yaml
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
  name: gitops-clusters
  namespace: openshift-gitops
spec:
  argoServer:
    argoNamespace: openshift-gitops
  placementRef:
    kind: Placement
    apiVersion: cluster.open-cluster-management.io/v1beta1
    name: clusters-for-gitops
    namespace: openshift-gitops
  gitopsAddon:
    enabled: true
    argoCDAgent:
      enabled: true
```

Apply:

```bash
oc apply -f appsmodel-pullpush/overlay/pull-agent/....
```

Verify agent deployment:

```bash
oc get gitopscluster gitops-clusters -n openshift-gitops \
  -o jsonpath='{.status.conditions}' | jq
```

Expected condition types: `ManifestWorksApplied`, `CertificatesReady`,
`ArgoCDAgentPrereqsReady`, `AddOnTemplateReady`, `ManagedclusteraddOnsReady`,
`AddOnDeploymentConfigsReady`, `ClustersRegistered`, `PlacementResolved`,
`Ready`.

**ApplicationSet**: Uses `spec.destination.name` (cluster name) instead of
`spec.destination.server` for routing. The agent principal maps Applications
to the correct spoke agent based on this field.

```yaml
template:
  spec:
    destination:
      name: "{{name}}"               # routed via agent principal
      namespace: my-app
```

> **Important**: The agent model requires a Load Balancer for the gRPC
> endpoint. The principal agent does not support high availability. See
> [ArgoCD Agent architecture](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.19/html-single/argo_cd_agent_architecture/index)
> for current limitations.

---

## Verification (all models)

After applying any overlay, verify that ArgoCD cluster secrets were created:

```bash
oc get secrets -n openshift-gitops -l argocd.argoproj.io/secret-type=cluster
```

Verify the `acm-placement` ConfigMap exists (used by `clusterDecisionResource`
generator):

```bash
oc get configmap acm-placement -n openshift-gitops -o yaml
```

---

## References


- https://rcarrata.com/openshift/argo-and-acm/
- [ACM 2.15 GitOps documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.15/html-single/gitops/index)
- [Introducing the Argo CD Application Pull Controller for RHACM](https://www.redhat.com/en/blog/introducing-the-argo-cd-application-pull-controller-for-red-hat-advanced-cluster-management)
- [ArgoCD Agent architecture (OpenShift GitOps 1.19)](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.19/html-single/argo_cd_agent_architecture/index)
- [OCM ArgoCD Pull Integration (GitHub)](https://github.com/open-cluster-management-io/argocd-pull-integration)
