# ACM bootstrap

Procedure to install Red Hat Advanced Cluster Management (ACM) on an OpenShift cluster and configure OpenShift GitOps on the hub.

## Prerequisites

Before installing, check that:

- Your worker nodes have enough capacity for ACM. See the [ACM install prerequisites](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html-single/install/index#requirements-and-recommendations) for current sizing.
- You have cluster-admin access.
- The `oc` CLI is authenticated against the cluster.

If you have limited worker resources, change `availabilityConfig` to `Basic` in `bootstrap/acm/03-multiclusterhub.yaml`. This runs one replica per component instead of two.

On ROSA HCP the `cluster-admins` Group is managed by ROSA and must not be overwritten. The manifest `clustergroups/11-argocd-admin-group.yaml` creates a separate `argocd-admins` Group for this reason. If you accidentally overwrite `cluster-admins`, re-authenticate with `rosa create admin` to regain access.

## 1. Install the PolicyGenerator plugin

PolicyGenerator is a Kustomize plugin used by the ACM governance policy framework. Step 5 uses it to install the OpenShift GitOps operator on the hub via a governance policy.

```bash
mkdir -p ${HOME}/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator

wget -O ${HOME}/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator/PolicyGenerator \
  https://github.com/open-cluster-management-io/policy-generator-plugin/releases/download/v1.16.0/linux-amd64-PolicyGenerator

chmod +x ${HOME}/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator/PolicyGenerator
```

## 2. Clone the repository

```bash
git clone https://github.com/CSA-RH/acm_bootstrap.git
cd acm_bootstrap
```

## 3. Install the ACM operator and MultiClusterHub

Apply the namespace, OperatorGroup, and Subscription:

```bash
oc apply -f bootstrap/acm/00-namespace.yaml
oc apply -f bootstrap/acm/01-operatorgroup.yaml
oc apply -f bootstrap/acm/02-acm-subscription.yaml
```

Wait for the operator CSV to reach `Succeeded`:

```bash
oc get csv -n open-cluster-management -w
```

Then create the MultiClusterHub:

```bash
oc apply -f bootstrap/acm/03-multiclusterhub.yaml
```

The MultiClusterHub takes several minutes to finish. Wait until its status reads `Running`:

```bash
oc get multiclusterhub multiclusterhub -n open-cluster-management -w
```

## 4. Deploy the cluster topology and policy namespace

This creates the policy namespace, ManagedClusterSets, ManagedClusters, their bindings, and the ArgoCD admin group. The GitOps Placement in step 5 requires ManagedClusterSetBindings in the policy namespace, so this step must run first.

These manifests define a sample topology for a blank cluster. If your environment already has ManagedClusterSets, ManagedClusters, or RBAC configured, use your existing setup. Applying them over an existing configuration may overwrite it.

One manifest (`09-rbac-appset-placementdecisions.yaml`) creates a Role and RoleBinding in the `openshift-gitops` namespace, which does not exist yet. It will fail on first apply; that is expected and fixed in step 5.

```bash
oc apply -f bootstrap/clustergroups/
```

## 5. Install the GitOps operator on the hub

```bash
kustomize build bootstrap/gitops --enable-alpha-plugins | oc apply -f -
oc get policy gitops-operator -n acm-policies -w
```

Wait until the policy is `Compliant` and the `openshift-gitops` namespace exists:

```bash
oc get ns openshift-gitops
```

Then re-apply the clustergroups to pick up the RBAC manifest that failed in step 4:

```bash
oc apply -f bootstrap/clustergroups/
```

## 6. Deploy operator policies via ArgoCD

This ApplicationSet uses a git directory generator that points to the [acm-policyframework-demo-ms](https://github.com/CSA-RH/acm-policyframework-demo-ms) repository. ArgoCD auto-discovers `operators/*/base` and `operators/*/overlays/*` directories and creates one Application per path. The operators deployed are: ACS, cert-manager, compliance-operator, and web-terminal.

The upstream repo includes `cert-manager0-default` which conflicts with `cert-manager1-base-with-overlays` (same policy and placement names). The ApplicationSet excludes `cert-manager0-default` to avoid the conflict.

```bash
oc apply -f argocd/operators-appset.yaml
```

## Known limitations

On ROSA HCP:

- The compliance operator `ocp4-cis` platform profile does not exist (the control plane is managed by Red Hat). The ScanSettingBinding will stay PENDING and the `compliance-cis-results` policy will remain NonCompliant.
- ACS CIS Kubernetes benchmark checks in the Configuration Management view will show "could not be assessed" because the ACS collector runs without `hostPID` and cannot inspect kubelet process arguments. Use the Compliance Operator results for CIS reporting instead.

On all platforms:

- Placeholder ManagedClusters (`cluster1`, `cluster2`) from step 4 are not real clusters. Policies targeting them will show `<none>` or NonCompliant until actual clusters join.

## References

- [ACM 2.16 install prerequisites](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html-single/install/index#requirements-and-recommendations)
- [ACM 2.16 Governance, Policy Generator](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html-single/governance/index#policy-generator)
- [policy-generator-plugin (GitHub)](https://github.com/open-cluster-management-io/policy-generator-plugin)
- [ArgoCD ApplicationSet Git generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/)
