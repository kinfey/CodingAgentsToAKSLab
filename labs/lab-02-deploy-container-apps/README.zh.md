# 实验 02 — 部署到 Container Apps：容器化舰队并用 Bicep 发布

> ⏱ **快速通道约 30 分钟 · 完整实验约 60 分钟** · 从交付源码构建 10 个容器镜像并推送到 ACR，然后用一个可复用的 **Bicep** 模块把 4 个 MCP 服务器 + 4 个专家 Agent 部署到 **Azure Container Apps**。

🇬🇧 [English](./README.md)

### ⏱ 想在 2 小时内跑完整个系列？

| | 步骤 | 预算 |
|---|---|---|
| **核心** | 2、3、4、5、6、10、11、12 | 约 30 分钟 |
| *可选* | 1 镜像结构 · 7 Bicep 讲解 · 8 手动部署 · 9 what-if · 13 机密验证 · 14 伸缩行为 | 约 30 分钟 |

> 步骤 8 和 9 是教学性绕路 —— 步骤 10（`deploy.sh`）本来就会部署全部八个应用。走快速通道跳过它们不会少任何基础设施。

## 场景故事

实验 01 给了你一套基础设施和一个可运行的本地 Compose 舰队。但 Compose 不是部署目标 —— ZavaShop 需要系统中"突发型"的部分在凌晨 3 点几乎零成本，同时能扛住下午 5 点的补货高峰。

九个服务中有八个符合这个特征：四个 **专家 Agent**（inventory、supplier、logistics、pricing）和四个 **MCP 工具服务器**。它们无状态、由 HTTP 触发、一天中大部分时间空闲。Azure Container Apps 正是合适的归宿：无需运维集群、KEDA HTTP 弹性伸缩、托管标识、Key Vault 支撑的机密。

编排器不同 —— 它长期驻留并由负载均衡器暴露。那是实验 03 和实验 04 的内容。

本实验结束时，ZavaShop 将拥有 **8 个在线的 Container App**，它们响应 `/healthz`、`/readyz` 和 `/invoke`，Copilot 令牌由 Key Vault 交付，且从不写入 Bicep 参数。

---

## 本实验相关的 Microsoft Learn 知识

- [Azure Container Apps 概述](https://learn.microsoft.com/zh-cn/azure/container-apps/overview) —— 8 个突发型服务的运行模型。
- [使用 Container Apps 和 Bicep 构建微服务](https://learn.microsoft.com/zh-cn/azure/container-apps/microservices-bicep) —— 被复用 8 次的参数化模块模式。
- [ACR 任务](https://learn.microsoft.com/zh-cn/azure/container-registry/container-registry-tasks-overview) —— `az acr build` 提供无守护进程的原生 `linux/amd64` 构建。
- [Azure Container Apps 中的托管标识](https://learn.microsoft.com/zh-cn/azure/container-apps/managed-identity) —— 无需注册表密码即可拉取镜像。
- [在 Container Apps 中引用 Key Vault 机密](https://learn.microsoft.com/zh-cn/azure/container-apps/manage-secrets) —— `keyVaultUrl` + `identity` 的机密写法。
- [Container Apps 运行状况探针](https://learn.microsoft.com/zh-cn/azure/container-apps/health-probes) —— 存活探针 `/healthz`，就绪探针 `/readyz`。
- [在 Container Apps 中设置缩放规则](https://learn.microsoft.com/zh-cn/azure/container-apps/scale-app) —— HTTP 并发规则。
- [Bicep `what-if` 部署](https://learn.microsoft.com/zh-cn/azure/azure-resource-manager/bicep/deploy-what-if) —— 应用前先预览。
- [Bicep 模块](https://learn.microsoft.com/zh-cn/azure/azure-resource-manager/bicep/modules) —— 为什么一个 `agent.bicep` 能服务 8 个应用。

---

## 你要部署什么

| # | Container App | 镜像 | 端口 | 入口 | 所需环境变量 |
|---|---|---|---|---|---|
| 1 | `inventory-mcp` | `zavashop/inventory-mcp` | 8080 | external | — |
| 2 | `supplier-mcp` | `zavashop/supplier-mcp` | 8080 | external | — |
| 3 | `shipping-mcp` | `zavashop/shipping-mcp` | 8080 | external | — |
| 4 | `pricing-mcp` | `zavashop/pricing-mcp` | 8080 | external | — |
| 5 | `inventory` | `zavashop/inventory` | 8000 | external | `ZAVA_INVENTORY_MCP_URL` |
| 6 | `supplier` | `zavashop/supplier` | 8000 | external | `ZAVA_SUPPLIER_MCP_URL` |
| 7 | `logistics` | `zavashop/logistics` | 8000 | external | `ZAVA_SHIPPING_MCP_URL` |
| 8 | `pricing` | `zavashop/pricing` | 8000 | external | `ZAVA_PRICING_MCP_URL` |

顺序很重要：先部署 MCP 服务器，因为专家 Agent 需要它们的 FQDN。

---

## 前置条件

```bash
cd /path/to/AKS-Lab-GitHubCopilot
source .env.lab

echo "$RG / $ACR / $CAE / $KV"
az account show --query name -o tsv
```

如果 `.env.lab` 不存在，请先回到 [实验 01](../lab-01-deployment-foundation/README.zh.md)。

---

## 步骤 1 — 构建前先理解镜像结构 *（可选）*

交付的应用会构建**一个共享基础镜像**，外加每个服务一个轻量镜像。先阅读两个 Dockerfile。

```bash
sed -n '1,60p' src/Dockerfile.base
sed -n '1,40p' src/agents/inventory/Dockerfile
sed -n '1,40p' src/mcp_servers/inventory/Dockerfile
```

必须带入 ACR 的关键事实：

| 事实 | 影响 |
|---|---|
| `src/Dockerfile.base` 安装所有 Python 依赖 | 服务镜像只复制源码 —— 首次之后的构建很快。 |
| Agent Dockerfile 接受 `BASE_IMAGE` 构建参数 | 必须传入 **ACR 全限定** 的基础镜像标签，而不是本地标签。 |
| 运行用户是 `zava`，uid `10001`，非 root | 与实验 03 的 AKS `podSecurityContext` 一致。 |
| Agent 监听 `8000`，MCP 服务器监听 `8080` | 决定 Bicep 中的 `targetPort`。 |

本实验系列所有镜像的命名约定：

```
$ACR.azurecr.io/zavashop/<service>:<短 git SHA>
```

**绝不使用 `:latest`。** 实验 04 的部署门禁会在 `infra/` 和工作流中 grep 该字符串，一旦出现就构建失败。

---

## 步骤 2 — 固定部署 SHA

本实验中构建和部署的一切都打上同一个不可变标签。

```bash
git status --porcelain      # 必须为空
export GIT_SHA=$(git rev-parse --short HEAD)
echo "deploying $GIT_SHA"
```

如果 `git status` 非空，请先提交或 stash。工作区不干净意味着标签无法标识正在运行的内容。

---

## 步骤 3 — 用 ACR Tasks 构建基础镜像

`az acr build` 会把构建上下文上传到 Azure 并在云端构建。无需本地 Docker 守护进程，Apple Silicon 上也不会出现架构不匹配。

```bash
az acr build -r $ACR \
  --platform linux/amd64 \
  -t zavashop/base:$GIT_SHA \
  -f src/Dockerfile.base .
```

验证：

```bash
az acr repository show-tags -n $ACR \
  --repository zavashop/base -o tsv
```

---

## 步骤 4 — 构建四个 MCP 服务器镜像

MCP 镜像不依赖基础镜像。

```bash
for service in inventory supplier shipping pricing; do
  az acr build -r $ACR \
    --platform linux/amd64 \
    -t zavashop/${service}-mcp:$GIT_SHA \
    -f src/mcp_servers/${service}/Dockerfile .
done
```

---

## 步骤 5 — 构建五个 Agent 镜像

Agent 镜像使用 ACR 全限定的基础镜像。

```bash
export ACR_LOGIN_SERVER="${ACR}.azurecr.io"

for service in orchestrator inventory supplier logistics pricing; do
  az acr build -r $ACR \
    --platform linux/amd64 \
    -t zavashop/${service}:$GIT_SHA \
    --build-arg BASE_IMAGE="$ACR_LOGIN_SERVER/zavashop/base:$GIT_SHA" \
    -f src/agents/${service}/Dockerfile .
done
```

> 编排器镜像虽然在实验 04 才部署，但在这里一并构建。一个 SHA、一轮构建、覆盖两个平面。

### 验收

```bash
az acr repository list -n $ACR -o tsv | sort
```

期望恰好是这十个仓库：

```
zavashop/base
zavashop/inventory
zavashop/inventory-mcp
zavashop/logistics
zavashop/orchestrator
zavashop/pricing
zavashop/pricing-mcp
zavashop/shipping-mcp
zavashop/supplier
zavashop/supplier-mcp
```

---

## 步骤 6 — 创建 Container Apps 环境

```bash
az containerapp env create \
  -n $CAE -g $RG -l $LOCATION \
  --logs-destination log-analytics \
  --logs-workspace-id $(az monitor log-analytics workspace show \
      -g $RG -n $LAW --query customerId -o tsv) \
  --logs-workspace-key $(az monitor log-analytics workspace \
      get-shared-keys -g $RG -n $LAW --query primarySharedKey -o tsv) \
  --tags project=zavashop lab=02
```

```bash
export CAE_ID=$(az containerapp env show -g $RG -n $CAE \
  --query id -o tsv)
echo "CAE_ID=$CAE_ID" >> .env.lab
```

等待就绪：

```bash
az containerapp env show -g $RG -n $CAE \
  --query properties.provisioningState -o tsv   # Succeeded
```

> 想在临时实验中避免日志摄取成本？把三个 `--logs-*` 参数替换为 `--logs-destination none`。其余完全不变，但 `az containerapp logs show` 将没有输出。

---

## 步骤 7 — 阅读 Bicep 模块 *（可选）*

打开 `infra/aca/agent.bicep`。它会以不同参数被部署 **八次**。逐一理解它固化的五件事：

```bash
sed -n '1,40p' infra/aca/agent.bicep
```

| 代码块 | 作用 | 为什么这样写 |
|---|---|---|
| `identity` | 以 `UserAssigned` 方式挂载实验 01 的 UAMI | 一个身份同时用于 ACR 拉取 *和* Key Vault 读取 |
| `configuration.secrets` | `keyVaultUrl: https://<kv>/secrets/GITHUB-TOKEN`，配合 `identity: uamiId` | 令牌永不作为 Bicep 参数，永不出现在部署日志中 |
| `configuration.registries` | 只有 `server` + `identity`（无密码） | ACR 管理员账户保持禁用 |
| `containers[0].probes` | 存活 `/healthz`，就绪 `/readyz` | 交付的 `make_app()` 已经暴露这两个端点 |
| `template.scale` | `minReplicas` 参数、`maxReplicas: 10`、HTTP 并发 `30` | 来自 Learn 文档的 KEDA HTTP 规则 |

每个应用需要你设置的两个参数：

- `targetPort` —— MCP 服务器为 `8080`，Agent 为 `8000`。
- `mcpEndpoints` —— 额外环境变量对象，例如 `{ "ZAVA_INVENTORY_MCP_URL": "https://.../mcp" }`。

部署任何东西之前先验证模板能编译：

```bash
az bicep build --file infra/aca/agent.bicep --stdout > /dev/null \
  && echo "bicep ok"
```

> **本实验为什么用 `minReplicas: 1`？** 缩容到零是 ACA 的杀手锏，但冷启动加上一次 Copilot SDK 调用可能超出实验 04 的冒烟测试预算。实验固定 `minReplicas=1` 以保证演示稳定。生产环境在测量冷启动预算后应设为 `0`。

---

## 步骤 8 — 手动部署第一个 MCP 服务器 *（可选）*

手动做第一个，以便清楚地看到模块产出了什么。

```bash
az deployment group create \
  -g $RG \
  -n aca-inventory-mcp \
  -f infra/aca/agent.bicep \
  -p name=inventory-mcp \
  -p envId="$CAE_ID" \
  -p image="$ACR_LOGIN_SERVER/zavashop/inventory-mcp:$GIT_SHA" \
  -p registryServer="$ACR_LOGIN_SERVER" \
  -p uamiId="$UAMI_RESOURCE_ID" \
  -p uamiClientId="$UAMI_CLIENT_ID" \
  -p keyVaultName="$KV" \
  -p targetPort=8080 \
  -p exposeIngress=true \
  -p minReplicas=1 \
  -o none
```

检查：

```bash
export INVENTORY_MCP_FQDN=$(az containerapp show \
  -g $RG -n inventory-mcp \
  --query properties.configuration.ingress.fqdn -o tsv)

curl -fsS "https://$INVENTORY_MCP_FQDN/healthz"
curl -fsS "https://$INVENTORY_MCP_FQDN/readyz"
```

两者都必须返回 `{"status":...}`。如果应用卡住，查看日志：

```bash
az containerapp logs show -g $RG -n inventory-mcp --tail 50
```

---

## 步骤 9 — 用 `what-if` 预览 *（可选）*

在部署剩余七个之前，先预览一个，养成习惯：

```bash
az deployment group what-if \
  -g $RG \
  -f infra/aca/agent.bicep \
  -p name=supplier-mcp \
  -p envId="$CAE_ID" \
  -p image="$ACR_LOGIN_SERVER/zavashop/supplier-mcp:$GIT_SHA" \
  -p registryServer="$ACR_LOGIN_SERVER" \
  -p uamiId="$UAMI_RESOURCE_ID" \
  -p uamiClientId="$UAMI_CLIENT_ID" \
  -p keyVaultName="$KV" \
  -p targetPort=8080 \
  -p exposeIngress=true \
  -p minReplicas=1
```

阅读 `+ Create` 块，确认没有打印任何机密值 —— 只有 `keyVaultUrl` 引用。

---

## 步骤 10 — 用 `deploy.sh` 部署全部八个

`infra/aca/deploy.sh` 会按依赖顺序执行八次同样的 `az deployment group create`，并把所有产生的 URL 写入 `.env.fqdns`。

```bash
sed -n '1,40p' infra/aca/deploy.sh
```

它做了什么：

1. source `.env.lab`，要求存在 `RG`、`ACR`、`CAE`、`KV`、`UAMI_*`。
2. 从 `DEPLOY_SHA`、`GIT_SHA` 或 `git rev-parse` 解析 `GIT_SHA`。
3. 部署 `inventory-mcp`、`supplier-mcp`、`shipping-mcp`、`pricing-mcp`。
4. 回读每个 MCP 的 FQDN 并拼出四个 `.../mcp` URL。
5. 部署四个专家 Agent，通过 `mcpEndpoints` 注入各自的 MCP URL。
6. 写出 `.env.fqdns`，其中的 `ZAVA_*_URL` 供实验 04 的 Helm values 使用。

先做语法检查，再运行：

```bash
bash -n infra/aca/deploy.sh && echo "shell syntax ok"

DEPLOY_SHA=$GIT_SHA bash infra/aca/deploy.sh
```

重复运行是安全的 —— 每次部署都是幂等的 ARM 更新。

---

## 步骤 11 — 验证八个应用

```bash
az containerapp list -g $RG -o table
```

八个都必须显示 `Succeeded`。

```bash
cat .env.fqdns
```

应包含 8 行 `ZAVA_*_URL` 以及 4 行 `ZAVA_*_MCP_URL`。

探测每个服务：

```bash
source .env.fqdns

for app in inventory-mcp supplier-mcp shipping-mcp pricing-mcp \
           inventory supplier logistics pricing; do
  fqdn=$(az containerapp show -g $RG -n $app \
    --query properties.configuration.ingress.fqdn -o tsv)
  printf '%-16s ' "$app"
  curl -fsS -o /dev/null -w '%{http_code}\n' "https://$fqdn/healthz"
done
```

八个 `200`。

---

## 步骤 12 — 端到端调用一个专家 Agent

这验证了完整链路：ACA 入口 → Agent → Copilot SDK（`gpt-5.5`）→ 通过 HTTP 调用 MCP 服务器。

```bash
INVENTORY_FQDN=$(az containerapp show -g $RG -n inventory \
  --query properties.configuration.ingress.fqdn -o tsv)

curl -fsS -X POST "https://$INVENTORY_FQDN/invoke" \
  -H 'content-type: application/json' \
  -d '{
        "sku": "ZS-1042",
        "store_id": "store-101",
        "goal": "Assess stock-out risk and recommend an action."
      }'
```

应得到一个引用 `ZS-1042` 库存水平的 JSON 响应。首次调用可能需要 20–40 秒等待 Copilot SDK 预热。

对 `supplier`、`logistics`、`pricing` 重复执行。

---

## 步骤 13 — 确认机密从未出现在参数中 *（可选）*

```bash
az containerapp show -g $RG -n inventory \
  --query properties.configuration.secrets -o json
```

你应该看到 `keyVaultUrl` 和 `identity`，且**没有** `value` 字段。

```bash
az deployment group show -g $RG -n aca-inventory-mcp \
  --query properties.parameters -o json | grep -i token
```

没有输出说明令牌从未作为参数传递过。

---

## 步骤 14 — 理解伸缩行为 *（可选）*

```bash
az containerapp show -g $RG -n pricing \
  --query properties.template.scale -o json
```

```json
{
  "minReplicas": 1,
  "maxReplicas": 10,
  "rules": [
    { "name": "http-concurrency",
      "http": { "metadata": { "concurrentRequests": "30" } } }
  ]
}
```

只对一个应用尝试更接近生产的配置：

```bash
az containerapp update -g $RG -n pricing --min-replicas 0
az containerapp revision list -g $RG -n pricing -o table
```

然后改回来，保证实验 04 的评测足够快：

```bash
az containerapp update -g $RG -n pricing --min-replicas 1
```

---

## ✅ 达成以下条件即完成实验 02

- [ ] `az acr repository list -n $ACR` 显示全部十个 `zavashop/*` 仓库。
- [ ] 每个镜像标签都等于短 git SHA；任何地方都没有 `:latest`。
- [ ] `az containerapp env show` 报告 `Succeeded`。
- [ ] `az containerapp list -g $RG -o table` 显示 8 个应用，全部 `Succeeded`。
- [ ] 全部 8 个 `/healthz` 探测返回 `200`。
- [ ] 对 `inventory` 的 `POST /invoke` 返回了关于 `ZS-1042` 的答复。
- [ ] `.env.fqdns` 已存在且被 git 忽略。
- [ ] 没有任何 Container App 机密暴露字面 `value`。

下一步 → [实验 03 — 部署 AKS 集群](../lab-03-deploy-aks-cluster/README.zh.md)

---

## 故障排查

| 现象 | 处理 |
|---|---|
| `az acr build` 报 `denied` | 重新 `az login`；确认实验 01 步骤 8 中资源组上的 `Contributor`。 |
| 镜像构建在 `BASE_IMAGE` 处失败 | 你传了本地标签。必须是 `$ACR.azurecr.io/zavashop/base:$GIT_SHA`。 |
| Container App 卡在 `Activating` | `az containerapp logs show -g $RG -n <app> --tail 100`。通常是镜像标签有误。 |
| `Failed to pull image` | UAMI 缺少 `AcrPull`，或推送的标签不存在。 |
| `GITHUB-TOKEN` 机密解析错误 | UAMI 缺少 `Key Vault Secrets User`，或机密名不是精确的 `GITHUB-TOKEN`。 |
| `/invoke` 返回 500 | Copilot 令牌无效或过期。重新写入 Key Vault，然后 `az containerapp revision restart`。 |
| `/invoke` 超时 | 提高 `ZAVA_COPILOT_TIMEOUT_SECONDS`，或先请求 `/healthz` 预热。 |
| `deploy.sh` 报 `missing UAMI_RESOURCE_ID` | `.env.lab` 过期。重新执行实验 01 步骤 13。 |
