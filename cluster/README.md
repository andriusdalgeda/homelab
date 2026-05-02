# Cluster Secrets

This cluster uses a small set of manually created bootstrap secrets, then
External Secrets pulls the rest from Azure Key Vault through the
`azure` `ClusterSecretStore`.

Store Azure Key Vault values as plain text unless a tool explicitly says
otherwise. Kubernetes and External Secrets handle Kubernetes Secret encoding.

## Bootstrap Secrets

These secrets must exist before the cluster can fully reconcile.

| Kubernetes secret | Namespace | Keys | Where to get it |
| --- | --- | --- | --- |
| `flux-system` | `flux-system` | Flux Git SSH identity keys | Created by `flux bootstrap github` or equivalent Flux bootstrap command. This is the deploy key Flux uses to read this Git repository. |
| `azure-keyvault-config` | `flux-system` | `AZURE_TENANT_ID`, `AZURE_KEYVAULT_URL` | `AZURE_TENANT_ID` comes from Microsoft Entra ID tenant properties. `AZURE_KEYVAULT_URL` comes from the Azure Key Vault overview page, for example `https://my-vault.vault.azure.net/`. |
| `azure-secret-sp` | `external-secrets` | `ClientID`, `ClientSecret` | Create an Entra ID app registration/service principal for External Secrets. `ClientID` is the application/client ID. `ClientSecret` is a client secret generated under Certificates & secrets. Grant this identity permission to read secrets from the Key Vault. |

Example bootstrap commands:

```bash
kubectl create secret generic azure-keyvault-config \
  -n flux-system \
  --from-literal=AZURE_TENANT_ID="<tenant-id>" \
  --from-literal=AZURE_KEYVAULT_URL="https://<vault-name>.vault.azure.net/"

kubectl create secret generic azure-secret-sp \
  -n external-secrets \
  --from-literal=ClientID="<app-client-id>" \
  --from-literal=ClientSecret="<app-client-secret>"
```

## Azure Key Vault Secrets

These values are pulled from Azure Key Vault by `ExternalSecret` resources.

| Key Vault secret | Kubernetes target | Used by | Where to get it |
| --- | --- | --- | --- |
| `ACR-LOGIN-SERVER` | `azdo-agent/acr-pull-secret`, `flux-system/azdo-image-repository` | ACR image pulls and Flux image repository substitution | Azure Container Registry overview page. Example: `myregistry.azurecr.io`. |
| `ACR-CLIENT-ID` | `azdo-agent/acr-pull-secret` | ACR pull authentication | Client ID of a service principal with `AcrPull` on the registry. |
| `ACR-CLIENT-SECRET` | `azdo-agent/acr-pull-secret` | ACR pull authentication | Client secret for the same `AcrPull` service principal. |
| `AZP-URL` | `azdo-agent/azdo-agent-credentials` | Azure DevOps agent and KEDA scaler auth | Azure DevOps organization URL, for example `https://dev.azure.com/<org>`. Use organization-level URL, not project URL. |
| `AZP-TOKEN` | `azdo-agent/azdo-agent-credentials` | Azure DevOps agent registration and KEDA queue checks | Azure DevOps PAT with `Agent Pools: Read & manage`. Store the plain PAT value. |
| `AZP-POOL` | `azdo-agent/azdo-agent-credentials`, `flux-system/azdo-image-repository` | Azure DevOps agent pool and Flux substitution | Azure DevOps pool name from Organization settings -> Agent pools. |
| `GRAFANA-ADMIN-USER` | `monitoring/grafana-admin-credentials` | Grafana admin login | Choose the admin username you want Grafana to use. |
| `GRAFANA-ADMIN-PASSWORD` | `monitoring/grafana-admin-credentials` | Grafana admin login | Generate a strong password and store it in Key Vault. |
| `TUNNEL-TOKEN` | `cloudflared/cloudflared-secret` | Cloudflare Tunnel | Cloudflare Zero Trust dashboard -> Networks -> Tunnels -> tunnel token. |
| `GIT-TOKEN` | `obsidian/git-credentials` | Obsidian Git backup/restore jobs | Git provider PAT with access to the Obsidian backup repository. |
| `SUBSCRIPTION-ID` | `velero/cloud-credentials`, `velero/velero-helm-values` | Velero Azure backup storage | Azure subscription ID containing the Velero backup resources. |
| `TENANT-ID` | `velero/cloud-credentials` | Velero Azure auth | Microsoft Entra tenant ID for the Velero service principal. |
| `VELERO-CLIENT-ID` | `velero/cloud-credentials` | Velero Azure auth | Client ID of the Velero service principal. |
| `VELERO-CLIENT-SECRET` | `velero/cloud-credentials` | Velero Azure auth | Client secret for the Velero service principal. |
| `VELERO-BACKUP-RESOURCE-GROUP` | `velero/cloud-credentials`, `velero/velero-helm-values` | Velero backup location | Azure resource group containing the Velero storage account. |
| `VELERO-STORAGE-ACCOUNT` | `velero/velero-helm-values` | Velero backup location | Azure Storage Account name used for Velero backups. |
| `VELERO-BLOB-CONTAINER` | `velero/velero-helm-values` | Velero backup location | Blob container name inside the Velero storage account. |

## Generated Kubernetes Secrets

External Secrets creates these Kubernetes Secrets from the Key Vault values:

| Kubernetes secret | Namespace | Created by |
| --- | --- | --- |
| `acr-pull-secret` | `azdo-agent` | `cluster/apps/azdo-agent/secrets/acr.yaml` |
| `azdo-agent-credentials` | `azdo-agent` | `cluster/apps/azdo-agent/secrets/azdo.yaml` |
| `azdo-image-repository` | `flux-system` | `cluster/infrastructure/external-secrets-resources/azdo-image-repository.yaml` |
| `cloudflared-secret` | `cloudflared` | `cluster/apps/cloudflared/secrets/tunnel.yaml` |
| `git-credentials` | `obsidian` | `cluster/apps/obsidian/secrets/git.yaml` |
| `grafana-admin-credentials` | `monitoring` | `cluster/infrastructure/monitoring/secrets/grafana.yaml` |
| `cloud-credentials` | `velero` | `cluster/infrastructure/velero/secrets/velero.yaml` |
| `velero-helm-values` | `velero` | `cluster/infrastructure/velero/secrets/velero.yaml` |

## Useful Checks

```bash
kubectl get externalsecret -A
kubectl describe clustersecretstore azure
kubectl get secret azure-keyvault-config -n flux-system
kubectl get secret azure-secret-sp -n external-secrets
```
