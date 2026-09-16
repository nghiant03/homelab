# AGENTS.md

## Repository purpose

Kubernetes GitOps homelab repo: plain YAML manifests and Flux `HelmRelease`/`HelmRepository` CRs grouped into Kustomize roots under `apps/` and `platforms/`. No application source code, Makefile, CI workflow, or scripts.

## Flux control flow

- `platforms/flux-system/gotk-sync.yaml` (generated, `DO NOT EDIT`): `GitRepository` points at `ssh://git@gitea-ssh.gitea.svc.cluster.local:22/homelab/homelab.git` branch `main` — in-cluster DNS so the GitOps loop has no dependency on external DNS or Tailscale. Flux `Kustomization` `flux-system` applies path `./platforms` with `prune: true`.
- `platforms/kustomization.yaml` is the aggregator for that path: it lists every platform directory plus `app-kustomization.yaml`.
- `platforms/app-kustomization.yaml` is a second Flux `Kustomization` (`apps`, path `./apps`, `dependsOn: flux-system`). Apps ARE deployed by Flux — adding a directory under `apps/` plus an entry in `apps/kustomization.yaml` is sufficient.
- Both Flux Kustomizations set `decryption.provider: sops` with `secretRef: sops-age` — SOPS Secrets are decrypted cluster-side by Flux using the `sops-age` Secret in `flux-system`. Commit only `ENC[...]` ciphertext.

## Layout

- `platforms/flux-system/` — Flux bootstrap. `gotk-components.yaml` and `gotk-sync.yaml` are generated; regenerate with Flux tooling, don't hand-edit.
- `platforms/tailscale-operator/` — Tailscale operator, CRDs (in `crd/` sub-root), RBAC, and the `tailscale` `IngressClass` used by other components.
- `platforms/traefik/` — shared tailnet entry point (HTTPS apps plus a TCP-22 passthrough for Gitea SSH). The `traefik-tailnet` Service in `kube-system` selects the existing k3s Traefik pods and is exposed by the Tailscale operator as `traefik-tailnet`. A `HelmChartConfig` customizes the existing k3s chart to publish this Service's load-balancer status to app Ingresses and declares the `ssh` entrypoint. No separate Traefik installation or node-IP overrides. Verify rollout via the traefik deployment args (`--entryPoints.ssh.address=:22/tcp`), `traefik-tailnet` Service ports (22/80/443), and the `gitea-ssh` IngressRouteTCP; the tailnet ACL must allow clients TCP 22 to the proxy tag, not only 80/443.
- `platforms/soft-serve/` — Soft Serve git server; legacy — the Flux git remote moved to Gitea (`gitea.home.arpa`). Kept running for its existing data.
- `platforms/gitea/` — Flux `HelmRelease` (chart `gitea`, pinned version) + a Traefik Ingress (`gitea.home.arpa`, `ingressClassName: traefik`, TLS via the `homelab-ca` ClusterIssuer; chart ingress block in `helm-release.yaml` enables it). Laptop git works over HTTPS (`https://gitea.home.arpa`) and SSH (`git@gitea.home.arpa:22` — routed through the shared entry point by the Traefik `ssh` entrypoint and the `IngressRouteTCP` in `ingress-tcp.yaml`); the chart's built-in SSH listener serves Flux over the in-cluster service `gitea-ssh.gitea.svc.cluster.local:22`. Sensitive chart values come from the SOPS-encrypted `gitea-values` Secret via `valuesFrom`.
- `platforms/kuberay-operator/`, `platforms/kubescape-operator/` — Helm-based components: each is just `namespace.yaml` + `helm-repository.yaml` + `helm-release.yaml` with a pinned chart version.
- `platforms/external-dns/` — RFC2136 provider against Technitium at `100.114.255.114`, manages the `home.arpa` zone from Service/Ingress sources (`--domain-filter=home.arpa`, `--policy=sync`). TSIG keys come from the SOPS Secret `external-dns-secret`. The Technitium zone must allow AXFR zone transfers and dynamic updates for the `external-dns` TSIG key (security policy domain `*.home.arpa`, record types `ANY` — external-dns also writes TXT registry records).
- `platforms/cert-manager/` — cert-manager v1.21.x (HelmRelease) with CRDs enabled. Uses the OCI HelmRepository `oci://quay.io/jetstack/charts`. `bootstrap.yaml` contains a SelfSigned `ClusterIssuer`, a 10-year RSA-4096 root `Certificate` (`isCA: true`, `rotationPolicy: Never`) stored in Secret `homelab-root-ca` in the `cert-manager` namespace, and the production `ClusterIssuer` `homelab-ca` (type `ca`, points at the root secret). trust-manager (`platforms/trust-manager/`) reads that same secret into a `ca-certificates.crt` Bundle in every namespace.
- `platforms/trust-manager/` — Jetstack trust-manager HelmRelease (deploys into the `cert-manager` namespace; chart `trust-manager`). One Bundle (`homelab-trust`, API `trust.cert-manager.io/v1alpha1`) merges the root CA secret with the system default CAs and writes a `ca-certificates.crt` ConfigMap into every namespace. Use `namespaceSelector` to scope down later.
- `platforms/coredns/` — only a `coredns-custom` ConfigMap in `kube-system` forwarding `home.arpa` to `100.114.255.114`; it relies on the cluster CoreDNS importing `coredns-custom`, there is no CoreDNS deployment here. Add the new zone here when introducing a public domain later.
- `apps/homepage/` — Homepage dashboard. Traefik Ingress on `homepage.home.arpa` (cert via `homelab-ca`). Ingresses of other components carry `gethomepage.dev/*` annotations for discovery (see `apps/headlamp/ingress.yaml`); keep them when adding ingresses.
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

### Gitea break-glass credentials

- `platforms/gitea/secret-values.yaml` is the SOPS-encrypted `gitea-values` Secret. Its embedded `values.yaml` contains `gitea.admin.username`, `password`, and `passwordMode: keepUpdated`, loaded by the HelmRelease through `valuesFrom`. The configured password is reapplied when the admin configuration init container runs. It is not continuous password drift detection.
- Keep `gitea_admin` for emergency access and use a separate personal account for daily work. Changing the Secret's username can create another administrator; it does not rename or remove the old account. Account rotation does not require deleting PostgreSQL or repository PVCs.
- Store an emergency copy of the credentials in a password manager accessible without this cluster or Gitea. Back up the SOPS age private key and an encrypted copy of this repository outside the cluster as well; a cluster-only copy is not a break-glass recovery path.
- Rotate through SOPS, reconcile the Secret, and ensure the Gitea admin configuration init container runs again (a Secret-only update does not necessarily restart the pod). Verify login before updating the emergency copy. A successful Helm upgrade alone does not confirm that the new credentials are active.

## Conventions

### Gitea database credentials

- The same SOPS-encrypted `platforms/gitea/secret-values.yaml` stores `postgresql.global.postgresql.auth`: database/username `gitea`, an application `password`, and an independent `postgresPassword` for the maintenance-only `postgres` superuser. Do not use the superuser as Gitea's application login or reuse the web-admin password.
- Gitea chart 12.5.3 derives its database `NAME`, `USER`, and `PASSWD` from these values. The PostgreSQL subchart also uses them for its `gitea-postgresql` Secret; do not introduce a separate `gitea.config.database.PASSWD` source of truth.
- Updating Helm values or a Secret does not by itself rotate passwords in an initialized PostgreSQL database. Change the existing roles, verify TCP authentication, synchronize Secrets, and then resume reconciliation. Keep all PVCs and database ownership intact.

- Resource names match the directory/component name; selectors use `app.kubernetes.io/name: <name>` (Tailscale operator follows upstream `app: operator` instead).
- Components get their own namespace in the same root — except `headlamp`, `coredns`, and the shared `traefik` configuration (all in `kube-system`).
- Images are pinned to explicit tags, except Tailscale operator (`stable`) and Headlamp (`latest`).
- Indentation is inconsistent across files (2-space vs 4-space in operator/SOPS files). Preserve local style when editing; don't reformat wholesale.

## Ingress and exposure

Tailnet is treated as LAN — every client uses Technitium (100.114.255.114, on the tailnet) for DNS and reaches the cluster by IP. So there is no separate "LAN vs tailnet" tier; everything that is reachable from tailnet devices is reachable as far as this repo is concerned. **No `*.ts.net` URLs in user-facing paths.**

- All app Ingresses use `ingressClassName: traefik`, with hostnames like `app.home.arpa`. Tailnet clients reach the shared Tailscale-managed `kube-system/traefik-tailnet` Service, which forwards TCP 80/443 to Traefik. The k3s chart's `providers.kubernetesIngress.publishedService.pathOverride` points to that Service, so its operator-assigned address propagates to every Ingress and external-dns (RFC2136 → Technitium). Do not add per-app node-IP target annotations or per-app Tailscale proxies for HTTP services.
- external-dns v0.22.0 collects both IP and hostname fields from Ingress status; its record-type conflict resolver prefers A/AAAA over a competing CNAME and can delete an owned obsolete CNAME under `--policy=sync`. Verify actual DNS records during cutover; the shared entry point must publish an IP, and an old Service must not keep claiming the app hostname. No MagicDNS lookup is needed for IP records.
- The shared entry point is node-independent, but its standalone Tailscale proxy and the current Traefik deployment are not highly available. Multi-node HA requires supported ingress ProxyGroups plus spread Traefik replicas; app availability also depends on storage (Gitea currently uses node-local `local-path` PVCs).
- TLS for `*.home.arpa` is signed by the internal root CA via the `homelab-ca` ClusterIssuer: add the annotation `cert-manager.io/cluster-issuer: homelab-ca` and a `spec.tls[].secretName` matching the leaf (cert-manager populates it). Traefik hot-reloads the Secret on renewal so no pod restart is needed.
- Raw-TCP workloads that can't be done via HTTP Ingress (e.g. Soft‑Serve SSH on `soft-serve.home.arpa:23231`) use `type: LoadBalancer` + `loadBalancerClass: tailscale` with `tailscale.com/hostname` + `external-dns.kubernetes.io/hostname` annotations. The Tailscale operator's support for HTTP Ingress is **only** for `*.ts.net` names — keep that out of this model. Gitea SSH (port 22) is the exception: it rides the shared `traefik-tailnet` entry point via the Traefik `ssh` entrypoint + `IngressRouteTCP` (`platforms/gitea/ingress-tcp.yaml`), so `gitea.home.arpa` serves both 443 and 22.
- Note the prefix: external-dns v0.22+ uses `external-dns.kubernetes.io/`; the legacy `external-dns.alpha.kubernetes.io/` annotations are ignored.
- Cleanup (Part A → end): once nothing references ts.net names, disable MagicDNS in the Tailscale admin console (DNS page). Global nameservers (`100.114.255.114`) and "Override DNS servers" stay on — they are independent of MagicDNS and keep Technitium resolution working. Then delete Technitium's conditional forwarder zone `tail36f6a3.ts.net` → `100.100.100.100`; it's only needed for resolving external-dns CNAMEs to Tailscale proxy `ts.net` names. `*.home.arpa` is unaffected.

## CA trust (internal root)

The internal root CA (`homelab-root-ca`) is generated by cert-manager in-cluster. Devices (Linux/macOS/Windows) must import it once to trust `*.home.arpa`:

```sh
kubectl -n cert-manager get secret homelab-root-ca \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > homelab-root-ca.crt
```

- Linux (Debian/Ubuntu): `sudo cp homelab-root-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates`. Firefox on Linux ignores the OS store by default — set `security.enterprise_roots.enabled=true` in `about:config` or import with `certutil` into NSS.
- macOS: `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain homelab-root-ca.crt` (covers Safari/Chrome/Edge/curl).
- Windows (admin PowerShell): `Import-Certificate -FilePath homelab-root-ca.crt -CertStoreLocation Cert:\LocalMachine\Root`.

The root key lives only in the cluster Secret `cert-manager/homelab-root-ca`. **Back it up** (e.g. SOPS-encrypt an export and git it under `platforms/cert-manager/`); losing the cluster loses the trust anchor and forces re-import on every device.

## Gotchas

- `apps/whoami` was removed; don't resurrect references to it.
- `gotk-sync.yaml` holds the Git source of truth — manual edits can break reconciliation or be overwritten by `flux bootstrap`.
- `platforms/flux-system/gotk-components.yaml` is large generated YAML; avoid broad search/replace there.
- Homepage needs its ServiceAccount + ClusterRole/ClusterRoleBinding (`apps/homepage/rbac.yaml`) for Kubernetes widgets.
- Soft Serve depends on its PVC and `soft-serve-admin-key` Secret; data path is `SOFT_SERVE_DATA_PATH=/soft-serve`.
- Removing a component means deleting its directory AND its entry in the parent `kustomization.yaml`; Flux `prune: true` will then delete it from the cluster.
- cert-manager CRDs install via `crds.enabled: true` on the HelmRelease. The first Flux apply of `platforms/cert-manager/bootstrap.yaml` races CRD installation and will transiently error on the `Certificate`/`ClusterIssuer` objects — Flux retries and self-heals. Cert readiness for Traefik/ingress shims: leaf `Certificate` default is 90 days; the root is pinned to 10 years with `rotationPolicy: Never` so device trust survives.
- Flux→Gitea SSH uses the chart's in-cluster `gitea-ssh.gitea.svc.cluster.local:22`. The `flux-system` secret's `known_hosts` must contain an entry for that host pointing at Gitea's generated SSH host key: `kubectl get secret -n gitea gitea-ssh-host-keys -o jsonpath='{.data}` (or `ssh-keyscan -p 22 gitea-ssh.gitea.svc.cluster.local` from a debug pod), then `kubectl edit secret flux-system -n flux-system` to add the line.

## Adding a new component

1. Create a directory under `apps/` or `platforms/` with a local `kustomization.yaml` listing every file.
2. Add a `namespace.yaml` if it runs in its own namespace.
3. Register the directory in `apps/kustomization.yaml` or `platforms/kustomization.yaml` — this is what makes Flux deploy it.
4. Encrypt any Secret with SOPS before committing.
5. For Helm charts, follow the `kubescape-operator` pattern: `helm-repository.yaml` + `helm-release.yaml` with pinned chart version; put sensitive values in a SOPS Secret referenced via `valuesFrom`.
6. Validate with `kustomize build <dir>` before committing.
