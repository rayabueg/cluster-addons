# cluster-addons

Cluster-config repo for the k8s-lab platform. Holds **per-cluster subscriptions**
to the addon library at [`rayabueg/platform-addons`](https://github.com/rayabueg/platform-addons).
Argo CD syncs this repo continuously.

The user-facing apps half lives in [`cluster-applications`](https://github.com/rayabueg/cluster-applications).

> **Shape:** this repo follows the Gen2 `sdp-cluster-addons` model — a `base/`
> uprev surface that remote-refs a versioned addon library, plus one directory
> per cluster. The Gen1 wave model (`waves/wave1`, `waves/wave2`) and in-repo
> rendered manifests have been retired; manifests now live in `platform-addons`
> and are pinned by git tag.

## Layout

```
base/<addon>/kustomization.yaml          # remote ref to platform-addons//<addon>/manifests?ref=platform-vX.Y.Z
clusters/
  _template/                             # copyable per-cluster skeleton
  k8s-lab/
    applicationset.yaml                  # Argo CD ApplicationSet (one per cluster)
    kustomization.yaml                   # namesuffix: -k8s-lab; wraps the AppSet
    addons/<addon>/kustomization.yaml    # references ../../../../base/<addon> + per-cluster patches
bootstrap/argocd/root-app.yaml           # bootstrap: apply once to seed Argo CD
```

`base/<addon>` is the **single uprev surface** — bumping its `?ref=` to a newer
`platform-vX.Y.Z` tag rolls that addon forward on every cluster that subscribes.
Per-cluster directories hold cluster-specific patches.

## Addon → cluster mapping

Each cluster carries only the addons whose tag set intersects its own. The lab
cluster `k8s-lab` is tagged `application kubeadm`. Addon eligibility is declared
in `platform-addons/<addon>/metadata.yaml spec.clusters`; this repo decides where
an addon *actually* runs by creating a subscription under
`clusters/<cluster>/addons/<addon>/`.

## Versioning (no waves)

There are no wave folders. Versioning lives in the addon library's git tags:

| Gen1 wave concept | Now |
|---|---|
| `wave2` → `addons/*/base/latest` | `platform-addons` `main` (untagged tip) |
| `wave1` → `addons/*/base/stable` | a pinned `platform-vX.Y.Z` tag |
| `promote.sh` (latest → stable) | cutting a new tag in `platform-addons` |

### Uprev flow

1. In `platform-addons`, render + commit the change and cut `platform-vX.Y.Z`.
2. Here, bump every `base/<addon>/kustomization.yaml` `?ref=` to the new tag.
3. Argo CD reconciles. (At lab scale this is a single `main` branch; the
   branch-per-env merge-up of upstream Gen2 is not used.)

## Adding a cluster

1. `cp -r clusters/_template clusters/<cluster-name>`.
2. Substitute `__CLUSTER_NAME__` and `__BRANCH__` in `applicationset.yaml` and
   `kustomization.yaml`.
3. Prune `addons/` to the cluster's eligible addons (see mapping above).
4. Add per-cluster patches under `addons/<addon>/` as needed.

## Bootstrap

```bash
export KUBECONFIG="$HOME/.kube/lima-k8s-lab"
kubectl apply -f cluster-addons/bootstrap/argocd/root-app.yaml
kubectl apply -f cluster-applications/bootstrap/argocd/root-app.yaml
kubectl -n argocd get applications
```

## Validation

```bash
# every cluster builds (skip _template — it has placeholders)
for c in clusters/*/; do
  [ "$(basename "$c")" = "_template" ] && continue
  kustomize build "$c" >/dev/null && echo "ok: $c" || echo "FAIL: $c"
done

# every base ref resolves (requires platform-addons pushed + tagged on GitHub)
for b in base/*/; do
  kustomize build "$b" >/dev/null && echo "ok: $b" || echo "FAIL: $b"
done
```

## Contributing

Format: `scope: summary` — e.g. `base: uprev platform-v0.2.0`,
`k8s-lab: istio — add per-cluster patch`, `add cluster: <name>`.

If working from the parent `k8s-lab` repo, this folder is a **git submodule**:
commit + push here first, then commit the updated submodule pointer in the parent.
