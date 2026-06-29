# ACM Bootstrap

This repository contains the bootstrap configuration for ACM. It is organized into two sections:

## 1. [Initial ACM Setup](bootstrap/README.md)

Prerequisites and initial hub cluster configuration: PolicyGenerator plugin
installation, cluster topology (ManagedClusterSets, ManagedClusters,
ManagedClusterSetBindings), RBAC, and OpenShift GitOps operator deployment
via governance policies.

## 2. [Application Deployment Models (Push / Pull)](appsmodel-pullpush/README.md)

Configuring managed clusters for OpenShift GitOps and deploying applications
via ArgoCD ApplicationSets. Covers the three supported deployment models:

- **Push**: Hub ArgoCD deploys directly to managed clusters.
- **Pull (ArgoCD Controller)**: Full ArgoCD instance on each spoke, applications propagated via `ManifestWork`.
- **Pull (ArgoCD Agent)**: Lightweight agent per spoke, gRPC-based sync with real-time status feedback.
