# 实验 03 — 部署 AKS 集群：Entra ID、Workload Identity 与 Helm

> ⏱ 约 75 分钟 · 预配一个符合登陆区规范的 AKS 集群，接入 Microsoft Entra ID + Azure RBAC，启用 Workload Identity 与 Key Vault CSI 驱动，然后阅读并离线渲染交付的 Helm Chart。真正的发布在实验 04。

🇬🇧 [English](./README.md)

## 场景故事

ZavaShop 九个服务中的八个已经跑在 Container Apps 上了。第九个 —— **编排器** —— 不行。

它是整个舰队的"店长"：长期驻留、多副本、由负载均衡器暴露、跨可用区分布，还持有一个绝不能出现在模板环境变量中的 Copilot 令牌。它需要一个具备 Azure 原生身份能力的 Kubernetes 运行时。

本实验的重点是 **理解 AKS 到足以运维它**，而不是 `kubectl apply`。你将：

1. 从第一分钟就带着登陆区管控项创建集群。
2. 证明人员用 **Microsoft Entra ID** 认证、用 **Azure RBAC** 授权，且从不接触管理员 kubeconfig。
3. 用 **Workload Identity** 给编排器 Pod 一个 Azure 身份 —— 集群里没有任何密钥。
4. 用 **Secrets Store CSI 驱动** 从 Key Vault 投射 `GITHUB-TOKEN`。
5. 阅读交付的 Helm Chart 并离线渲染，直到你能准确预测它创建的每一个对象。

结束时，集群已具备生产形态、Chart 已验证 —— 但什么都还没部署。这个刻意留下的空档就是实验 04。

---

## 本实验相关的 Microsoft Learn 知识

- [Azure Kubernetes 服务 (AKS) 概述](https://learn.microsoft.com/zh-cn/azure/aks/intro-kubernetes) —— AKS 替你管什么、什么仍归你管。
- [AKS 核心概念](https://learn.microsoft.com/zh-cn/azure/aks/core-aks-concepts) —— 控制平面、节点池、命名空间、工作负载。
- [AKS 体系结构指南](https://learn.microsoft.com/zh-cn/azure/architecture/reference-architectures/containers/aks-start-here) —— 当前的生产参考架构。
- [AKS 登陆区加速器](https://learn.microsoft.com/zh-cn/azure/cloud-adoption-framework/scenarios/app-platform/aks/landing-zone-accelerator) —— 本集群对应的设计领域。
- [AKS 的 Microsoft Entra ID 集成](https://learn.microsoft.com/zh-cn/azure/aks/enable-authentication-microsoft-entra-id) —— 不使用本地账户的集群认证。
- [将 Azure RBAC 用于 Kubernetes 授权](https://learn.microsoft.com/zh-cn/azure/aks/manage-azure-rbac) —— 用 Azure 角色分配实现 Kubernetes 授权。
- [AKS 工作负载标识](https://learn.microsoft.com/zh-cn/azure/aks/workload-identity-overview) —— 把 Kubernetes 服务账户联合到 UAMI。
- [AKS 上的 Secrets Store CSI 驱动](https://learn.microsoft.com/zh-cn/azure/aks/csi-secrets-store-driver) —— 把 Key Vault 机密挂载进 Pod。
- [Azure CNI Overlay 网络](https://learn.microsoft.com/zh-cn/azure/aks/azure-cni-overlay) —— 本实验使用的节省 IP 的网络插件。
- [AKS 的 Azure Policy](https://learn.microsoft.com/zh-cn/azure/aks/use-azure-policy) —— 集群护栏与合规报告。
- [Microsoft Defender for Containers](https://learn.microsoft.com/zh-cn/azure/defender-for-cloud/defender-for-containers-introduction) —— 集群运行时威胁检测。
- [容器见解](https://learn.microsoft.com/zh-cn/azure/azure-monitor/containers/container-insights-overview) —— 节点、Pod、容器遥测。
- [AKS Helm 快速入门](https://learn.microsoft.com/zh-cn/azure/aks/quickstart-helm) —— `infra/aks/helm/zavashop` 使用的 Chart 流程。
- [Pod 安全上下文](https://learn.microsoft.com/zh-cn/azure/aks/developer-best-practices-pod-security) —— 非 root、只读根文件系统的基线。

---

## 前置条件

```bash
cd /path/to/AKS-Lab-GitHubCopilot
source .env.lab

echo "$RG / $AKS / $ACR / $KV / $LAW_ID"
echo "$AKS_ADMINS_GROUP_ID"
```

实验 02 必须已完成：八个 Container App 在线且 `.env.fqdns` 存在。编排器镜像 `zavashop/orchestrator:<sha>` 已在 ACR 中。

---

## 第 A 部分 — 创建集群

### 步骤 1 — 敲命令之前先决定集群形态

下面命令中的每一个参数都在回答一个登陆区设计问题。

| 设计领域 | 决策 | 参数 |
|---|---|---|
| 计算 | 2 节点，`Standard_D4s_v6` —— 足够跑 2 个编排器副本加系统 Pod | `--node-count`、`--node-vm-size` |
| 身份（认证） | 人员用 Microsoft Entra ID 登录 | `--enable-aad` |
| 身份（授权） | Kubernetes RBAC 委派给 Azure 角色分配 | `--enable-azure-rbac` |
| 管理员访问 | 仅实验 01 创建的 Entra 组成员 | `--aad-admin-group-object-ids` |
| 工作负载身份 | Pod 通过 OIDC 颁发者联合到 UAMI | `--enable-oidc-issuer`、`--enable-workload-identity` |
| 机密 | Key Vault 经 CSI 投射 | `--enable-addons azure-keyvault-secrets-provider` |
| 治理 | Policy 附加组件提供护栏 | `--enable-addons azure-policy` |
| 监控 | 容器见解 → 实验 01 的工作区 | `--enable-addons monitoring`、`--workspace-resource-id` |
| 网络 | Azure CNI Overlay 节省 IP | `--network-plugin azure --network-plugin-mode overlay` |
| 注册表 | kubelet 用托管标识从 ACR 拉取 | `--attach-acr` |

### 步骤 2 — 创建集群

```bash
az aks create \
  -g $RG -n $AKS \
  --location $LOCATION \
  --node-count 2 \
  --node-vm-size Standard_D4s_v6 \
  --enable-aad \
  --enable-azure-rbac \
  --aad-admin-group-object-ids $AKS_ADMINS_GROUP_ID \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --enable-addons monitoring,azure-policy,azure-keyvault-secrets-provider \
  --enable-secret-rotation \
  --workspace-resource-id $LAW_ID \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --attach-acr $ACR \
  --generate-ssh-keys \
  --tags project=zavashop lab=03
```

需要 5–10 分钟。等待期间可以阅读下面的第 B 部分。

如果订阅没有 `Standard_D4s_v6` 配额，可替换为 `Standard_D4s_v5` 或 `Standard_D2s_v5`（至少 2 节点）。

### 步骤 3 — 启用 Defender 配置文件

```bash
az aks update \
  -g $RG -n $AKS \
  --enable-defender \
  --defender-config \
      logAnalyticsWorkspaceResourceId=$LAW_ID
```

### 步骤 4 — 把集群信息写入 `.env.lab`

```bash
export AKS_ID=$(az aks show -g $RG -n $AKS --query id -o tsv)
export AKS_OIDC=$(az aks show -g $RG -n $AKS \
  --query oidcIssuerProfile.issuerUrl -o tsv)

cat >> .env.lab <<EOF
AKS_ID=$AKS_ID
AKS_OIDC=$AKS_OIDC
EOF

echo "$AKS_OIDC"
```

---

## 第 B 部分 — 身份：人和 Pod 是怎么进来的

两条完全独立的路径。把它们混为一谈是 AKS 上最常见的错误。

```
人员                                     Pod
────                                     ───
Entra ID 登录                            ServiceAccount 令牌
   │                                        │
   ▼                                        ▼
集群上的 Azure RBAC 角色                  UAMI 上的联合凭据
   │                                        │
   ▼                                        ▼
kubectl（无本地管理员账户）               DefaultAzureCredential → Key Vault
```

### 步骤 5 — 通过 Azure RBAC 授予人员访问权限

```bash
az role assignment create \
  --assignee $AKS_ADMINS_GROUP_ID \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --scope $AKS_ID
```

给 UAMI 分配 CI 所需的最小权限 —— 仅获取 kubeconfig：

```bash
az role assignment create \
  --assignee-object-id $UAMI_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $AKS_ID
```

> CI 绝不获得 `RBAC Cluster Admin`。实验 04 的工作流只需要在一个命名空间里执行 `helm upgrade`。

### 步骤 6 — 用正确的方式获取 kubeconfig

```bash
az aks get-credentials -g $RG -n $AKS --overwrite-existing
kubectl get nodes -o wide
```

第一条 `kubectl` 命令会触发 Entra ID 设备代码登录。这就是认证已经联合化的证据。

```bash
kubectl auth can-i create deployment -n zavashop
kubectl auth can-i '*' '*' --all-namespaces
```

**绝不要**执行 `az aks get-credentials --admin`。它完全绕过 Entra ID，实验 04 的门禁会把它判为失败。

### 步骤 7 — 创建命名空间和服务账户

```bash
kubectl create namespace zavashop \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl label namespace zavashop \
  project=zavashop lab=03 --overwrite
```

Helm Chart 自己会创建 `orchestrator-sa`（见 `templates/serviceaccount.yaml`），但联合凭据必须在任何 Pod 启动之前就存在。

### 步骤 8 — 把服务账户联合到 UAMI

```bash
az identity federated-credential create \
  --name fc-orchestrator \
  --identity-name $UAMI \
  --resource-group $RG \
  --issuer "$AKS_OIDC" \
  --subject system:serviceaccount:zavashop:orchestrator-sa \
  --audiences api://AzureADTokenExchange
```

subject 的格式是 `system:serviceaccount:<命名空间>:<服务账户>`。如果 Chart 的命名空间或 SA 名称变了，这里也必须改。

```bash
az identity federated-credential list \
  --identity-name $UAMI -g $RG \
  --query "[].{name:name, subject:subject}" -o table
```

应该看到两行：`gha-aks-lab-main`（实验 01）和 `fc-orchestrator`。

### 步骤 9 — 验证 CSI 驱动正在运行

```bash
kubectl get pods -n kube-system \
  -l app=secrets-store-csi-driver

kubectl get pods -n kube-system \
  -l app=secrets-store-provider-azure
```

如果创建时没有启用该附加组件：

```bash
az aks enable-addons \
  -g $RG -n $AKS \
  --addons azure-keyvault-secrets-provider \
  --enable-secret-rotation
```

### 步骤 10 — 端到端验证身份链路

在使用 Helm 之前先测试，这样失败点毫无歧义。

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orchestrator-sa
  namespace: zavashop
  annotations:
    azure.workload.identity/client-id: "$UAMI_CLIENT_ID"
EOF
```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: wif-smoke
  namespace: zavashop
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: "$UAMI_CLIENT_ID"
    keyvaultName: "$KV"
    tenantId: "$AZURE_TENANT_ID"
    objects: |
      array:
        - |
          objectName: GITHUB-TOKEN
          objectType: secret
EOF
```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: wif-smoke
  namespace: zavashop
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: orchestrator-sa
  restartPolicy: Never
  containers:
    - name: probe
      image: mcr.microsoft.com/azure-cli:latest
      command: ["sh", "-c", "ls -l /mnt/secrets-store && sleep 30"]
      volumeMounts:
        - name: secrets
          mountPath: /mnt/secrets-store
          readOnly: true
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: wif-smoke
EOF
```

```bash
kubectl -n zavashop wait --for=condition=Ready pod/wif-smoke --timeout=120s
kubectl -n zavashop logs wif-smoke
```

目录列表中出现 `GITHUB-TOKEN`，说明 Workload Identity、联合凭据、Key Vault RBAC 授权和 CSI 驱动全部正确。

清理冒烟对象 —— Chart 会自带这些资源：

```bash
kubectl -n zavashop delete pod wif-smoke
kubectl -n zavashop delete secretproviderclass wif-smoke
```

> 如果 Pod 一直是 `ContainerCreating`，执行 `kubectl -n zavashop describe pod wif-smoke`。CSI 的错误信息会精确指出失败环节。

---

## 第 C 部分 — Helm：先读懂 Chart 再运行

### 步骤 11 — Chart 结构

```bash
find infra/aks/helm/zavashop -type f | sort
```

```
infra/aks/helm/zavashop/Chart.yaml
infra/aks/helm/zavashop/values.yaml
infra/aks/helm/zavashop/templates/_helpers.tpl
infra/aks/helm/zavashop/templates/deployment.yaml
infra/aks/helm/zavashop/templates/secretproviderclass.yaml
infra/aks/helm/zavashop/templates/service.yaml
infra/aks/helm/zavashop/templates/serviceaccount.yaml
```

四个 Kubernetes 对象，每个对应一个关注点：

| 模板 | 对象 | 演示的 AKS 概念 |
|---|---|---|
| `serviceaccount.yaml` | `ServiceAccount` | Workload Identity 注解 `azure.workload.identity/client-id` |
| `secretproviderclass.yaml` | `SecretProviderClass` | Key Vault → CSI 投射 `GITHUB-TOKEN` |
| `deployment.yaml` | `Deployment` | 副本、探针、安全上下文、拓扑分布、CSI 卷 |
| `service.yaml` | `Service` | `LoadBalancer` 把 80 端口映射到容器 8000 |

### 步骤 12 — 必填 values

```bash
cat infra/aks/helm/zavashop/values.yaml
```

有六项没有默认值，**必须**提供：

| Value | 来源 |
|---|---|
| `image.repository` | `$ACR.azurecr.io/zavashop/orchestrator` |
| `image.tag` | 实验 02 的 git SHA |
| `workloadIdentity.clientId` | `$UAMI_CLIENT_ID` |
| `keyVault.name` | `$KV` |
| `tenantId` | `$AZURE_TENANT_ID` |
| `a2a.{inventory,supplier,logistics,pricing}` | 实验 02 的 `.env.fqdns` |

Chart 使用了 Helm 的 `required` 函数，所以缺值会在渲染阶段失败，而不是部署出一个坏 Pod。

### 步骤 13 — 阅读 Deployment 的安全姿态

```bash
sed -n '15,50p' infra/aks/helm/zavashop/templates/deployment.yaml
```

需要识别的五项加固决策：

1. Pod 标签 `azure.workload.identity/use: "true"` —— 令牌注入的必要条件。
2. `podSecurityContext`：`runAsNonRoot`、uid/gid/fsGroup `10001` —— 与 `Dockerfile.base` 一致。
3. `securityContext`：`allowPrivilegeEscalation: false`、`readOnlyRootFilesystem: true`、丢弃全部 capabilities。
4. 基于 `topology.kubernetes.io/zone` 的 `topologySpreadConstraints`，`maxSkew: 1`。
5. 容器在启动时从 CSI 挂载点读取令牌，因此该值绝不出现在 `kubectl get pod -o yaml` 中。

> `readOnlyRootFilesystem: true` 正是 Chart 在 `/home/zava/.cache` 挂载 `emptyDir` 的原因 —— Copilot SDK 需要一个可写的缓存路径。

### 步骤 14 — Lint

```bash
helm lint infra/aks/helm/zavashop \
  --set image.repository=example.azurecr.io/zavashop/orchestrator \
  --set image.tag=abc1234 \
  --set workloadIdentity.clientId=00000000-0000-0000-0000-000000000000 \
  --set keyVault.name=kv-example \
  --set tenantId=00000000-0000-0000-0000-000000000000 \
  --set a2a.inventory=https://example/invoke \
  --set a2a.supplier=https://example/invoke \
  --set a2a.logistics=https://example/invoke \
  --set a2a.pricing=https://example/invoke
```

期望：`1 chart(s) linted, 0 chart(s) failed`。

### 步骤 15 — 用真实值渲染

```bash
source .env.fqdns
export GIT_SHA=$(git rev-parse --short HEAD)

helm template zavashop infra/aks/helm/zavashop \
  -n zavashop \
  --set image.repository="$ACR.azurecr.io/zavashop/orchestrator" \
  --set image.tag="$GIT_SHA" \
  --set workloadIdentity.clientId="$UAMI_CLIENT_ID" \
  --set keyVault.name="$KV" \
  --set tenantId="$AZURE_TENANT_ID" \
  --set a2a.inventory="$ZAVA_INVENTORY_URL" \
  --set a2a.supplier="$ZAVA_SUPPLIER_URL" \
  --set a2a.logistics="$ZAVA_LOGISTICS_URL" \
  --set a2a.pricing="$ZAVA_PRICING_URL" \
  > /tmp/zavashop-rendered.yaml
```

检查将要创建的对象：

```bash
grep '^kind:' /tmp/zavashop-rendered.yaml | sort -u
```

```
kind: Deployment
kind: SecretProviderClass
kind: Service
kind: ServiceAccount
```

断言实验 04 门禁依赖的三个属性：

```bash
grep -c 'image: ' /tmp/zavashop-rendered.yaml          # 1
grep 'image: ' /tmp/zavashop-rendered.yaml             # 以 :$GIT_SHA 结尾
grep -i 'latest' /tmp/zavashop-rendered.yaml           # 无输出
```

确认没有机密值泄漏进清单：

```bash
grep -i 'GITHUB_TOKEN' /tmp/zavashop-rendered.yaml
```

唯一匹配应该是容器参数中的 `cat /mnt/secrets-store/GITHUB-TOKEN` 命令 —— 绝不是字面令牌。

### 步骤 16 — 服务端 dry run

这会针对活的 API 服务器做校验，包括准入控制和 Azure Policy 附加组件，但不创建任何东西。

```bash
helm template zavashop infra/aks/helm/zavashop \
  -n zavashop \
  --set image.repository="$ACR.azurecr.io/zavashop/orchestrator" \
  --set image.tag="$GIT_SHA" \
  --set workloadIdentity.clientId="$UAMI_CLIENT_ID" \
  --set keyVault.name="$KV" \
  --set tenantId="$AZURE_TENANT_ID" \
  --set a2a.inventory="$ZAVA_INVENTORY_URL" \
  --set a2a.supplier="$ZAVA_SUPPLIER_URL" \
  --set a2a.logistics="$ZAVA_LOGISTICS_URL" \
  --set a2a.pricing="$ZAVA_PRICING_URL" \
  | kubectl apply -f - --dry-run=server
```

每一行都必须以 `(server dry run)` 结尾。到此为止 —— 实验 04 才执行真正的 `helm upgrade --install`。

---

## 第 D 部分 — 验证登陆区管控项

```bash
az aks show -g $RG -n $AKS --query aadProfile -o yaml

az aks show -g $RG -n $AKS \
  --query aadProfile.enableAzureRBAC -o tsv              # true

az aks show -g $RG -n $AKS \
  --query addonProfiles.azurepolicy.enabled -o tsv       # true

az aks show -g $RG -n $AKS \
  --query securityProfile.defender.securityMonitoring.enabled -o tsv

az aks show -g $RG -n $AKS \
  --query oidcIssuerProfile.enabled -o tsv               # true

az aks show -g $RG -n $AKS \
  --query securityProfile.workloadIdentity.enabled -o tsv

az aks show -g $RG -n $AKS \
  --query addonProfiles.omsagent.config.logAnalyticsWorkspaceResourceID \
  -o tsv
```

确认 Azure Policy 附加组件确实在评估：

```bash
kubectl get pods -n kube-system -l app=azure-policy
kubectl get constrainttemplates 2>/dev/null | head
```

确认容器见解正在接收数据（创建后约 10 分钟）：

```bash
kubectl get pods -n kube-system -l dsName=ama-logs-ds
```

完整的身份 + 机密交接清单见 [infra/aks/wif/README.zh.md](../../infra/aks/wif/README.zh.md)。

---

## ✅ 达成以下条件即完成实验 03

- [ ] 经 Entra ID 登录后 `kubectl get nodes` 返回 2 个 `Ready` 节点。
- [ ] `aadProfile.enableAzureRBAC` 为 `true`。
- [ ] Entra 组拥有 `Azure Kubernetes Service RBAC Cluster Admin`；UAMI 只有 `Cluster User`。
- [ ] `oidcIssuerProfile.enabled` 与 `securityProfile.workloadIdentity.enabled` 均为 `true`。
- [ ] Azure Policy 附加组件与 Defender 安全监控均为 `true`。
- [ ] 容器见解指向实验 01 的工作区。
- [ ] 存在 `fc-orchestrator` 联合凭据，subject 为 `system:serviceaccount:zavashop:orchestrator-sa`。
- [ ] `wif-smoke` Pod 从 CSI 挂载点列出了 `GITHUB-TOKEN`，并已删除。
- [ ] `helm lint` 返回 `0 chart(s) failed`。
- [ ] `helm template` 恰好渲染出 `Deployment`、`Service`、`ServiceAccount`、`SecretProviderClass`。
- [ ] `kubectl apply --dry-run=server` 通过。
- [ ] 目前还没有真正部署任何东西。

下一步 → [实验 04 — 部署到生产](../lab-04-deploy-production/README.zh.md)

---

## 故障排查

| 现象 | 处理 |
|---|---|
| `az aks create` 配额不足 | 试试 `Standard_D4s_v5`/`Standard_D2s_v5`，或换一个区域。 |
| `az aks create` 超过 15 分钟无响应 | 另开一个 shell 查看 `az aks show --query provisioningState`；如果是 `Failed` 则删除重建。 |
| `kubectl` 返回 `Unauthorized` | 重新执行 `az aks get-credentials --overwrite-existing`；确认你在 Entra 管理员组内。 |
| `kubectl` 返回 `Forbidden` | 说明 Azure RBAC 正在生效。确认组在 `$AKS_ID` 上的角色分配并等待生效。 |
| `wif-smoke` 卡在 `ContainerCreating` | `kubectl -n zavashop describe pod wif-smoke` —— CSI 事件会指出失败原因。 |
| CSI 报 `failed to get objectType:secret` | UAMI 缺少 `Key Vault Secrets User`，或保管库名/租户 ID 有误。 |
| CSI 报 `no matching federated identity record` | subject 不匹配。必须是 `system:serviceaccount:zavashop:orchestrator-sa`。 |
| `helm lint` 在 `required` 处失败 | 你漏掉了步骤 12 中六个必填值之一。 |
| `--dry-run=server` 被策略拒绝 | Azure Policy 正在强制某个约束。请阅读消息并调整集群策略，而不是 Chart 的安全上下文。 |
