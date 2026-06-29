# Initial ACM Setup

This document describes the required configuration to install do the initial ACM configuration:

---

## 1. Install in you laptop the PolicyGenerator plug-in

This is required to install the OpenShift-Gitops operator in the Hub cluster.

**Install PolicyGenerator locally to your laptop:**

```bash
mkdir -p ${HOME}/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator

wget -O ${HOME}/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator/PolicyGenerator \
  https://github.com/open-cluster-management-io/policy-generator-plugin/releases/download/v1.16.0/linux-amd64-PolicyGenerator

chmod +x ${HOME}/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator/PolicyGenerator
```

## 2. Configure the cluster topology sets and the policy namespace

```bash
oc apply -f bootstrap/clustergroups/
```

## 3. Install GitOps operator on the hub:

```bash
kustomize build bootstrap/gitops --enable-alpha-plugins | oc apply -f -
oc get policy gitops-operator -n acm-policies -w
```

References:
- [ACM 2.16 Governance - Policy Generator](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html-single/governance/index#policy-generator)
- [open-cluster-management-io/policy-generator-plugin (GitHub)](https://github.com/open-cluster-management-io/policy-generator-plugin)
- [ArgoCD - ApplicationSet Git Generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/)
