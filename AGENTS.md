# AGENTS.md — k8s-homelab

## What this repo is

GitOps repository for a multi-cluster Kubernetes homelab, managed entirely through ArgoCD ApplicationSets. All YAML files are ArgoCD `Application` or `ApplicationSet` resources. There is no application code, no CI, no tests, and no lint step — changes are validated by ArgoCD sync.

**Remote**: `git@github.com:vadimzharov/k8s-homelab.git`

---

## Directory layout

| Directory | Purpose | ArgoCD project |
|---|---|---|
| `k8s-hub-apps/` | Bootstrap: ArgoCD AppProjects, App-of-Apps, secrets plugin, Vault ClusterSecretStore | — |
| `k8s-hub-clusters-configuration/` | `ExternalSecret` resources that pull secrets from Vault into K8s Secrets (per-cluster and shared) | `k8s-homelab-clusters-config` |
| `appsets-k8s-cluster/` | Cluster-level workloads (Cilium config, ingress certs, metrics-server, External Secrets operator) | `k8s-homelab-cluster` |
| `appsets-k8s-infra/` | Infrastructure services (Pi-hole, Tailscale, CSI storage, cert-manager, Velero, WireGuard) | `k8s-homelab-infra` |
| `appsets-k8s-apps/` | End-user applications (Immich, Jellyfin, LiteLLM, Ollama, vLLM, Arr stack, etc.) — 63 appsets | `k8s-homelab-apps` |
| `k8s-cluster/` | Standalone manifests for cluster operators (KubeVirt, CDI, ingress cert reading) — not ArgoCD-managed | — |
| `deprecated/` | Retired appsets kept for reference (e.g. old Cilium helm install) | — |

---

## Architecture

### App-of-Apps pattern

`k8s-hub-apps/argocd-k8s-app-of-apps.yaml` is the single entrypoint. It contains three `ApplicationSet` generators that fan out to registered clusters based on cluster labels:

- Clusters with `appsets-infra: "installed"` → deploy everything from `appsets-k8s-infra/`
- Clusters with `appsets-apps: "installed"` → deploy everything from `appsets-k8s-apps/`
- Clusters with `appsets-cluster: "installed"` → deploy everything from `appsets-k8s-cluster/`

To target an appset at a specific cluster, add the matching label to that cluster's ArgoCD `Cluster` resource (e.g. `immich-m: "installed"`).

### Secrets & environment variables

The **ArgoCD Secrets Plugin** is the preferred mechanism for injecting any environment-specific value into ApplicationSets — not just secrets, but any variable that differs per deployment target (e.g. `{{ domainname }}`, `{{ piholepassword }}`, usernames, IPs, etc.). The full flow:

1. **Vault** is the source of truth. `k8s-hub-apps/secrets-store.yaml` defines a `ClusterSecretStore` pointing at `vault-ui.vault.svc.cluster.local`.
2. **ExternalSecrets** in `k8s-hub-clusters-configuration/` pull from Vault into K8s `Secret` objects in the `argocd` namespace.
3. **ArgoCD Secrets Plugin** (`k8s-hub-apps/argocd-secrets-plugin.yaml`) reads those K8s Secrets and injects values into ApplicationSet templates at render time. Appsets reference it via `plugin.configMapRef.name: argocd-secrets-plugin`.
4. Template variables (`{{ variableName }}`) are resolved by the plugin from the named K8s Secret.

**Never commit real values.** All sensitive or environment-specific data flows through Vault → ExternalSecret → ArgoCD plugin.

**When generating a new appset**: if the template requires a variable that doesn't exist in any known ExternalSecret, **do not invent a value** — notify the user and wait for them to create the variable in the appropriate secret before proceeding.

### Helm + Multi-source patterns

Appsets use Helm charts (bjw-s/app-template, upstream charts, custom OCI charts) with multi-source ArgoCD. Common pattern:
- Source 1: `bjw-s/app-template` for raw resources (PVCs, CRDs, Secrets, backups)
- Source 2: Application-specific chart for the actual workload

### Ingress / routing

Two ingress paths coexist:
- **NGINX Ingress** — used by older appsets (annotation `ingressClassName: nginx`)
- **Cilium Gateway API** — used by newer appsets (`HTTPRoute` with `parentRefs: cilium-gw` in `cilium-system`)

### Dashboard annotations

Appsets annotated with `appset-dashboard-appname` and `appset-dashboard-groupname` are picked up by the custom dashboard app (`appset-apps-dashboard.yaml`) to build a browsable application catalog.

---

## Working with this repo

### Before making changes
- If adding a new appset, decide which directory it belongs to (`infra` vs `apps` vs `cluster`) and which project it uses.
- For appsets in `appsets-k8s-apps/`, the **ArgoCD Secrets Plugin** is the strongly preferred mechanism for injecting any environment-specific value (not just secrets — e.g. `{{ domainname }}`, usernames, IPs). Reference an existing `ExternalSecret` name in the plugin generator, or add a new `ExternalSecret` in `k8s-hub-clusters-configuration/` if needed.

### YAML gotchas
- ArgoCD is sensitive to YAML indentation in multi-source ApplicationSets. Helm `values:` blocks must be indented correctly under `helm:` — a common mistake is misindenting `releaseName` or `values` relative to the `helm:` key.
- The `forceRename` field in `bjw-s/app-template` raw resources is required when multiple raw resources share a kind (prevents name collision in the rendered manifest).

### LiteLLM specifics
- `appset-litellm.yaml` is the largest and most complex appset (571 lines). It contains inline Python callbacks, multi-source Helm, and a custom `smart-router` model. Detailed comments in the file explain routing tiers, GPU model status, and known issues.
- The vLLM backends (qwen2.5-7b, gpt-oss-20b) are currently decommissioned — the single RTX 3090 runs ik_llama.cpp serving Qwen3.6-27B instead.
- kagent's ModelConfig is not in this repo (it's managed separately) and currently points at the decommissioned `gpt-oss-20b`.

### .gitignore
- `k8s-hub-argocd/*` is ignored — this directory (when present locally) contains ArgoCD installation files not tracked in git.
