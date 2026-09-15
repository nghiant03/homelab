# Shared tailnet ingress

Traffic: tailnet client → Tailscale TCP proxy → Traefik → app Service/pods.
TLS terminates at Traefik using cert-manager's `homelab-ca` certificates.

The operator assigns the `traefik-tailnet` LoadBalancer an address. Traefik copies
its status to app Ingresses via `publishedService.pathOverride`; external-dns
publishes their `*.home.arpa` records in Technitium. No node IP is configured.
The proxy device is named `traefik-tailnet`; that name is not an application URL.

This root customizes the existing k3s-managed Traefik using `HelmChartConfig`.
It does not install another controller or replace the existing `traefik` Service.
The configuration targets the observed k3s chart `39.0.701+up39.0.7` and its
`web`/`websecure` pod ports (8000/8443). The existing `HelmChart.spec.set` must not
override `providers.kubernetesIngress.publishedService`.

## Staged rollout

Run cluster commands on the node from an updated checkout. Keep the parent Flux
Kustomization suspended during this cutover. No database or credential changes
are required here. Before creating the shared proxy, ensure tailnet policy allows
the intended clients to reach its operator-assigned tags on TCP 80/443; permissions
limited to the old per-app device do not automatically apply to the new proxy.

### 1. Create and test the shared proxy

```sh
sudo kubectl apply -f platforms/traefik/service.yaml
sudo kubectl -n kube-system get service traefik-tailnet -w
```

Wait for an operator-assigned IP in `EXTERNAL-IP`, then stop the watch. Inspect
the status and backend readiness:

```sh
sudo kubectl -n kube-system get service traefik-tailnet \
  -o jsonpath='{.status.loadBalancer.ingress}{"\n"}'
sudo kubectl -n kube-system get endpointslices \
  -l kubernetes.io/service-name=traefik-tailnet
```

The endpoints must match ready Traefik pods and expose ports 8000 and 8443.
Before changing DNS, test from a tailnet client with the exported public root CA
certificate (see `AGENTS.md` for export and OS trust installation):

```sh
TAILNET_IP="REPLACE_WITH_SHARED_SERVICE_IP"
curl --cacert homelab-root-ca.crt \
  --resolve "gitea.home.arpa:443:$TAILNET_IP" \
  --connect-timeout 5 --max-time 15 -sS -D - -o /dev/null \
  https://gitea.home.arpa/
```

Require a successful TLS verification and HTTP 200 (or an expected app redirect).
The IP above is only a diagnostic input, not a value to save in Git or DNS.

### 2. Switch automatic address publication

```sh
sudo kubectl apply -f platforms/traefik/helm-chart-config.yaml
sudo kubectl -n kube-system get deployment traefik -w
```

The k3s Helm controller upgrades Traefik asynchronously. After the deployment
updates, stop the watch and verify the rollout and effective argument:

```sh
sudo kubectl -n kube-system rollout status deployment/traefik --timeout=180s
sudo kubectl -n kube-system get deployment traefik \
  -o jsonpath='{.spec.template.spec.containers[0].args}{"\n"}'
sudo kubectl get ingress -A -o wide
```

Require exactly one `--providers.kubernetesingress.ingressendpoint.publishedservice`
argument, with value `kube-system/traefik-tailnet`. App Ingress addresses must match
the shared Service status, rather than node addresses. A rollout command can
return early before the Helm controller acts; the effective argument and Ingress
status are the decisive checks.

Remove the competing hostname annotation from the legacy Gitea Service, if it
still exists (it is already absent from the desired repository manifests):

```sh
sudo kubectl -n gitea annotate service gitea-tailscale \
  external-dns.kubernetes.io/hostname-
```

### 3. Verify DNS and trusted HTTPS

```sh
dig @100.114.255.114 gitea.home.arpa A +short
dig @100.114.255.114 gitea.home.arpa CNAME +short
sudo kubectl -n external-dns logs deployment/external-dns --since=5m
```

The A record must contain the shared proxy IPv4 address, not the old Gitea proxy
or node IP. The CNAME query should be empty. external-dns v0.22.0 prefers IP
records when status also includes a Tailscale hostname; a conflict warning about
discarding a CNAME candidate is expected. It can remove old CNAMEs it owns under
the existing TXT registry and sync policy. If a stale record remains, inspect
ownership/update errors instead of assuming DNS converged.

After DNS caches expire, on the client:

```sh
curl --cacert homelab-root-ca.crt --connect-timeout 5 --max-time 15 \
  -sS -D - -o /dev/null https://gitea.home.arpa/
```

Install the root CA into the client's OS/browser trust store, then verify browser
login. Repeat DNS/TLS checks for Homepage and Headlamp once their Ingresses and
Certificates are ready. Keep MagicDNS/conditional-forwarder cleanup separate until
all remaining consumers have been checked.

### 4. Retire the old proxy and resume GitOps

After the new path works, delete only the obsolete exposure Service:

```sh
sudo kubectl -n gitea delete service gitea-tailscale --ignore-not-found
```

The Tailscale operator cleans up its proxy. This does not delete Gitea's pods,
database, SSH ClusterIP Service, or PVCs. Keep Soft Serve's dedicated TCP exposure.

Ensure this change has been committed/pushed and the live Flux GitRepository has
fetched that revision before resuming the parent Kustomization. If Gitea's
temporary no-rollback remediation settings from database recovery are still
active, restore its repository settings after confirming the successful upgrade
and login. Resume the parent with:

```sh
sudo kubectl -n flux-system patch kustomization flux-system --type merge \
  -p '{"spec":{"suspend":false}}'
```

## Adding apps and nodes

New web apps need a Service and a Traefik Ingress with a `home.arpa` hostname,
`homelab-ca` annotation and TLS Secret name. They need no IP override or additional
Tailscale proxy. Add manifests to their Kustomize root as usual.

This standalone proxy is node-independent, not highly available. Preserve its
operator-managed identity/state when rescheduling. Node failure can interrupt
access while the proxy or Traefik recovers. HA needs a supported ingress
`ProxyGroup`, multiple Traefik replicas spread across nodes, and suitable app
storage; current Gitea `local-path` PVCs remain tied to their storage node.

References:
- https://tailscale.com/kb/1439/kubernetes-operator-cluster-ingress
- https://docs.k3s.io/networking/networking-services#traefik-ingress-controller
- https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/#ingressendpointpublishedservice
- https://github.com/kubernetes-sigs/external-dns/blob/v0.22.0/plan/conflict.go
