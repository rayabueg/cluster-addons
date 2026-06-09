# Cluster directory template

Copy this directory to `clusters/<cluster-name>/` when onboarding a new cluster
to this Argo CD instance.

## Steps

1. Copy: `cp -r clusters/_template clusters/<cluster-name>`.
2. In `applicationset.yaml` and `kustomization.yaml`, replace every
   `__CLUSTER_NAME__` with the real cluster name and every `__BRANCH__` with the
   branch Argo should track (the lab uses `main`).
3. Prune `addons/` to the addons this cluster should run. Each addon's
   eligibility is declared in `platform-addons/<addon>/metadata.yaml spec.clusters`.
   The lab cluster `k8s-lab` carries tags `application kubeadm`.
4. Add per-cluster patches under `addons/<addon>/` as needed.

## Adding an addon to a cluster

Create `addons/<addon>/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../../../base/<addon>

# patches: []   # cluster-specific patches go here
```

`../../../../base/<addon>` resolves to the tag that `base/<addon>` is pinned to
in the addon library ([`platform-addons`](https://github.com/rayabueg/platform-addons)),
so this directory never names an addon version.

## ApplicationSet notes (lab-specific — keep these)

- **Destination is in-cluster** (`server: https://kubernetes.default.svc`) — Argo
  manages its own cluster.
- **Client-side apply** (ServerSideApply omitted) — required so `ignoreDifferences`
  works for runtime-injected fields (CRD `selectableFields`, webhook `caBundle` /
  `failurePolicy`).
- `CreateNamespace=true` — the lab has no separate namespace-provisioning path.
