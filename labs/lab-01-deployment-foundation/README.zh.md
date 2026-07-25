# 实验 01 — 部署基础：接收 Agent 舰队并搭好 Azure 底座

> ⏱ 约 60 分钟 · **本实验系列不生成代码。** 多 Agent 应用已经存在于 `src/` 中。你要做的是克隆它、验证它能运行，并搭建它将被部署到的 Azure 基础设施。

🇬🇧 [English](./README.md)

## 场景故事

你是 **ZavaShop**（一家拥有 500+ 门店的全球零售商）的值班平台工程师。

上个迭代，一组 **GitHub Copilot Custom Coding Agents** 交付了一套完整的多 Agent 供应链应用。它已经在这个仓库里了：

- 5 个 Microsoft Agent Framework Agent（1 个编排器 + 4 个专家）
- 4 个 FastMCP 工具服务器
- 单元测试、集成测试和黄金评测集
- Dockerfile、Helm Chart、ACA Bicep 模块和 CD 工作流

没有一行是手写的。现在业务方要求 **本周内上线到 Azure**。

这四个实验中你的任务不是构建应用，而是回答一个问题：

> *"我刚收到一个由 Custom Coding Agents 产出的多 Agent 项目。如何快速、安全、可重复地把它部署到 Azure 上？"*

| 实验 | 你要做什么 |
|---|---|
| **01（本实验）** | 克隆交付的应用，验证本地可运行，预配共享的 Azure 基础设施 |
| **02** | 容器化整个舰队，用 **Bicep 把 8 个服务部署到 Azure Container Apps** |
| **03** | 学习并预配 **AKS** —— Entra ID、Azure RBAC、Workload Identity、Helm |
| **04** | **真实的端到端部署**：Bicep + Helm + GitHub Actions OIDC、冒烟测试、评测、Day-2 运维 |

---

## 你收到的是什么

这就是你要部署的应用。请阅读它，不要重写它。

### 运行时服务（9 个容器 + 1 个基础镜像）

| 服务 | 源码 | 运行环境 | 端口 | 路由 |
|---|---|---|---|---|
| `orchestrator` | `src/agents/orchestrator/` | AKS（实验 03/04） | 8000 | `/healthz` `/readyz` `/invoke` `/plan` |
| `inventory` | `src/agents/inventory/` | ACA（实验 02） | 8000 | `/healthz` `/readyz` `/invoke` |
| `supplier` | `src/agents/supplier/` | ACA（实验 02） | 8000 | `/healthz` `/readyz` `/invoke` |
| `logistics` | `src/agents/logistics/` | ACA（实验 02） | 8000 | `/healthz` `/readyz` `/invoke` |
| `pricing` | `src/agents/pricing/` | ACA（实验 02） | 8000 | `/healthz` `/readyz` `/invoke` |
| `inventory-mcp` | `src/mcp_servers/inventory/` | ACA（实验 02） | 8080 | `/healthz` `/readyz` `/mcp` |
| `supplier-mcp` | `src/mcp_servers/supplier/` | ACA（实验 02） | 8080 | `/healthz` `/readyz` `/mcp` |
| `shipping-mcp` | `src/mcp_servers/shipping/` | ACA（实验 02） | 8080 | `/healthz` `/readyz` `/mcp` |
| `pricing-mcp` | `src/mcp_servers/pricing/` | ACA（实验 02） | 8080 | `/healthz` `/readyz` `/mcp` |
| `base` | `src/Dockerfile.base` | 共享层 | — | — |

### 代码中固化的业务场景

> *"SKU `ZS-1042` 将在周五于 `store-101` 缺货。我们该怎么办？"*

编排器上的 `POST /plan` 会通过 A2A HTTP 扇出到四个专家 Agent，每个专家调用自己的 MCP 服务器，编排器最终返回一个结构化的 `Plan`，包含 `stock_view`、`po_view`、`shipping_view`、`price_view`、`summary` 和 `next_actions`。

### 交付仓库的结构

```
src/
├── Dockerfile.base                 # python:3.11-slim，非 root uid 10001
├── shared/                         # Settings、telemetry、Copilot client、FastAPI 工厂
├── agents/<name>/                  # agent.py · tools.py · prompts.py · server.py · Dockerfile
└── mcp_servers/<name>/             # server.py · models.py · store.py · Dockerfile
tests/                              # 单元 · 集成 · tests/evals（黄金集）
infra/
├── aca/agent.bicep + deploy.sh     # 实验 02
└── aks/helm/zavashop + aks/wif     # 实验 03 / 04
.github/workflows/deploy.yml        # 实验 04
docker-compose.yml                  # 本地整体运行
```

---

## 本实验相关的 Microsoft Learn 知识

- [Azure 容器注册表简介](https://learn.microsoft.com/zh-cn/azure/container-registry/container-registry-intro) —— ACA 与 AKS 共用一个注册表。
- [Azure Key Vault 概述](https://learn.microsoft.com/zh-cn/azure/key-vault/general/overview) —— Copilot 的 `GITHUB-TOKEN` 存放处。
- [Azure 资源的托管标识](https://learn.microsoft.com/zh-cn/entra/identity/managed-identities-azure-resources/overview) —— 一个 UAMI 服务于 AKS Pod、ACA 应用和 GitHub Actions。
- [工作负载标识联合](https://learn.microsoft.com/zh-cn/entra/workload-id/workload-identity-federation) —— CI 与 Pod 的无密钥认证。
- [为 Azure 配置 GitHub Actions OIDC](https://learn.microsoft.com/zh-cn/azure/developer/github/connect-from-azure-openid-connect) —— `repo:OWNER/REPO:ref:refs/heads/main` 主体格式。
- [Microsoft Defender for Containers](https://learn.microsoft.com/zh-cn/azure/defender-for-cloud/defender-for-containers-introduction) —— 部署门禁检查的容器安全基线。
- [Azure Monitor 容器见解](https://learn.microsoft.com/zh-cn/azure/azure-monitor/containers/container-insights-overview) —— 实验 03 中 AKS 将接入的工作区。
- [AKS 登陆区加速器](https://learn.microsoft.com/zh-cn/azure/cloud-adoption-framework/scenarios/app-platform/aks/landing-zone-accelerator) —— 本基础设施对应的设计领域。

---

## 本实验预配的资源

| 资源 | 命名模式 | 使用方 |
|---|---|---|
| 资源组 | `rg-zavashop-lab` | 全部 |
| 容器注册表 | `acrzavashop<rand>` | 实验 02 + 03 + 04 |
| Log Analytics 工作区 | `law-zava-<rand>` | 实验 03（容器见解、Defender） |
| 用户分配托管标识 | `uami-zavashop` | ACA 应用、AKS Pod、GitHub Actions |
| Entra ID 安全组 | `zavashop-aks-admins` | 实验 03 的人员集群访问 |
| Key Vault | `kv-zava-<rand>` + `GITHUB-TOKEN` | 所有工作负载 |
| Defender for Cloud 计划 | `Containers`、`KeyVaults` | 实验 04 部署门禁 |
| `.env.lab` | 仓库根目录，已 git-ignore | 实验 02–04 |

> 这里**不创建** AKS —— 那是实验 03。**不创建** Container Apps 环境 —— 那是实验 02。

---

## 步骤 0 — 工具检查

```bash
az version                  # >= 2.65
kubectl version --client
helm version                # >= 3.14
docker version
python --version            # 3.11+（CI 使用 3.13）
uv --version
git --version
```

安装缺失的工具：

```bash
# macOS
brew install azure-cli kubectl helm uv git

# Windows
winget install --id Microsoft.AzureCLI
winget install --id Kubernetes.kubectl
winget install --id Helm.Helm
winget install --id astral-sh.uv
```

添加后续实验需要的 Azure CLI 扩展：

```bash
az extension add -n containerapp --upgrade
az provider register -n Microsoft.App --wait
az provider register -n Microsoft.ContainerService --wait
```

---

## 步骤 1 — 克隆交付的应用

```bash
git clone https://github.com/microsoft/AKS-Lab-GitHubCopilot.git
cd AKS-Lab-GitHubCopilot
```

按照 CI 的方式安装 Python 环境：

```bash
uv sync
```

确认交付内容完整：

```bash
ls src/agents          # inventory logistics orchestrator pricing supplier
ls src/mcp_servers     # inventory pricing shipping supplier
ls infra/aca           # agent.bicep deploy.sh
ls infra/aks/helm/zavashop
```

---

## 步骤 2 — 10 分钟代码导览

按顺序阅读这六个文件。你不会修改它们，但实验 02–04 的每一个部署决策都能追溯到其中之一。

| # | 文件 | 关注点 |
|---|---|---|
| 1 | `src/shared/settings.py` | 每个运行时开关都是一个 `ZAVA_*` 环境变量。这就是完整的部署契约。 |
| 2 | `src/shared/server.py` | `make_app()` 为每个服务提供 `/healthz`、`/readyz`、`/invoke`。探针开箱即用。 |
| 3 | `src/agents/orchestrator/server.py` | `POST /plan` 扇出到四个专家并组装 `Plan`。 |
| 4 | `src/agents/orchestrator/a2a.py` | 专家通过 **URL** 访问，绝不通过 Python import —— 这正是它们能跑在 ACA 上的原因。 |
| 5 | `src/mcp_servers/inventory/server.py` | FastMCP 监听 `8080`，路径 `/mcp`，另加自定义 `/healthz` + `/readyz`。 |
| 6 | `src/Dockerfile.base` | 多阶段构建，非 root 用户 `zava` uid `10001`，`EXPOSE 8000`。 |

你必须在 Azure 中满足的部署契约：

| 环境变量 | 使用方 | 在 Azure 中的取值 |
|---|---|---|
| `GITHUB_TOKEN` | 所有 Agent | Key Vault `GITHUB-TOKEN`（AKS 用 CSI，ACA 用 `secretRef`） |
| `ZAVA_COPILOT_MODEL` | 所有 Agent | `gpt-5.5` |
| `ZAVA_COPILOT_TIMEOUT_SECONDS` | 所有 Agent | `120` |
| `ZAVA_<NAME>_MCP_URL` | 专家 Agent | ACA MCP FQDN + `/mcp`（实验 02） |
| `ZAVA_<NAME>_A2A_URL` | 编排器 | ACA 专家 FQDN + `/invoke`（实验 04） |
| `AZURE_CLIENT_ID` | 所有工作负载 | UAMI 客户端 ID（Workload Identity） |

---

## 步骤 3 — 运行质量门禁

交付的代码本身是绿的。在动 Azure 之前先验证它 —— 实验 04 的部署工作流在这一步失败时会拒绝发布。

```bash
uv run poe check
```

依次执行 `ruff check` → `ruff format --check` → `pyright`（strict）→ `pytest`。

期望输出结尾：

```
=== N passed in Xs ===
```

如果失败，请修复环境（通常是 `uv sync` 过期），而不要"修复"应用代码 —— 它是交付物。

---

## 步骤 4 — 获取 GitHub Copilot 令牌

本仓库中的每个 Agent 都使用 **GitHub Copilot SDK**，`model="gpt-5.5"`。不需要任何 Azure OpenAI 部署。认证方式是从 `GITHUB_TOKEN` 读取的 GitHub 令牌。

1. 打开 <https://github.com/settings/personal-access-tokens/new>。
2. 创建一个带 **Copilot（读取）** 权限的 **细粒度 PAT**。
3. **不要**授予仓库写权限。

```bash
export GITHUB_TOKEN="<你的细粒度-PAT>"
```

冒烟测试：

```bash
uv run python - <<'PY'
import asyncio
from agent_framework.github import GitHubCopilotAgent, GitHubCopilotOptions

async def main():
    agent = GitHubCopilotAgent(
        name="smoke",
        instructions="Reply briefly.",
        options=GitHubCopilotOptions(model="gpt-5.5"),
    )
    result = await agent.run("Say hi from gpt-5.5")
    print(result.output_text)

asyncio.run(main())
PY
```

> ⚠️ 绝不要提交令牌。它将在步骤 10 进入 Key Vault，并由 CSI（AKS）和 `secretRef`（ACA）注入到工作负载中。

---

## 步骤 5 — 在本地运行整个舰队（推荐）

这是理解你即将部署的内容的最快方式。Docker Compose 会用与 Azure 相同的镜像启动全部 9 个服务。

```bash
export GIT_SHA=$(git rev-parse --short HEAD)

docker compose --profile build build zavashop-base
docker compose build
docker compose up -d
```

观察健康状态：

```bash
docker compose ps
```

执行与实验 04 对 AKS 相同的调用：

```bash
curl -fsS http://localhost:8000/healthz
curl -fsS http://localhost:8000/readyz

curl -fsS -X POST http://localhost:8000/plan \
  -H 'content-type: application/json' \
  -d '{
        "goal": "Store 101 will stock out of SKU ZS-1042 by Friday.",
        "sku": "ZS-1042",
        "store_id": "store-101"
      }'
```

清理：

```bash
docker compose down -v
```

---

## 步骤 6 — 登录 Azure 并设置命名变量

```bash
az login --use-device-code
az account set --subscription "<你的订阅 ID>"
```

```bash
export LOCATION="eastus2"
export RG="rg-zavashop-lab"
export RAND="$RANDOM"
export ACR="acrzavashop${RAND}"
export LAW="law-zava-${RAND}"
export KV="kv-zava-${RAND}"
export UAMI="uami-zavashop"
export AKS="aks-zavashop"
export CAE="cae-zavashop"
export AKS_ADMINS_GROUP="zavashop-aks-admins"
```

> 这里 `AKS` 和 `CAE` 只是名字。资源分别在实验 03 和实验 02 创建 —— 但名字现在就写进 `.env.lab`，让后续实验保持可直接复制粘贴。

---

## 步骤 7 — 资源组、容器注册表、Log Analytics

```bash
az group create -n $RG -l $LOCATION \
  --tags project=zavashop lab=01
```

```bash
az acr create -n $ACR -g $RG \
  --sku Standard \
  --admin-enabled false \
  --tags project=zavashop lab=01
```

> `--admin-enabled false` 是刻意的。实验 02–04 使用托管标识拉取镜像，绝不使用注册表用户名和密码。

```bash
az monitor log-analytics workspace create \
  -g $RG -n $LAW -l $LOCATION \
  --tags project=zavashop lab=01

export LAW_ID=$(az monitor log-analytics workspace show \
  -g $RG -n $LAW --query id -o tsv)
```

---

## 步骤 8 — 唯一的托管标识

一个用户分配托管标识（UAMI）同时作为 **ACA 应用**、**AKS Pod** 和 **GitHub Actions** 的身份。只创建一次，就让后续实验全程无密钥。

```bash
az identity create -n $UAMI -g $RG -l $LOCATION \
  --tags project=zavashop lab=01

export UAMI_CLIENT_ID=$(az identity show -n $UAMI -g $RG \
  --query clientId -o tsv)
export UAMI_PRINCIPAL_ID=$(az identity show -n $UAMI -g $RG \
  --query principalId -o tsv)
export UAMI_RESOURCE_ID=$(az identity show -n $UAMI -g $RG \
  --query id -o tsv)
```

授予它今天需要的两个角色：

```bash
export ACR_ID=$(az acr show -n $ACR -g $RG --query id -o tsv)
export RG_ID=$(az group show -n $RG --query id -o tsv)

# 1. 数据平面：拉取镜像（ACA 和 AKS kubelet 使用）
az role assignment create \
  --assignee-object-id $UAMI_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role AcrPull \
  --scope $ACR_ID

# 2. 控制平面：az acr build、ACA Bicep 部署、AKS 更新
az role assignment create \
  --assignee-object-id $UAMI_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role Contributor \
  --scope $RG_ID
```

> 💡 **两个都必须有。** `AcrPull` 只是数据平面。缺少 `Contributor`（或任何授予 `Microsoft.ContainerRegistry/registries/read` 的角色）时，CI 会报出误导性的错误：*"The resource with name '<acr>' could not be found"*。AKS 专属的 Cluster User 角色将在实验 03 集群创建后再添加。

---

## 步骤 9 — 集群运维人员的 Entra ID 组

实验 03 会为 AKS 启用 Microsoft Entra ID + Azure RBAC。现在先创建承载人员的组。

```bash
az ad group create \
  --display-name $AKS_ADMINS_GROUP \
  --mail-nickname $AKS_ADMINS_GROUP

export AKS_ADMINS_GROUP_ID=$(az ad group show \
  --group $AKS_ADMINS_GROUP --query id -o tsv)

export MY_OID=$(az ad signed-in-user show --query id -o tsv)

az ad group member add \
  --group $AKS_ADMINS_GROUP_ID \
  --member-id $MY_OID
```

> 如果租户禁止创建组，请向 Entra 管理员索要一个组对象 ID，然后执行 `export AKS_ADMINS_GROUP_ID="<组对象 ID>"` 继续。

---

## 步骤 10 — Key Vault 与 `GITHUB-TOKEN` 机密

```bash
az keyvault create -n $KV -g $RG -l $LOCATION \
  --enable-rbac-authorization \
  --tags project=zavashop lab=01

export KV_ID=$(az keyvault show -n $KV -g $RG --query id -o tsv)
```

授予 UAMI 读权限、你自己写权限：

```bash
az role assignment create \
  --assignee-object-id $UAMI_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope $KV_ID

az role assignment create \
  --assignee $MY_OID \
  --role "Key Vault Secrets Officer" \
  --scope $KV_ID
```

写入 Copilot 令牌（RBAC 生效可能需要一分钟，遇到 403 请重试）：

```bash
az keyvault secret set \
  --vault-name $KV \
  --name GITHUB-TOKEN \
  --value "$GITHUB_TOKEN" \
  -o none

az keyvault secret show \
  --vault-name $KV --name GITHUB-TOKEN \
  --query id -o tsv
```

机密名 `GITHUB-TOKEN` 被硬编码在 `infra/aca/agent.bicep` 和 `infra/aks/helm/zavashop/templates/secretproviderclass.yaml` 中。不要改名。

---

## 步骤 11 — Defender for Cloud 基线

实验 04 的部署门禁要求这两个计划为 `Standard`，否则拒绝发布。

```bash
az provider register -n Microsoft.Security --wait

az security pricing create -n Containers --tier Standard
az security pricing create -n KeyVaults --tier Standard
```

验证：

```bash
az security pricing show -n Containers --query pricingTier -o tsv   # Standard
az security pricing show -n KeyVaults  --query pricingTier -o tsv   # Standard
```

> 如果实验订阅有中央策略阻止此操作，请记录现有配置后继续。实验 04 会说明如何记录例外，而不是绕过门禁。

---

## 步骤 12 — GitHub Actions 联合凭据

实验 04 会从 GitHub Actions 执行同样的发布流程，且**不使用任何客户端密钥**。现在把 UAMI 联合到你 fork 的 `main` 分支。

```bash
export GH_REPO_SLUG="OWNER/REPO"   # 例如 contoso/AKS-Lab-GitHubCopilot

az identity federated-credential create \
  --name gha-aks-lab-main \
  --identity-name $UAMI \
  --resource-group $RG \
  --issuer https://token.actions.githubusercontent.com \
  --subject "repo:${GH_REPO_SLUG}:ref:refs/heads/main" \
  --audiences api://AzureADTokenExchange
```

请逐字核对 subject —— 这里的拼写错误会在后面表现为 `AADSTS70021: no matching federated identity record found`。

```bash
az identity federated-credential list \
  --identity-name $UAMI -g $RG \
  --query "[].{name:name, subject:subject}" -o table
```

---

## 步骤 13 — 把基础信息持久化到 `.env.lab`

后续每个实验都以 `source .env.lab` 开始。

```bash
export AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)

cat > .env.lab <<EOF
LOCATION=$LOCATION
RG=$RG
RAND=$RAND
ACR=$ACR
ACR_ID=$ACR_ID
AKS=$AKS
CAE=$CAE
KV=$KV
KV_ID=$KV_ID
LAW=$LAW
LAW_ID=$LAW_ID
UAMI=$UAMI
UAMI_CLIENT_ID=$UAMI_CLIENT_ID
UAMI_PRINCIPAL_ID=$UAMI_PRINCIPAL_ID
UAMI_RESOURCE_ID=$UAMI_RESOURCE_ID
AKS_ADMINS_GROUP=$AKS_ADMINS_GROUP
AKS_ADMINS_GROUP_ID=$AKS_ADMINS_GROUP_ID
RG_ID=$RG_ID
COPILOT_MODEL=gpt-5.5
AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID
AZURE_TENANT_ID=$AZURE_TENANT_ID
GH_REPO_SLUG=$GH_REPO_SLUG
EOF
```

确保它永远不会被提交：

```bash
grep -qxF '**/.env.lab' .gitignore \
  || echo '**/.env.lab' >> .gitignore

grep -qxF '**/.env.fqdns' .gitignore \
  || echo '**/.env.fqdns' >> .gitignore

git ls-files --error-unmatch .env.lab >/dev/null 2>&1 \
  && echo "ERROR: .env.lab is tracked — rotate the PAT and git rm --cached it"
```

> `.env.lab` 只包含资源名和身份 ID —— **不包含**真实的 `GITHUB_TOKEN`。令牌只存在于你的 shell 和 Key Vault 中。

---

## 步骤 14 — 注册 GitHub 仓库机密与变量

只有实验 04 的 CD 流水线需要，但趁值还在 shell 里先设置好。

```bash
gh secret set AZURE_SUBSCRIPTION_ID \
  --body "$AZURE_SUBSCRIPTION_ID" --repo "$GH_REPO_SLUG"
gh secret set AZURE_TENANT_ID \
  --body "$AZURE_TENANT_ID" --repo "$GH_REPO_SLUG"
gh secret set AZURE_CLIENT_ID \
  --body "$UAMI_CLIENT_ID" --repo "$GH_REPO_SLUG"
```

```bash
gh variable set AZURE_RESOURCE_GROUP \
  --body "$RG" --repo "$GH_REPO_SLUG"
gh variable set AZURE_CONTAINER_REGISTRY \
  --body "$ACR" --repo "$GH_REPO_SLUG"
gh variable set AKS_CLUSTER \
  --body "$AKS" --repo "$GH_REPO_SLUG"
gh variable set CONTAINER_APPS_ENV \
  --body "$CAE" --repo "$GH_REPO_SLUG"
gh variable set KEY_VAULT_NAME \
  --body "$KV" --repo "$GH_REPO_SLUG"
gh variable set UAMI_NAME \
  --body "$UAMI" --repo "$GH_REPO_SLUG"
gh variable set UAMI_RESOURCE_ID \
  --body "$UAMI_RESOURCE_ID" --repo "$GH_REPO_SLUG"
```

这些名称与 `.github/workflows/deploy.yml` 完全一致。

---

## ✅ 验证

```bash
source .env.lab

az group show -n $RG --query properties.provisioningState -o tsv
az acr show -n $ACR -g $RG --query loginServer -o tsv
az identity show -n $UAMI -g $RG --query clientId -o tsv
az keyvault secret show --vault-name $KV --name GITHUB-TOKEN \
  --query name -o tsv
az security pricing show -n Containers --query pricingTier -o tsv
az security pricing show -n KeyVaults --query pricingTier -o tsv

az role assignment list --assignee "$UAMI_PRINCIPAL_ID" --all \
  --query "[].{role:roleDefinitionName, scope:scope}" -o table
```

达成以下条件即完成实验 01：

- [ ] `uv run poe check` 全绿。
- [ ] Copilot SDK 冒烟测试打印出了 `gpt-5.5` 的回复。
- [ ] 本地 Compose 舰队的 `POST /plan` 返回了结构化计划。
- [ ] `rg-zavashop-lab` 中包含 ACR、Key Vault、Log Analytics 和 UAMI。
- [ ] UAMI 拥有 `AcrPull`（ACR 范围）和 `Contributor`（资源组范围）。
- [ ] Key Vault 中存在 `GITHUB-TOKEN`。
- [ ] Defender 的 `Containers` 和 `KeyVaults` 计划返回 `Standard`。
- [ ] `gha-aks-lab-main` 联合凭据存在且 subject 正确。
- [ ] 仓库根目录存在 `.env.lab` 且已被 git 忽略。

下一步 → [实验 02 — 部署到 Container Apps](../lab-02-deploy-container-apps/README.zh.md)

---

## 故障排查

| 现象 | 处理 |
|---|---|
| `uv run poe check` 导入失败 | 重新执行 `uv sync`；确认 Python 3.11+。 |
| Copilot SDK 返回 401 | PAT 缺少 Copilot 权限，或账号没有 Copilot 席位。 |
| Copilot SDK 对 `gpt-5.5` 返回 403 | 你的 Copilot 计划可能不含 `gpt-5.5`。检查 <https://github.com/settings/copilot>。 |
| `az keyvault secret set` 返回 403 | RBAC 尚未生效。等待 60 秒后重试。 |
| `docker compose up` 报 `BASE_IMAGE` 错误 | 先导出 `GIT_SHA` 再执行 `docker compose --profile build build zavashop-base`。 |
| `az ad group create` 被拒绝 | 租户策略限制。向 Entra 管理员索要组对象 ID。 |
| `az security pricing create` 被拒绝 | 中央 Defender 策略。记录现有配置后继续。 |
| `.env.lab` 被误提交 | 立即轮换 PAT，然后执行 `git rm --cached .env.lab`。 |
