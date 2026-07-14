# ACM Bootstrap

This repository contains the bootstrap configuration for ACM. It is organized into two sections:

> **Disclaimer:** This repository is for learning and testing purposes only.
> All configurations should be thoroughly tested in a non-production
> environment before applying to any production cluster.

## 1. [Initial ACM Setup](bootstrap/README.md)

Prerequisites and initial hub cluster configuration: PolicyGenerator plugin
installation, cluster topology (ManagedClusterSets, ManagedClusters,
ManagedClusterSetBindings), RBAC, and OpenShift GitOps operator deployment
via governance policies.

The **PolicyGenerator** is a Kustomize plugin that converts simple YAML
definitions into ACM `Policy`, `PlacementBinding`, and `PlacementRule`
resources. It allows defining compliance and configuration policies in a
concise format and generating the full ACM governance objects needed to
enforce them across managed clusters.

## 2. [Application Deployment Models (Push / Pull)](appsmodel-pullpush/README.md)

Configuring managed clusters for OpenShift GitOps and deploying applications
via ArgoCD ApplicationSets. Covers the three supported deployment models:

- **Push**: Hub ArgoCD deploys directly to managed clusters.
- **Pull (ArgoCD Controller)**: Full ArgoCD instance on each spoke, applications propagated via `ManifestWork`.
- **Pull (ArgoCD Agent)**: Lightweight agent per spoke, gRPC-based sync with real-time status feedback.


## 3. If using using a SNO, increase allowed pods to 500.

NOTE: This was done for test porposes only, dont use this configurations in Production.

I used a SingleNodeOpenShift sandbox cluster for the testings, therefore it was required to increase the number of allowed pods to 500, to allow more than 250 to be schedualed into the cluster.

1.0.1.Get the labels of the mc
oc get machineconfigpool worker -o yaml | grep 'labels:' -A4

1.0.2.Create a KubeletConfig, adding the labels from 1.1
Note: If SNO, should be the "master".

```bash
cat << EOF > kubeletconfig.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: set-max-pods 
spec:
  machineConfigPoolSelector:
    matchLabels:
      pools.operator.machineconfiguration.openshift.io/master: "" 
  kubeletConfig:
    podsPerCore: 10 
    maxPods: 500
EOF

oc apply -f kubeletconfig.yaml
```

Applying this configuration will automatically execute a restart of the node, to apply the configuration.

Check that the node is configured with 500 pods
```bash
oc get node <node-name> -o jsonpath='{.status.allocatable.pods}'
```
