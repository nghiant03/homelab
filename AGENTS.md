# AGENTS.md

## Repository purpose

Kubernetes GitOps homelab repo (k3s): plain YAML manifests and Flux `HelmRelease`/`HelmRepository`/`OCIRepository` CRs grouped into Kustomize roots under `apps/` and `platforms/`. No application source code, Makefile, CI workflow, or scripts.

## Flux control flow

- `platforms/flux-system/gotk-sync.yaml` (generated, `DO NOT EDIT`): `GitRepository` points at `ssh://git@gitea-ssh.gitea.svc.cluster.local:22/nghiant03/homelab.git` branch `main` (in-cluster DNS, so the GitOps loop has no dependency on external DNS or Tailscale; interval 1m). Flux `Kustomization` `flux-system` applies path `./platforms` with `prune: true` (interval 10m).
- `platforms/kustomization.yaml` is the aggregator for that path: it lists every platform directory plus `app-kustomization.yaml`.
- `platforms/app-kustomization.yaml` is a second Flux `Kustomization` (`apps`, path `./apps`, `dependsOn: flux-system`). Apps ARE deployed by Flux; adding a directory under `apps/` plus an entry in `apps/kustomization.yaml` is sufficient.
- Both Flux Kustomizations set `decryption.provider: sops` with `secretRef: sops-age`; SOPS Secrets are decrypted cluster-side by Flux using the `sops-age` Secret in `flux-system`. Commit only `ENC[...]` ciphertext.
- HelmReleases are NOT continuously reconciled against live cluster state: the helm-controller compares a digest of chart version + values against its status and skips when "in-sync". A Helm-managed resource deleted out-of-band (e.g. `kubectl delete cronjob`) stays deleted until the release changes. To force recreation, change any value and let Flux upgrade; the `reconcile.fluxcd.io/requestedAt` annotation alone does nothing on an in-sync release. Plain YAML under the kustomize-controller does self-heal.

## Layout

Each immediate app/platform directory is a standalone Kustomize root with its own `kustomization.yaml`. New manifest files must be added to the directory's `resources` list or Kustomize will not render them.

### platforms/

- `flux-system/` — Flux bootstrap. `gotk-components.yaml` and `gotk-sync.yaml` are generated; regenerate with Flux tooling, don't hand-edit.
- `tailscale-operator/` — Tailscale k8s operator (plain manifests, image `tailscale/k8s-operator:stable`, proxy image pinned via `PROXY_IMAGE: tailscale/tailscale:v1.102.4`). Operator OAuth client in SOPS Secret `operator-oauth` (keys `client-id`, `client-secret`). CRDs live in the `crd/` sub-root (Connector, DNSConfig, ProxyClass, ProxyGroup, ProxyGroupPolicy, PeerRelay, Recorder, Tailnet). Declares the `tailscale` `IngressClass`. Proxy pods get tag `tag:k3s`, operator `tag:k3s-operator`.
- `traefik/` — shared tailnet entry point configuration (no Traefik install; k3s ships one). `service.yaml`: `tailnet-traefik` LoadBalancer Service in `kube-system` (`loadBalancerClass: tailscale`, hostname `tailnet-traefik`) selecting the existing k3s Traefik pods, ports 80/443/22. `helm-chart-config.yaml`: k3s `HelmChartConfig` publishing that Service's address to app Ingresses (`publishedService.pathOverride: kube-system/tailnet-traefik`) and declaring the internal `ssh` entrypoint (port 22, `expose: false`). Verify rollout via the traefik deployment args (`--entryPoints.ssh.address=:22/tcp`), the Service ports, and the `gitea-ssh` IngressRouteTCP; the tailnet ACL must allow clients TCP 22 to the proxy tag, not only 80/443.
- `gitea/` — Flux `HelmRelease` (chart `gitea` from `https://dl.gitea.com/charts/`, version `12.7.0`). HTTPS at `gitea.home.arpa` via chart-rendered Ingress (`ingressClassName: traefik`, TLS `homelab-ca`); SSH for laptops rides the shared entry point (`ssh` entrypoint + `IngressRouteTCP` in `ingress-tcp.yaml` → chart's `gitea-ssh` Service, port 22). Flux uses the in-cluster `gitea-ssh.gitea.svc.cluster.local:22`. Server config in values: `ROOT_URL: https://gitea.home.arpa/`, `PUBLIC_URL_DETECTION: never`, `SSH_LISTEN_PORT: 2222`, registration disabled, `DEFAULT_BRANCH: main`. Sensitive values (admin credentials, DB auth) come from the SOPS Secret `gitea-values` via `valuesFrom`. The Gitea container mounts the trust-manager `homelab-trust` ConfigMap over `/etc/ssl/certs/ca-certificates.crt` so webhook deliveries to `https://*.home.arpa` verify against the internal root CA. Storage: `local-path` PVCs (gitea 20Gi, postgresql 10Gi).
- `kuberay-operator/` — Helm-based: `namespace.yaml` + `helm-repository.yaml` (`https://ray-project.github.io/kuberay-helm/`) + `helm-release.yaml` (chart `kuberay-operator` `1.6.1`).
- `kyverno/` — Kyverno policy engine (chart `3.9.1` from `https://kyverno.github.io/kyverno/`, admission controller 1 replica). One MutatingPolicy `inject-ca` (`mutating-policy.yaml`, CEL-based `policies.kyverno.io/v1` API) mutates every Pod CREATEd in the `woodpecker-ci` and `renovate` namespaces to mount the trust-manager `homelab-trust` ConfigMap (read-only subPath) at `/etc/ssl/certs/ca-certificates.crt` and `/etc/docker/certs.d/gitea.home.arpa/ca.crt`. The mutation is an `ApplyConfiguration` CEL expression that adds only the CA entries (copying existing volumes would include atomic projected-volume fields ApplyConfiguration cannot mutate). Extending coverage to a new runner namespace = add it to the `matchConditions` expression. Kyverno is critical infrastructure: its admission webhooks intercept pod creation in most namespaces (kube-system/kyverno excluded), so a down/wedged Kyverno blocks scheduling cluster-wide.
- `renovate/` — Renovate self-hosted (chart from `https://docs.renovatebot.com/helm-charts`, version pinned and expected to go stale fast; Renovate PRs its own chart bumps). Runs as a daily 01:00 CronJob. Config inline in `helm-release.yaml` (`renovate.config`: platform gitea, endpoint `https://gitea.home.arpa/api/v1`, autodiscover `nghiant03/*`); the chart renders it into a `renovate-config` ConfigMap mounted as `/usr/src/app/config.json` via `RENOVATE_CONFIG_FILE`. The Gitea PAT lives in the SOPS Secret `renovate-token` (key `RENOVATE_TOKEN`), injected via the chart's top-level `existingSecret` (envFrom). `LOG_LEVEL: debug` is set via `env`. TLS to Gitea is trusted via the kyverno CA injection. Debug a run with `kubectl -n renovate create job --from=cronjob/renovate <name>`.
- `external-dns/` — plain Deployment, image `registry.k8s.io/external-dns/external-dns:v0.22.0`, RFC2136 provider against Technitium at `100.90.10.86`, manages the `home.arpa` zone from Service/Ingress sources (`--domain-filter=home.arpa`, `--policy=sync`, TXT registry prefix `external-dns-`, owner id `homelab`, `--rfc2136-axfr`). TSIG key name + secret come from the SOPS Secret `external-dns-secret` (keys `tsig-keyname`, `tsig-secret`). The Technitium zone must allow AXFR and dynamic updates for that TSIG key (record types ANY; external-dns also writes TXT records).
- `cert-manager/` — cert-manager chart `1.21.2` via OCI HelmRepository `oci://quay.io/jetstack/charts`, CRDs via `crds.enabled: true`. `bootstrap.yaml`: SelfSigned bootstrap `ClusterIssuer` → 10-year RSA-4096 root `Certificate` (`isCA: true`, `rotationPolicy: Never`) stored in Secret `homelab-root-ca` → production `ClusterIssuer` `homelab-ca` (type `ca`).
- `trust-manager/` — trust-manager chart `0.18.0` (same OCI repo, deploys into the `cert-manager` namespace). One Bundle `homelab-trust` (`trust.cert-manager.io/v1alpha1`) merges Secret `homelab-root-ca` (`tls.crt`) with `useDefaultCAs` and writes a `ca-certificates.crt` ConfigMap named `homelab-trust` into every namespace (`namespaceSelector: {}` — scope down later if needed).
- `coredns/` — only a `coredns-custom` ConfigMap in `kube-system` forwarding `home.arpa` to `100.90.10.86` with `force_tcp` (the forward crosses the tailnet and UDP replies were being lost). Relies on the k3s CoreDNS importing `coredns-custom`; there is no CoreDNS deployment here. `prefer_tcp` is NOT a valid `forward` option and crashloops CoreDNS; use `force_tcp`.
- `cloudflared/` — Cloudflare Tunnel connector: plain Deployment (2 replicas, `cloudflare/cloudflared:2026.9.1`, token from SOPS Secret `cloudflared-token` key `token`, metrics/health on :2000, locked-down securityContext, `automountServiceAccountToken: false`). Tunnel routes are managed in the Cloudflare dashboard, not here.
- `woodpecker-ci/` — Woodpecker CI server + agent, HelmRelease from OCIRepository `oci://ghcr.io/woodpecker-ci/helm/woodpecker` tag `3.7.5`. Server: Gitea OAuth (client id/secret from SOPS Secret `gitea-oauth`), `WOODPECKER_GITEA_URL` uses the in-cluster `http://gitea-http.gitea.svc.cluster.local:3000` while browser OAuth redirects use `https://gitea.home.arpa` (`WOODPECKER_EXPERT_FORGE_OAUTH_HOST`); UI at `woodpecker-ci.home.arpa` (Traefik Ingress, `homelab-ca`). Agent uses the kubernetes backend, step pods in the `woodpecker-ci` namespace. `pvc.yaml` adds a 10Gi `build-cache` PVC. Step pods get the internal CA via the kyverno `inject-ca` policy.
- `headlamp/` — Headlamp dashboard, plain manifests deploying into `kube-system` (deliberately no `namespace.yaml`). Image `ghcr.io/headlamp-k8s/headlamp:latest`, in-cluster mode; the Flux plugin (`headlamp-plugin-flux:v0.6.0`) is installed via initContainer into an `emptyDir`. Ingress `headlamp.home.arpa`. `secret-operator-token.yaml` is a `kubernetes.io/service-account-token` Secret for the `headlamp-operator` ServiceAccount (login token). RBAC in `rbac.yaml`.

### apps/

- `homepage/` — Homepage dashboard (`ghcr.io/gethomepage/homepage:v2.4.0`), Ingress `homepage.home.arpa`. Central catalog: web UIs are discovered from `gethomepage.dev/*` ingress annotations (keep adding them to new ingresses; discovered group names must match the layout groups: `Apps`, `Platform`, `Kubernetes`, `Network`). Manual entries, group layout (tabs `Overview` and `Cluster`), and the flat dark + amber theme live in `config-map.yaml`; the background is set by overriding homepage's `--bg-color` variable because `#__next` paints over `body`/`html` rules. Discovered ingresses carry `gethomepage.dev/showStats: "true"`; platform components without a UI are listed manually in the `Platform` group with `namespace`/`podSelector`/`showStats`. The Tailscale widget key comes from SOPS Secret `homepage-secrets` (key `HOMEPAGE_VAR_TAILSCALE_KEY`). Homepage needs its ServiceAccount + RBAC (`rbac.yaml`) for Kubernetes widgets.
- `syncthing/` — Syncthing always-on node (`syncthing/syncthing:2.1.5`), sync hub for offline password databases (KeePassXC `.kdbx`) on tailnet devices. GUI at `syncthing.home.arpa` (Traefik + `homelab-ca`); sync protocol (TCP/QUIC 22000) exposed through a separate `syncthing-tailnet` LoadBalancer Service (`loadBalancerClass: tailscale`) published as `syncthing-sync.home.arpa`. The two hostnames must stay distinct (different tailnet proxies). Devices use the hub's device ID and address `tcp://syncthing-sync.home.arpa:22000`; set a GUI username/password on first login.

## Commands

No build/test/lint tooling exists. Validate individual roots with:

```sh
kustomize build platforms          # whole platform tree
kustomize build apps               # all apps
kustomize build platforms/gitea    # single component
```

`kubectl kustomize <dir>` works equivalently. Helm-based components render fine this way because `HelmRelease`/`HelmRepository` are plain CRs; this does NOT validate chart values against the actual chart.

## Secrets and SOPS

`.sops.yaml`: all `*.yaml`/`*.yml` match; only `data`/`stringData` fields are encrypted (`encrypted_regex: "^(data|stringData)"`); single age recipient. The age private key on the admin machine is `~/.config/sops/age/k3s.agekey` (export `SOPS_AGE_KEY_FILE` if sops can't find it).

Encrypted Secrets currently committed: `platforms/tailscale-operator/secret-oauth.yaml`, `platforms/external-dns/secret-key.yaml`, `platforms/gitea/secret-values.yaml` (whole `values.yaml` chart values blob), `platforms/renovate/secret-token.yaml`, `platforms/cloudflared/secret-token.yaml`, `platforms/woodpecker-ci/secret-oauth.yaml`, `apps/homepage/secret-key.yaml`, `platforms/headlamp/secret-operator-token.yaml`.

Never replace `ENC[...]` values with plaintext. Edit encrypted files through SOPS (`sops <file>`), not with a plain editor. Non-`data`/`stringData` fields stay readable; keep sensitive values out of them.

### Gitea break-glass credentials

- `platforms/gitea/secret-values.yaml` (SOPS Secret `gitea-values`) embeds `values.yaml` with `gitea.admin.username`, `password`, and `passwordMode: keepUpdated`, loaded via `valuesFrom`. The password is reapplied when the admin configuration init container runs; it is not continuous password drift detection.
- Keep the chart admin account for emergency access and use a separate personal account for daily work. Changing the Secret's username can create another administrator; it does not rename or remove the old account. Account rotation does not require deleting PostgreSQL or repository PVCs.
- Store an emergency copy of the credentials in a password manager accessible without this cluster or Gitea. Back up the SOPS age private key and an encrypted copy of this repository outside the cluster as well; a cluster-only copy is not a break-glass recovery path.
- Rotate through SOPS, reconcile the Secret, and ensure the Gitea admin configuration init container runs again (a Secret-only update does not necessarily restart the pod). Verify login before updating the emergency copy.

### Gitea database credentials

- The same `gitea-values` Secret stores `postgresql.global.postgresql.auth`: database/username `gitea`, an application `password`, and an independent `postgresPassword` for the maintenance-only `postgres` superuser. Do not use the superuser as Gitea's application login or reuse the web-admin password.
- The Gitea chart derives its database `NAME`, `USER`, and `PASSWD` from these values; the PostgreSQL subchart uses them for its own Secret. Do not introduce a separate `gitea.config.database.PASSWD` source of truth.
- Updating Helm values or a Secret does not by itself rotate passwords in an initialized PostgreSQL database. Change the existing roles, verify TCP authentication, synchronize Secrets, then resume reconciliation. Keep all PVCs and database ownership intact.

## Conventions

- Resource names match the directory/component name; selectors use `app.kubernetes.io/name: <name>` (Tailscale operator follows upstream `app: operator` instead).
- Components get their own namespace in the same root, except `headlamp`, `coredns`, and the shared `traefik` configuration (all in `kube-system`).
- Images are pinned to explicit tags, except Tailscale operator (`stable`) and Headlamp (`latest`).
- Indentation is inconsistent across files (2-space vs 4-space in operator/SOPS files). Preserve local style when editing; don't reformat wholesale.

## Ingress and exposure

Tailnet is treated as LAN: every client uses Technitium (`100.90.10.86`, on the tailnet) for DNS and reaches the cluster by IP. No separate LAN-vs-tailnet tier. **No `*.ts.net` URLs in user-facing paths.**

- All app Ingresses use `ingressClassName: traefik` with hostnames like `app.home.arpa`. Tailnet clients reach the shared `kube-system/tailnet-traefik` Service, which forwards TCP 80/443 (and 22 for Gitea SSH) to Traefik. `publishedService.pathOverride` propagates the operator-assigned address to every Ingress and to external-dns. Do not add per-app node-IP targets or per-app Tailscale proxies for HTTP services.
- external-dns v0.22 collects both IP and hostname fields from Ingress status; its record-type conflict resolver prefers A/AAAA over a competing CNAME and can delete an owned obsolete CNAME under `--policy=sync`. Verify actual DNS records during cutover. No MagicDNS lookup is needed for IP records.
- The shared entry point is node-independent, but its Tailscale proxy and the current Traefik deployment are not highly available. App availability also depends on storage (node-local `local-path` PVCs).
- TLS for `*.home.arpa` is signed by the internal root CA via the `homelab-ca` ClusterIssuer: add annotation `cert-manager.io/cluster-issuer: homelab-ca` and a `spec.tls[].secretName`; cert-manager populates the Secret and Traefik hot-reloads it on renewal.
- Raw-TCP workloads that can't use HTTP Ingress use `type: LoadBalancer` + `loadBalancerClass: tailscale` with `tailscale.com/hostname` and `external-dns.kubernetes.io/hostname` annotations. The Tailscale operator's HTTP Ingress support is only for `*.ts.net` names; keep that out of this model. Gitea SSH (port 22) is the exception: it rides the shared `tailnet-traefik` entry point via the Traefik `ssh` entrypoint + `IngressRouteTCP`, so `gitea.home.arpa` serves both 443 and 22.
- Annotation prefix: external-dns v0.22+ uses `external-dns.kubernetes.io/`; legacy `external-dns.alpha.kubernetes.io/` annotations are ignored.

## CA trust (internal root)

The internal root CA (`homelab-root-ca`) is generated by cert-manager in-cluster. Devices must import it once to trust `*.home.arpa`:

```sh
kubectl -n cert-manager get secret homelab-root-ca \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > homelab-root-ca.crt
```

- Linux (Debian/Ubuntu): `sudo cp homelab-root-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates`. Firefox on Linux ignores the OS store by default: set `security.enterprise_roots.enabled=true` in `about:config` or import with `certutil` into NSS.
- macOS: `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain homelab-root-ca.crt`.
- Windows (admin PowerShell): `Import-Certificate -FilePath homelab-root-ca.crt -CertStoreLocation Cert:\LocalMachine\Root`.

The root key lives only in the cluster Secret `cert-manager/homelab-root-ca`. **Back it up** (e.g. SOPS-encrypt an export and commit it under `platforms/cert-manager/`); losing the cluster loses the trust anchor and forces re-import on every device.

## Gotchas

- `gotk-sync.yaml` holds the Git source of truth; manual edits can break reconciliation or be overwritten by `flux bootstrap`.
- `platforms/flux-system/gotk-components.yaml` is large generated YAML; avoid broad search/replace there.
- Removing a component means deleting its directory AND its entry in the parent `kustomization.yaml`; Flux `prune: true` will then delete it from the cluster.
- cert-manager CRDs install via `crds.enabled: true` on the HelmRelease. The first Flux apply of `platforms/cert-manager/bootstrap.yaml` races CRD installation and will transiently error on the `Certificate`/`ClusterIssuer` objects; Flux retries and self-heals. Leaf cert default is 90 days; the root is pinned to 10 years with `rotationPolicy: Never`.
- Flux→Gitea SSH uses the chart's in-cluster `gitea-ssh.gitea.svc.cluster.local:22`. The `flux-system` secret's `known_hosts` must contain that host pointing at Gitea's generated SSH host key: `kubectl get secret -n gitea gitea-ssh-host-keys -o jsonpath='{.data}'` (or `ssh-keyscan` from a debug pod), then `kubectl edit secret flux-system -n flux-system`.
- HelmRelease drift: deleting a Helm-managed resource does not self-heal (see Flux control flow). Also, a JSON Patch `add` on an existing `env` array REPLACES it; append with `/env/-` instead. A wiped `RENOVATE_CONFIG_FILE` makes Renovate silently default to platform `github` and fail with a generic "Authentication failure".
- CI pods (woodpecker-ci, renovate namespaces) trust the internal CA only via the kyverno `inject-ca` MutatingPolicy; if Kyverno is down, those namespaces cannot schedule pods at all.

## Adding a new component

1. Create a directory under `apps/` or `platforms/` with a local `kustomization.yaml` listing every file.
2. Add a `namespace.yaml` if it runs in its own namespace.
3. Register the directory in `apps/kustomization.yaml` or `platforms/kustomization.yaml`; this is what makes Flux deploy it.
4. Encrypt any Secret with SOPS before committing.
5. For Helm charts, follow the `kuberay-operator` pattern: `helm-repository.yaml` + `helm-release.yaml` with pinned chart version; put sensitive values in a SOPS Secret referenced via `valuesFrom`.
6. For web UIs, add `gethomepage.dev/*` annotations to the Ingress (group must match a Homepage layout group).
7. Validate with `kustomize build <dir>` before committing.
