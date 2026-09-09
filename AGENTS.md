# AGENTS.md

## Repository purpose

Kubernetes GitOps homelab repo: plain YAML manifests and Flux `HelmRelease`/`HelmRepository` CRs grouped into Kustomize roots under `apps/` and `platforms/`. No application source code, Makefile, CI workflow, or scripts.

## Flux control flow

- `platforms/flux-system/gotk-sync.yaml` (generated, `DO NOT EDIT`): `GitRepository` points at `ssh://flux@soft-serve.tail36f6a3.ts.net:23231/homelab.git` branch `main`; Flux `Kustomization` `flux-system` applies path `./platforms` with `prune: true`.
- `platforms/kustomization.yaml` is the aggregator for that path: it lists every platform directory plus `app-kustomization.yaml`.
- `platforms/app-kustomization.yaml` is a second Flux `Kustomization` (`apps`, path `./apps`, `dependsOn: flux-system`). Apps ARE deployed by Flux — adding a directory under `apps/` plus an entry in `apps/kustomization.yaml` is sufficient.
- Both Flux Kustomizations set `decryption.provider: sops` with `secretRef: sops-age` — SOPS Secrets are decrypted cluster-side by Flux using the `sops-age` Secret in `flux-system`. Commit only `ENC[...]` ciphertext.

## Layout

- `platforms/flux-system/` — Flux bootstrap. `gotk-components.yaml` and `gotk-sync.yaml` are generated; regenerate with Flux tooling, don't hand-edit.
- `platforms/tailscale-operator/` — Tailscale operator, CRDs (in `crd/` sub-root), RBAC, and the `tailscale` `IngressClass` used by other components.
- `platforms/soft-serve/` — Soft Serve git server; still the Flux git remote (part of the GitOps loop) even though Gitea also exists.
- `platforms/gitea/` — Flux `HelmRelease` (chart `gitea`, pinned version) + custom Tailscale `Ingress`. Chart ingress is disabled; the repo defines its own `gitea-http` ClusterIP Service + Ingress. Sensitive chart values come from the SOPS-encrypted `gitea-values` Secret via `valuesFrom`.
- `platforms/kuberay-operator/`, `platforms/kubescape-operator/` — Helm-based components: each is just `namespace.yaml` + `helm-repository.yaml` + `helm-release.yaml` with a pinned chart version.
- `platforms/external-dns/` — RFC2136 provider against `192.168.1.10`, manages the `home.arpa` zone from Service/Ingress sources (`--domain-filter=home.arpa`, `--policy=sync`). TSIG keys come from the SOPS Secret `external-dns-secret`.
- `platforms/coredns/` — only a `coredns-custom` ConfigMap in `kube-system` forwarding `home.arpa` to `100.77.78.126`; it relies on the cluster CoreDNS importing `coredns-custom`, there is no CoreDNS deployment here.
- `apps/homepage/` — Homepage dashboard. Tailscale-only ingress. Ingresses of other components carry `gethomepage.dev/*` annotations for discovery (see `apps/headlamp/ingress.yaml`); keep them when adding ingresses.
- `apps/headlamp/` — Headlamp runs in `kube-system` (deliberately no `namespace.yaml`); Flux/kubescape plugins are installed via initContainers into an `emptyDir`.

Each immediate app/platform directory is a standalone Kustomize root with its own `kustomization.yaml`. New manifest files must be added to the directory's `resources` list or Kustomize will not render them.

## Commands

No build/test/lint tooling exists. Validate individual roots with:

```sh
kustomize build platforms          # whole platform tree
kustomize build apps               # all apps
kustomize build platforms/gitea    # single component
```

`kubectl kustomize <dir>` works equivalently. Helm-based components render fine this way because `HelmRelease`/`HelmRepository` are plain CRs — this does NOT validate the chart values against the actual chart.

## Secrets and SOPS

`.sops.yaml`: all `*.yaml`/`*.yml` match; only `data`/`stringData` fields are encrypted (`encrypted_regex: "^(data|stringData)"`); single age recipient.

Encrypted Secrets currently committed: `platforms/soft-serve/secret-admin-key.yaml`, `platforms/tailscale-operator/secret-operator-key.yaml`, `platforms/external-dns/secret-key.yaml`, `platforms/gitea/secret-values.yaml` (whole `values.yaml` chart values blob), `apps/homepage/secret-key.yaml`, `apps/headlamp/secret-operator-token.yaml`.

Never replace `ENC[...]` values with plaintext. Edit encrypted files through SOPS (`sops <file>`), not with a plain editor. Non-`data`/`stringData` fields stay readable — keep sensitive values out of them.

## Conventions

- Resource names match the directory/component name; selectors use `app.kubernetes.io/name: <name>` (Tailscale operator follows upstream `app: operator` instead).
- Components get their own namespace in the same root — except `headlamp` (in `kube-system`) and `coredns` (ConfigMap in `kube-system`).
- Images are pinned to explicit tags, except Tailscale operator (`stable`) and Headlamp (`latest`).
- Indentation is inconsistent across files (2-space vs 4-space in operator/SOPS files). Preserve local style when editing; don't reformat wholesale.

## Ingress and exposure

- LAN hosts use `*.home.arpa` (records created by external-dns); Tailscale hosts use the short name with `ingressClassName: tailscale` + `tls.hosts` (e.g. `homepage`, `headlamp`, `gitea`).
- Soft Serve instead uses `type: LoadBalancer` + `loadBalancerClass: tailscale` for SSH.
- The `tailscale` IngressClass comes from `platforms/tailscale-operator/` — it must stay applied for any Tailscale ingress/LB to work.

## Gotchas

- `apps/whoami` was removed; don't resurrect references to it.
- `gotk-sync.yaml` holds the Git source of truth — manual edits can break reconciliation or be overwritten by `flux bootstrap`.
- `platforms/flux-system/gotk-components.yaml` is large generated YAML; avoid broad search/replace there.
- Homepage needs its ServiceAccount + ClusterRole/ClusterRoleBinding (`apps/homepage/rbac.yaml`) for Kubernetes widgets.
- Soft Serve depends on its PVC and `soft-serve-admin-key` Secret; data path is `SOFT_SERVE_DATA_PATH=/soft-serve`.
- Removing a component means deleting its directory AND its entry in the parent `kustomization.yaml`; Flux `prune: true` will then delete it from the cluster.

## Adding a new component

1. Create a directory under `apps/` or `platforms/` with a local `kustomization.yaml` listing every file.
2. Add a `namespace.yaml` if it runs in its own namespace.
3. Register the directory in `apps/kustomization.yaml` or `platforms/kustomization.yaml` — this is what makes Flux deploy it.
4. Encrypt any Secret with SOPS before committing.
5. For Helm charts, follow the `kubescape-operator` pattern: `helm-repository.yaml` + `helm-release.yaml` with pinned chart version; put sensitive values in a SOPS Secret referenced via `valuesFrom`.
6. Validate with `kustomize build <dir>` before committing.
