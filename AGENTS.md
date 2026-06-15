# Agent Guide

This is a Talos Linux Kubernetes homelab cluster managed with GitOps (ArgoCD) and SOPS encryption. Read this before making any changes.

## Cluster

- **Control plane:** `192.168.0.150` (also schedulable — no NoSchedule taint)
- **Worker:** `192.168.0.155` (HP thin client — EFI variable store is limited, `talosctl upgrade` requires clean EFI NVRAM)
- **Talos:** v1.13.4 with factory schematic `613e1592b2da41ae` (includes `iscsi-tools` + `util-linux-tools` for Longhorn)
- **Kubernetes:** v1.36.1
- **MetalLB pool:** `192.168.0.225–192.168.0.255`
- **Ingress:** Traefik v3 using native Kubernetes Gateway API (`HTTPRoute`, not `IngressRoute`)
- **Auth:** Authentik at `authentik.home.local` — GitHub OAuth upstream, OIDC downstream to ArgoCD and future services
- **TLS:** cert-manager v1.17.2 with Cloudflare DNS-01, wildcard cert `*.petzko.sh` stored in `traefik/petzko-sh-tls`
- **DDNS:** `favonia/cloudflare-ddns` keeps `petzko.sh` and `*.petzko.sh` A records current
- **Storage:** Longhorn (replicated) + local-path-provisioner (default StorageClass for lightweight use)
- **Domain:** `petzko.sh` registered and DNS managed via Cloudflare

## Dual-Traefik Network Architecture

Two LoadBalancer IPs on the **same** Traefik deployment — one internal, one external:

| Service | MetalLB IP | Ports | Purpose |
|---|---|---|---|
| `traefik` (helm-managed) | `192.168.0.225` | 80, 8080 | Internal only — `*.home.local` |
| `traefik-external` (raw manifest) | `192.168.0.226` | 443 | External — `*.petzko.sh`, router port-forwards here |

Router forwards port `443` → `192.168.0.226` **only**. The `.225` IP is never exposed publicly.

**Gateway listeners on `traefik-gateway`:**
- `web` (port 8000) → `sectionName: web` — internal `*.home.local` HTTP routes
- `websecure` (port 8443) → `sectionName: websecure` — external `*.petzko.sh` HTTPS routes, TLS via `petzko-sh-tls` cert

**HTTPRoute conventions:**
- Internal services: `parentRefs[].sectionName: web`, hostname `*.home.local`
- External services: `parentRefs[].sectionName: websecure`, hostname `*.petzko.sh`
- Never mix: Longhorn, ArgoCD, Traefik dashboard, Authentik are internal-only forever

## Repository layout

```
talos/          # SOPS-encrypted Talos machine configs
kubernetes/
  helmfile.yaml         # Bootstrap only — installs ArgoCD, applies root App
  apps/                 # ArgoCD Application definitions (App-of-Apps pattern)
  manifests/            # Raw Kubernetes manifests (applied via ArgoCD directory source)
  values/               # Helm values files (referenced via ArgoCD multi-source)
```

## Talos machine configs

**Never decrypt, edit, and re-encrypt these files programmatically.** The risk of wiping a file via a failed redirect is too high.

- When a Talos config change is needed, provide the YAML snippet and location — the user applies it manually.
- `talosctl` is configured globally via `~/.talos/` — never pass `--talosconfig` or decrypt `talos/talosconfig`.
- Run `talosctl` commands directly: `talosctl <cmd> -n 192.168.0.150`
- SOPS key is at `~/.config/sops/age/keys.txt`. Set `SOPS_AGE_KEY_FILE` if needed.
- `.sops.yaml` encrypts everything matching `talos/*.yaml` and `talos/talosconfig`.
- Worker node (`192.168.0.155`) install disk is `/dev/sda` (no USB present). Config has `/dev/sda`.

## Adding a new service

Follow this pattern every time:

**1. ArgoCD Application** → `kubernetes/apps/<name>.yaml`

For upstream Helm charts, use multi-source:
```yaml
sources:
  - repoURL: https://<chart-repo>
    chart: <chart>
    targetRevision: <version>
    helm:
      valueFiles:
        - $values/kubernetes/values/<name>.yaml
  - repoURL: git@github.com:petzkod5/rivendell-talos-cluster.git
    targetRevision: HEAD
    ref: values
```

For local manifests, use directory source:
```yaml
source:
  repoURL: git@github.com:petzkod5/rivendell-talos-cluster.git
  targetRevision: HEAD
  path: kubernetes/manifests/<name>
```

**2. Values file** → `kubernetes/values/<name>.yaml` (for Helm charts)

**3. Manifests** → `kubernetes/manifests/<name>/` (for raw YAML — CRs, config, HTTPRoutes)

**4. Expose via Traefik** — choose internal or external:

Internal (`*.home.local`, never public):
```yaml
parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: traefik-gateway
    namespace: traefik
    sectionName: web
hostnames:
  - <name>.home.local
```

External (`*.petzko.sh`, publicly reachable via port-forwarded `.226`):
```yaml
parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: traefik-gateway
    namespace: traefik
    sectionName: websecure
hostnames:
  - <name>.petzko.sh
```

Always include `group`, `kind`, and `weight` on `backendRefs` — Gateway API webhook adds these defaults and ArgoCD will report drift without them.

**5. CoreDNS** — internal services only. Add to `kubernetes/manifests/coredns/configmap.yaml`:
```
192.168.0.225 <name>.home.local
```
External `*.petzko.sh` services resolve via public Cloudflare DNS — no CoreDNS entry needed.

**6. Sync waves** — set `argocd.argoproj.io/sync-wave` annotations to enforce ordering:
- Wave 1: CRDs, MetalLB, CoreDNS
- Wave 2: MetalLB config (IPAddressPool/L2Advertisement)
- Wave 3: Traefik, cert-manager, metrics-server
- Wave 4: Traefik config, cert-manager config, Longhorn, Authentik, cloudflare-ddns
- Wave 5+: Application-level services

## Namespace Pod Security Standards

Talos enforces `baseline` PSS by default. Namespaces that need privileged pods (MetalLB, Longhorn, local-path-provisioner) require labels. For ArgoCD-managed namespaces use `managedNamespaceMetadata`:

```yaml
syncPolicy:
  managedNamespaceMetadata:
    labels:
      pod-security.kubernetes.io/enforce: privileged
      pod-security.kubernetes.io/audit: privileged
      pod-security.kubernetes.io/warn: privileged
  syncOptions:
    - CreateNamespace=true
```

## Secrets

**Never commit secrets to git.** Create Kubernetes Secrets manually before ArgoCD syncs the app.

| Secret | Namespace | Keys | Purpose |
|---|---|---|---|
| `argocd-secret` | `argocd` | `oidc.authentik.clientSecret` | ArgoCD OIDC via Authentik |
| `authentik-secrets` | `authentik` | `AUTHENTIK_SECRET_KEY`, `postgresql-password` | Authentik + PostgreSQL |
| `cloudflare-api-token` | `cert-manager` | `api-token` | cert-manager DNS-01 challenges |
| `cloudflare-api-token` | `cloudflare-ddns` | `api-token` | DDNS A record updates |

## CoreDNS

CoreDNS uses explicit upstream nameservers `1.1.1.1` and `8.8.8.8` (not `/etc/resolv.conf`). Talos's `hostDNS.forwardKubeDNSToHost` makes `/etc/resolv.conf` point to `127.0.0.53` which is unreachable from pod network namespaces.

When adding a new `*.home.local` service, patch both:
1. `kubernetes/manifests/coredns/configmap.yaml` (git — ArgoCD applies)
2. `kubectl patch configmap coredns -n kube-system` (live — takes effect immediately)

Then restart CoreDNS: `kubectl rollout restart deployment/coredns -n kube-system`

## Longhorn

- `preUpgradeChecker.jobEnabled: false` is required for ArgoCD compatibility — without it, the pre-upgrade hook fails because the service account doesn't exist yet when the hook runs.
- Data path: `/var/lib/longhorn` (configured via `machine.kubelet.extraMounts` on both nodes)
- Both nodes have the factory schematic with `iscsi-tools` and `util-linux-tools`
- `longhorn-system` namespace requires privileged PSS

## Bootstrap (fresh cluster)

**Pre-requisites before `helmfile sync`:**
```sh
# cert-manager Cloudflare token
kubectl create namespace cert-manager
kubectl create secret generic cloudflare-api-token -n cert-manager \
  --from-literal=api-token=<token>

# DDNS Cloudflare token
kubectl create namespace cloudflare-ddns
kubectl create secret generic cloudflare-api-token -n cloudflare-ddns \
  --from-literal=api-token=<token>

# Authentik secrets
kubectl create namespace authentik
kubectl create secret generic authentik-secrets -n authentik \
  --from-literal=AUTHENTIK_SECRET_KEY=<key> \
  --from-literal=postgresql-password=<password>
```

**Bootstrap:**
```sh
cd kubernetes
helmfile sync   # installs ArgoCD + applies root Application
```

Then in ArgoCD UI:
1. Add repo credentials (SSH key for `git@github.com:petzkod5/rivendell-talos-cluster.git`)
2. Sync the `root` Application — ArgoCD installs everything else in wave order
3. After Authentik is up: configure GitHub OAuth source and create ArgoCD OIDC provider
4. Patch `argocd-secret` with OIDC client secret

## Commit conventions

Use [Conventional Commits](https://www.conventionalcommits.org/):
- `feat:` new service or capability
- `fix:` bug fix
- `refactor:` restructuring without behaviour change
- `chore:` dependency updates, version bumps

Always include:
```
Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
```
