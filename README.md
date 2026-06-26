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

- **ManagedClusterSetBinding** — binds cluster sets to the `openshift-gitops` namespace.
- **Placement** — selects clusters for GitOpsCluster registration and app targeting.
- **GitOpsCluster** — registers managed clusters in ArgoCD as cluster secrets.

See [applications/README.md](applications/README.md) for step-by-step instructions.

## Governance

_(placeholder — governance bootstrap resources will be added here)_
