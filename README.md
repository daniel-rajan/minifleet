# minifleet

A small GitOps practice repo that uses [Flux](https://fluxcd.io/) to deploy Helm charts to Kubernetes, with Kustomize handling per-environment configuration.

It covers two common ways to deploy a Helm chart with Flux:

| App | Chart source | How it works |
|-----|--------------|--------------|
| **podinfo** | Remote Helm repository (`HelmRepository`) | Pulls `podinfo` chart `6.7.0` from `stefanprodan.github.io/podinfo` |
| **webapp** | Chart stored in this Git repo (`GitRepository`) | Builds the local chart at `thirdparty/webapp/chart` (nginx) |

## Repository layout

```
.
├── clusters/
│   └── dev/                         # Flux config for the "dev" cluster
│       ├── flux-system/
│       │   └── git-source.yaml      # GitRepository "mini-fleet" -> this repo, branch main
│       └── thirdparty/
│           ├── podinfo.yaml         # Flux Kustomization -> thirdparty/podinfo/overlays/dev
│           └── webapp.yaml          # Flux Kustomization -> thirdparty/webapp/overlays/dev (dependsOn podinfo)
└── thirdparty/
    ├── podinfo/
    │   ├── base/                    # Namespace, HelmRepository, HelmRelease
    │   └── overlays/dev/            # Patches HelmRelease values (replicaCount: 1)
    └── webapp/
        ├── base/                    # Namespace, HelmRelease (chart from GitRepository)
        ├── chart/                   # Local Helm chart (helm create scaffold, nginx image)
        └── overlays/dev/            # Dev overlay (currently just the base)
```

## How it fits together

```
GitRepository "mini-fleet" (flux-system)
        │
        ├── Kustomization "podinfo" ──> thirdparty/podinfo/overlays/dev
        │        └── HelmRepository + HelmRelease "podinfo" (namespace: podinfo)
        │
        └── Kustomization "webapp"  ──> thirdparty/webapp/overlays/dev   (waits for podinfo)
                 └── HelmRelease "webapp" (namespace: webapp)
                          └── chart: ./thirdparty/webapp/chart from GitRepository "mini-fleet"
```

1. Flux polls this repo every minute through the `mini-fleet` `GitRepository`.
2. Each Flux `Kustomization` in `clusters/dev/thirdparty/` renders a Kustomize overlay and applies it, reconciling every 10 minutes with `prune: true`.
3. The overlays produce `HelmRelease` objects, and the Flux helm-controller installs or upgrades the charts.

### Base and overlays

Each app has a `base/` with environment-neutral resources and an `overlays/<env>/` that references the base and adds patches. For example, the podinfo dev overlay sets Helm values through a strategic-merge patch on the `HelmRelease`:

```yaml
# thirdparty/podinfo/overlays/dev/values-patch.yaml
spec:
  values:
    replicaCount: 1
```

To add another environment such as `prod`, create `thirdparty/<app>/overlays/prod/` and a matching `clusters/prod/` directory.

## Prerequisites

- A Kubernetes cluster (kind, minikube, k3d, etc.)
- [`flux` CLI](https://fluxcd.io/flux/installation/)
- `kubectl`, and optionally `kustomize` and `helm` for local testing

## Getting started

Install the Flux controllers into the cluster:

```bash
flux check --pre
flux install
```

Apply the cluster config:

```bash
kubectl apply -f clusters/dev/flux-system/git-source.yaml
kubectl apply -f clusters/dev/thirdparty/
```

Alternatively, `flux bootstrap github --owner=daniel-rajan --repository=minifleet --path=clusters/dev --personal` installs Flux and has it manage `clusters/dev` from Git.

Watch it reconcile:

```bash
flux get sources git
flux get kustomizations
flux get helmreleases -A
kubectl get pods -n podinfo
kubectl get pods -n webapp
```

Force an immediate sync after pushing a change:

```bash
flux reconcile kustomization podinfo --with-source
```

## Testing locally

Render the overlays without touching a cluster:

```bash
kustomize build thirdparty/podinfo/overlays/dev
kustomize build thirdparty/webapp/overlays/dev
```

Lint and render the local chart:

```bash
helm lint thirdparty/webapp/chart
helm template webapp thirdparty/webapp/chart
```

## Accessing the apps

```bash
kubectl -n podinfo port-forward svc/podinfo 9898:9898   # http://localhost:9898
kubectl -n webapp  port-forward svc/webapp  8080:80     # http://localhost:8080
```
