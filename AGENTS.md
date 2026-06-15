# Agent Guide

This is a Talos Linux Kubernetes homelab cluster managed with GitOps (ArgoCD) and SOPS encryption. Read this before making any changes.

## Cluster

- **Control plane:** `192.168.0.150` (also schedulable — no NoSchedule taint)
- **Worker:** `192.168.0.155`
- **Talos:** v1.13.4 | **Kubernetes:** v1.36.1
- **MetalLB pool:** `192.168.0.225–192.168.0.255`
- **Ingress:** Traefik v3 using native Kubernetes Gateway API (`HTTPRoute`, not `IngressRoute`)
- **Auth:** Authentik at `authentik.home.local` — GitHub OAuth upstream, OIDC downstream to ArgoCD and future services
- **TLS:** cert-manager with Cloudflare DNS-01, wildcard cert for `*.petzko.sh`
- **DDNS:** `favonia/cloudflare-ddns` keeps `petzko.sh` A records current

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
- Never mix: services like Longhorn, ArgoCD, Traefik dashboard are internal-only forever

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

**4. Expose via Traefik** → add an `HTTPRoute` to `kubernetes/manifests/<name>-config/httproute.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <name>
  namespace: <namespace>
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: traefik-gateway
      namespace: traefik
  hostnames:
    - <name>.home.local
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - group: ""
          kind: Service
          name: <service-name>
          port: <port>
          weight: 1
```

Always include `group`, `kind`, and `weight` — Gateway API webhook adds these defaults and ArgoCD will report drift without them.

**5. CoreDNS** → add the new hostname to `kubernetes/manifests/coredns/configmap.yaml` in the `hosts` block so in-cluster pods can resolve it:
```
192.168.0.225 <name>.home.local
```

**6. Sync waves** — set `argocd.argoproj.io/sync-wave` annotations to enforce ordering:
- Wave 1: CRDs, MetalLB, CoreDNS
- Wave 2: MetalLB config (IPAddressPool/L2Advertisement)
- Wave 3: Traefik
- Wave 4: Traefik config, service HTTPRoutes
- Wave 5+: Applications

## Namespace Pod Security Standards

Talos enforces `baseline` PSS by default. Namespaces that need privileged pods (MetalLB speaker, local-path-provisioner) require labels. For ArgoCD-managed namespaces use `managedNamespaceMetadata`:

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

- **Never commit secrets to git.**
- Sensitive values (API keys, passwords, OIDC secrets) are created manually as Kubernetes Secrets before ArgoCD syncs the app.
- Document the required secret name and keys in a comment in the values file.
- ArgoCD OIDC secret lives in `argocd-secret` under key `oidc.authentik.clientSecret`.
- Authentik secrets live in `authentik/authentik-secrets` with keys `AUTHENTIK_SECRET_KEY` and `postgresql-password`.

## Bootstrap (fresh cluster)

```sh
cd kubernetes
helmfile sync   # installs ArgoCD + applies root Application
```

Then in ArgoCD UI:
1. Add repo credentials (SSH key for `git@github.com:petzkod5/rivendell-talos-cluster.git`)
2. Sync the `root` Application — ArgoCD installs everything else in wave order
3. Pre-create namespace secrets before syncing apps that need them (Authentik, etc.)

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
