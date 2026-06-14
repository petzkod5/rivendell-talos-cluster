# rivendell-talos-cluster

Configuration for my homelab Talos Kubernetes cluster.

- **Control plane:** `192.168.0.150`
- **Worker:** `192.168.0.155`

## Layout

- `talos/` — Talos machine configs and `talosconfig` (SOPS-encrypted with age)
- `kubernetes/` — in-cluster manifests
- `.sops.yaml` — encryption rules

## Prerequisites

```sh
brew install sops age talosctl kubectl helm
```

Restore the age private key from Bitwarden to `~/.config/sops/age/keys.txt` (mode `600`), then:

```sh
export SOPS_AGE_KEY_FILE="$HOME/.config/sops/age/keys.txt"
```

(macOS doesn't search `~/.config` by default; the export is in `~/.zshrc`.)

## SOPS usage

Edit in place (decrypts to `$EDITOR`, re-encrypts on save):

```sh
sops talos/controlplane.yaml
```

Decrypt to stdout:

```sh
sops -d talos/talosconfig
```

Decrypt in place (for use with `talosctl`, then re-encrypt):

```sh
sops -d -i talos/talosconfig
# ... use talosconfig ...
sops -e -i talos/talosconfig
```

Encrypt a new file (rules in `.sops.yaml` pick the recipient):

```sh
sops -e -i talos/<file>.yaml
```

## Applying Talos config

```sh
talosctl --talosconfig <(sops -d talos/talosconfig) \
  apply-config --nodes 192.168.0.150 --file <(sops -d talos/controlplane.yaml)
```
