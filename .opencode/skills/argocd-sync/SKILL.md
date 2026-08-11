---
name: argocd-sync
description: Trigger an ArgoCD sync after pushing ApplicationSet changes to k8s-homelab. Use whenever the user says a change was pushed and needs syncing, or asks to sync/refresh a specific ArgoCD Application.
---
## What I do
Syncs an ArgoCD Application after its manifest changed in git, using the argocd-mcp tools.

## Procedure
1. Identify the child Application name: ApplicationSets generate one per
   cluster as `<cluster>-<appset-name>` (e.g. `appset-ik-llama-qwen3-6-27b.yaml`
   → `k8s02-ik-llama-qwen3-6-27b`). Use `list_applications` if unsure.
2. Call `sync_application` for that Application.
3. Verify: check the Application's sync status and health afterward
   (`get_application`) - confirm `status.sync.status: Synced` and
   `status.health.status: Healthy`, and that it's actually running the new
   change, not just reporting "Synced" against a stale cached manifest.
4. If the change doesn't appear to have taken effect despite a "Synced"
   result (a known ArgoCD gotcha - the repo-server can serve a cached
   Helm render), try again after a short pause, or ask the user to
   confirm before assuming success.
