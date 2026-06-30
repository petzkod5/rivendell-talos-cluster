# Agent Guide

This is a Talos Linux Kubernetes homelab cluster managed with GitOps (ArgoCD) and SOPS encryption. Read this before making any changes.

## Cluster

- **Control plane:** `192.168.0.150` (also schedulable — no NoSchedule taint)
- **Worker:** `192.168.0.155` (HP thin client — EFI variable store is limited, `talosctl upgrade` requires clean EFI NVRAM)
- **Talos:** v1.13.4 with factory schematic `613e1592b2da41ae` (includes `iscsi-tools` + `util-linux-tools` for Longhorn)
- **Kubernetes:** v1.36.1
- **MetalLB pool:** `192.168.0.225–192.168.0.255`
- **Ingress:** Traefik v3 using native Kubernetes Gateway API (`HTTPRoute`, not `IngressRoute`)
- **Auth:** Authentik at `authentik.home.local` — GitHub OAuth upstream, OIDC downstream to ArgoCD and future services. ArgoCD RBAC defaults authenticated users to `role:readonly`; Authentik group `authentik Admins` maps to `role:admin`. The Hermes dashboard is gated by Authentik forward-auth (domain-level) via Traefik Middleware `hermes/authentik-forwardauth`; the dashboard's own gate is disabled with `HERMES_DASHBOARD_INSECURE=1`, so Authentik is the single login.
- **Hermes WebUI:** the community UI (`nesquena/hermes-webui`) runs at `hermes-ui.home.local`, gated by the same Authentik forward-auth Middleware `hermes/authentik-forwardauth` (its own auth is disabled, so Authentik is the single gate). It is a co-located second runtime (separate Deployment pinned to the agent's node, sharing the `hermes-data` PVC with `HERMES_HOME=/opt/data`) that reads the agent's `state.db` directly; the agent Deployment is untouched.
- **Hermes terminal backend:** **ssh** — the agent runs shell commands on the Ubuntu VM `petzko-ubuntu-vm.pintail-monster.ts.net` as user `hermes` (NOPASSWD sudo), reached via the Tailscale operator egress Service `hermes/ubuntu-vm`. The key comes from Secret `hermes-ssh-key`, placed by the `seed-ssh-key` initContainer at `/opt/data/home/.ssh/id_ed25519` (mode 0600, UID 10000). Configured **two places that must stay in sync**: `TERMINAL_*` env in the agent Deployment (the gateway / API / Telegram / Photon path reads env directly) **and** the full `terminal.{backend,cwd,ssh_host,ssh_user,ssh_port,ssh_key}` block in `config.yaml` (seeded by the `seed-ssh-config` initContainer). Both the `hermes.home.local` dashboard chat and the webui spawn workers that bridge `config.yaml`→env and do **not** inherit the Deployment's `TERMINAL_SSH_*`, so the connection params must be in `config.yaml` too — `backend: ssh` alone yields `ValueError: SSH environment requires ssh_host and ssh_user`. **Port 2222, not 22:** the VM has Tailscale SSH enabled (`RunSSH`), which hijacks tailnet `:22` and bypasses key auth — so the VM's real `sshd` also listens on `2222` (via an `ssh.socket` systemd drop-in) and the agent does key auth there. The Tailscale **network grant** `tag:k8s` → VM `tcp:2222` authorizes the egress proxy (SSH-rule egress is impossible: Tailscale forbids a tagged node from SSHing into a user-owned device). ArgoCD `ignoreDifferences` lets the operator keep its `spec.externalName` rewrite on the egress Service. The agent Deployment also sets `HERMES_WRITE_SAFE_ROOT=""` (empty), overriding the image's `/opt/data` default: that guard confines `write_file` to a local root and would otherwise deny every remote write as a "protected system/credential file" — the credential-by-name deny list still applies, and the agent has root via sudo regardless.
- **Hermes Ollama provider:** the agent can use the local Ollama on `petzko-pc-01.pintail-monster.ts.net:11434` (model `qwen2.5-coder:14b`), reached via a second Tailscale operator egress Service `hermes/ollama` at `http://ollama.hermes.svc.cluster.local:11434/v1`. Registered as a `custom_providers` entry (name `ollama`) in `config.yaml` + the seed ConfigMap; selectable in the model picker as `custom:ollama`. Requires the tailnet grant `tag:k8s` → petzko-pc-01 `tcp:11434` (use a `hosts` alias in the policy — the IP is set manually). The default `model.*` is left unchanged.
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
  - repoURL: https://github.com/petzkod5/rivendell-talos-cluster.git
    targetRevision: main
    ref: values
```

For local manifests, use directory source:
```yaml
source:
  repoURL: https://github.com/petzkod5/rivendell-talos-cluster.git
  targetRevision: main
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

**Never commit plaintext secrets to git.** Talos machine configs remain SOPS+age encrypted. Kubernetes application secrets are being migrated to Bitwarden Secrets Manager via External Secrets Operator (ESO).

Bitwarden Secrets Manager:
- Organization: `Elessar` (`37b972a5-3124-4652-aedd-b46900380e9d`)
- Project: `rivendell-talos-cluster` (`ce084d0f-926e-46b9-9299-b4770120e063`)
- ESO bootstrap credential: manually create/update Secret `external-secrets/bitwarden-access-token` with key `token` before syncing ESO config.
- Current app-secret migrations use `creationPolicy: Merge` for safe live rollout. Fresh-cluster bootstrap still needs target Secret objects to exist until a later PR switches selected secrets to `Owner` or `Orphan`.

| Secret | Namespace | Keys | Source | Purpose |
|---|---|---|---|---|
| `argocd-secret` | `argocd` | `admin.password`, `admin.passwordMtime`, `oidc.authentik.clientSecret`, `server.secretkey` | ESO + Bitwarden (`k8s/argocd/argocd-secret/*`) | ArgoCD local admin, OIDC client secret, and session signing key |
| `authentik-secrets` | `authentik` | `AUTHENTIK_SECRET_KEY`, `postgresql-password` | ESO + Bitwarden (`k8s/authentik/authentik-secrets/*`) | Authentik + PostgreSQL |
| `cloudflare-api-token` | `cert-manager` | `api-token` | ESO + Bitwarden (`k8s/cert-manager/cloudflare-api-token/api-token`) | cert-manager DNS-01 challenges |
| `cloudflare-api-token` | `cloudflare-ddns` | `api-token` | ESO + Bitwarden (`k8s/cloudflare-ddns/cloudflare-api-token/api-token`) | DDNS A record updates |
| `grafana-admin` | `monitoring` | `admin-user`, `admin-password` | ESO + Bitwarden (`k8s/monitoring/grafana-admin/*`) | Grafana bootstrap/admin credentials |
| `grafana-oidc` | `monitoring` | `GF_AUTH_GENERIC_OAUTH_CLIENT_ID`, `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET` | ESO + Bitwarden (`k8s/monitoring/grafana-oidc/*`) | Grafana Authentik OIDC client credentials |
| `hermes-secrets` | `hermes` | `OPENROUTER_API_KEY`, `API_SERVER_KEY`, `PHOTON_ALLOWED_USERS` | Manual for now | Hermes LLM provider key + API-server bearer token + Photon iMessage allowlist (E.164, comma-separated). Dashboard auth is via Authentik forward-auth (Traefik Middleware `hermes/authentik-forwardauth`), not in-app basic-auth. |
| `hermes-ssh-key` | `hermes` | `id_ed25519` | Manual for now | SSH key the agent uses for its ssh terminal backend (`hermes@petzko-ubuntu-vm`, NOPASSWD sudo). |
| `renovate` | `renovate` | `token` | ESO + Bitwarden (`k8s/renovate/renovate/token`) | Renovate GitHub token |

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

Some app secrets are already ESO + Bitwarden backed but still use `creationPolicy: Merge`; create the target Secret objects during fresh bootstrap until a later PR moves them to a creation policy that can recreate missing Secrets.

```sh
# cert-manager Cloudflare token target Secret for ESO Merge-mode bootstrap
kubectl create namespace cert-manager
kubectl create secret generic cloudflare-api-token -n cert-manager \
  --from-literal=api-token=<token>

# DDNS Cloudflare token target Secret for ESO Merge-mode bootstrap
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
1. Add repo credentials (SSH key for `https://github.com/petzkod5/rivendell-talos-cluster.git`)
2. Sync the `root` Application — ArgoCD installs everything else in wave order
3. After Authentik is up: configure GitHub OAuth source and create ArgoCD OIDC provider
4. Store/update the ArgoCD OIDC client secret in Bitwarden (`k8s/argocd/argocd-secret/oidc.authentik.clientSecret`) and let ESO sync `argocd-secret`. Emergency manual patches to `argocd-secret` must be followed by updating Bitwarden, or ESO will restore the Bitwarden value.

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
