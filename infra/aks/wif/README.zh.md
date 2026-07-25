# AKS Workload Identity 与 CSI 交接清单

🇬🇧 [English](./README.md)

本文档记录 ZavaShop 编排器在 AKS 上的身份与机密投射检查项。它在 [实验 03](../../../labs/lab-03-deploy-aks-cluster/README.zh.md) 中搭建，并在 [实验 04](../../../labs/lab-04-deploy-production/README.zh.md) 中作为发布门禁重新校验。

## 1. 重新联合 UAMI

实验 01 创建 UAMI；实验 03 创建 AKS OIDC 颁发者。重建或验证编排器服务账户的联合凭据：

```bash
source .env.lab

az identity federated-credential create \
  --name fc-orchestrator \
  --identity-name "$UAMI" \
  --resource-group "$RG" \
  --issuer "$AKS_OIDC" \
  --subject system:serviceaccount:zavashop:orchestrator-sa \
  --audience api://AzureADTokenExchange
```

## 2. 安装 Secrets Store CSI 驱动

按集群基线启用 AKS 附加组件，或安装驱动加 Azure 提供程序：

```bash
az aks enable-addons \
  -g "$RG" -n "$AKS" \
  --addons azure-keyvault-secrets-provider

kubectl get pods -n kube-system -l app=secrets-store-csi-driver
```

## 3. 在 Key Vault 中写入 `GITHUB-TOKEN`

实验中手工写入一个 GitHub Copilot 可用的令牌。**不要**提交到仓库。

```bash
az keyvault secret set \
  --vault-name "$KV" \
  --name GITHUB-TOKEN \
  --value "$GITHUB_TOKEN"
```

Helm Chart 通过 `SecretProviderClass` 把它投射为 Kubernetes 机密 `github-token`，再映射进容器的 `GITHUB_TOKEN`。

## 4. 校验 Entra ID 与 Azure RBAC

人员运维必须经 Microsoft Entra ID + Azure RBAC 认证，而不是本地管理员 kubeconfig。

```bash
az aks show -g "$RG" -n "$AKS" --query aadProfile -o yaml

az role assignment list --scope "$AKS_ID" \
  --query "[].{role:roleDefinitionName, principal:principalName}" \
  -o table
```

CI 中不要申请本地管理员 kubeconfig 凭据。

## 5. 校验 Defender、Policy 与监控

```bash
az aks show -g "$RG" -n "$AKS" \
  --query addonProfiles.azurepolicy.enabled -o tsv

az aks show -g "$RG" -n "$AKS" \
  --query securityProfile.defender.securityMonitoring.enabled -o tsv

az security pricing show -n Containers --query pricingTier -o tsv
az security pricing show -n KeyVaults --query pricingTier -o tsv

az aks show -g "$RG" -n "$AKS" \
  --query addonProfiles.omsagent.config.logAnalyticsWorkspaceResourceID -o tsv
```

## 6. 生产环境的 GitHub App 令牌代理（占位说明）

实验使用托管在 Key Vault 中的 `GITHUB-TOKEN`。生产环境应替换为 GitHub App 加 OIDC 支撑的令牌代理，为编排器 Pod 颁发短期 Copilot SDK 凭据。CSI 与 Workload Identity 模式保持不变，只是把凭据来源从静态机密换成代理端点。
