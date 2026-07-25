# 把一个 **GitHub Copilot 自定义 Coding Agents** 应用部署到 Azure

![bg](./imgs/arch.png)

> 你刚刚接手了一个已经完工的多 Agent 应用。它**不是**手写的 —— 一支由六个 **GitHub Copilot 自定义 Coding Agents** 组成的团队产出了规格说明、Agent 代码、MCP 服务器、测试、Bicep 和 Helm Chart。你的任务是接下来的部分：**把它快速、安全地跑到 Azure 上。**
>
> 技术栈：**Microsoft Agent Framework (MAF)** + **GitHub Copilot SDK**（`gpt-5.5`）+ **Azure Container Apps** + **AKS** + **Microsoft Entra ID** + **Microsoft Defender for Cloud**。

🇬🇧 [English](./README.md)

---

## 🎬 场景

一个合作团队用 GitHub Copilot 自定义 Coding Agents 构建了 **ZavaShop** —— 一个 AI 原生的零售供应链控制平面。代码完整、类型齐全、测试通过、已容器化。唯一缺的，是它在 Azure 上的落脚点。

你是平台工程师。四个实验，一个下午，从 `git clone` 到一个由 Entra ID、Workload Identity、Key Vault 和 Defender for Cloud 守护的线上控制平面。

**这套实验不要求你生成代码。** `src/` 下的应用就是你收到的交付物。你要做的是读懂它、打包它、部署它、运维它。

| # | 实验 | 快速通道 | 完整 | 产出 |
|---|---|---|---|---|
| 01 | [部署基础](./labs/lab-01-deployment-foundation/README.zh.md) | 约 25 分钟 | 约 60 分钟 | 接收交付的应用，用 Docker Compose 在本地跑通舰队，并搭好后续每一次部署都依赖的 Azure 基础：RG、ACR、Log Analytics、UAMI、Entra 组、Key Vault、Defender、GitHub OIDC |
| 02 | [部署到 Container Apps](./labs/lab-02-deploy-container-apps/README.zh.md) | 约 30 分钟 | 约 60 分钟 | 用 ACR Tasks 构建 10 个镜像，并用一个可复用的 **Bicep** 模块把 8 个服务部署到 **Azure Container Apps** |
| 03 | [部署 AKS 集群](./labs/lab-03-deploy-aks-cluster/README.zh.md) | 约 35 分钟 | 约 75 分钟 | 预配符合登陆区规范的 **AKS** 集群：**Microsoft Entra ID** + Azure RBAC、Workload Identity、Key Vault CSI、Azure Policy、Defender —— 然后读懂并离线渲染 **Helm** Chart |
| 04 | [部署到生产](./labs/lab-04-deploy-production/README.zh.md) | 约 30 分钟 | 约 75 分钟 | 真正的发布：门禁 → `what-if` → **Bicep** 发 ACA → **Helm** 发 AKS → 冒烟 → 评测 → 无密钥 GitHub Actions CD → Day-2 局部发布与回滚 |

> **两小时通道。** 每个实验开头都有一张表，把步骤分为 **核心** 和 *可选*。只做核心步骤，整个系列约 2 小时完成；可选步骤是深入讲解，后续实验不依赖它们。

每个实验目录下都同时提供英文 `README.md` 与中文 `README.zh.md`。

### 实验之间传递的东西

四个实验通过仓库根目录下两个被 git 忽略的文件串联。每个实验开头都会 `source` 它们，因此请按顺序做。

| 文件 | 由谁写入 | 内容 | 被谁消费 |
|---|---|---|---|
| `.env.lab` | 实验 01 步骤 13 写入，实验 02 步骤 6、实验 03 步骤 4 追加 | `RG`、`ACR`、`LAW`、`KV`、`UAMI_*`、`AKS_ADMINS_GROUP_ID`、`CAE_ID`、`AKS_OIDC`、`AKS_ID` | 实验 02、03、04 |
| `.env.fqdns` | 实验 02 的 `deploy.sh` 写入，实验 04 刷新 | 4 个专家 Agent 与 4 个 MCP 服务器的 `ZAVA_*_URL` | 实验 03、04（Helm values 与 `/plan`） |

两个文件都不含机密 —— Copilot 令牌只存在于 Key Vault。实验 01 会把它们加入 `.gitignore` 并断言未被跟踪。

---

## 🛍 ZavaShop 是什么

**ZavaShop** 是一家拥有 500+ 门店的高速成长型全球零售商。它的供应链跑在一堆遗留 ERP、供应商门户和临时表格上。你收到的这个应用就是它的 AI 原生控制平面 —— 一支协作的 Agent 舰队：

| 应用 Agent | 职责 | 运行时 |
|---|---|---|
| `OrchestratorAgent` | "店长" —— 把目标路由给各专家 Agent，并把结果合并成一份计划 | **AKS** —— 集群在实验 03，工作负载在实验 04 |
| `InventoryAgent` | 监控门店与仓库的缺货风险 | **ACA**（实验 02） |
| `SupplierAgent` | 通过 MCP 工具起草采购订单 | **ACA**（实验 02） |
| `LogisticsAgent` | 规划发运、跟踪 ETA、异常改道 | **ACA**（实验 02） |
| `PricingAgent` | 基于需求与竞品信号给出动态定价建议 | **ACA**（实验 02） |

每个专家 Agent 都通过一个专属的 **MCP 服务器**（`inventory-mcp`、`supplier-mcp`、`shipping-mcp`、`pricing-mcp`）访问后端能力，因此模型永远不持有业务状态。

| 层次 | 服务 | 运行时 | 端口与探针 |
|---|---|---|---|
| 编排 | `orchestrator` | AKS + Helm（实验 03 → 实验 04） | Uvicorn `8000`；`/healthz`、`/readyz`、`/plan`、`/invoke` |
| 专家 Agent | `inventory`、`supplier`、`logistics`、`pricing` | Azure Container Apps（实验 02） | Uvicorn `8000`；`/healthz`、`/readyz`、`/invoke` |
| MCP 工具 | `inventory-mcp`、`supplier-mcp`、`shipping-mcp`、`pricing-mcp` | Azure Container Apps（实验 02） | FastMCP `8080`；`/healthz`、`/readyz`、`/mcp` |

镜像以短 git SHA 打标，通过 ACR Tasks 为 `linux/amd64` 构建。**任何地方都没有 `:latest`。** 实验中 ACA 服务保持 `minReplicas=1`，以保证冒烟测试与线上评测稳定；生产登陆区在设计好 DNS、网络和冷启动预算后可以再考虑私有入口与缩容到零。

### 贯穿全程的业务场景

> 门店 `store-101` 的 SKU `ZS-1042` 将在周五缺货。请给出处置计划。

一次对编排器的 `POST /plan` 会扇出到四个专家 Agent，每个再调用自己的 MCP 服务器，最终返回一份包含 `stock_view`、`po_view`、`shipping_view`、`price_view`、`summary`、`next_actions` 的合并计划。这一次调用就是整套部署的验收测试。

---

## 🏗 目标架构

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ 安全与身份护栏                                                                │
│                                                                              │
│  Microsoft Entra ID + Azure RBAC                                             │
│  - GitHub Actions OIDC -> UAMI（仓库中不存客户端密钥）              实验 01     │
│  - Key Vault GITHUB-TOKEN -> ACA 上的 keyVaultUrl 机密          实验 02     │
│  - AKS 的 Entra ID + Azure RBAC、Workload Identity、KV CSI      实验 03     │
│                                                                              │
│  Microsoft Defender for Cloud                                                │
│  - Defender Containers + KeyVaults 计划设为 Standard             实验 01     │
│  - AKS Defender 配置文件 + Azure Policy 附加组件                实验 03     │
│  - 以上全部在发布前作为硬门禁重新断言                          实验 04     │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ AKS      集群在实验 03 创建 · 编排器在实验 04 发布                          │
│  OrchestratorAgent（MAF Workflow + GitHub Copilot SDK）                       │
│  ServiceAccount: orchestrator-sa + azure.workload.identity/use=true          │
│  路由: /healthz, /readyz, /plan, /invoke                                      │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │ A2A / HTTP (/invoke)
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ Azure Container Apps 专家 Agent      实验 02 部署 · 实验 04 重新发布            │
│  inventory  | supplier  | logistics  | pricing                               │
│  路由: /healthz, /readyz, /invoke                                             │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │ Copilot SDK 经 HTTP 调用远程 MCP
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ Azure Container Apps MCP 服务器      实验 02 部署 · 实验 04 重新发布            │
│  inventory-mcp | supplier-mcp | shipping-mcp | pricing-mcp                   │
│  路由: /healthz, /readyz, /mcp                                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 架构中的 Entra ID 与 Defender for Cloud

这些管控项包裹在舰队外围，而不是混进零售业务逻辑里。实验 01 与实验 03 负责预配，实验 04 在发布前把它们作为硬门禁重新校验一遍。

| 管控项 | 出现位置 | 保护什么 |
|---|---|---|
| AKS 的 Microsoft Entra ID | 实验 03 用 `--enable-aad --enable-azure-rbac` 创建集群。人员访问走 Entra ID + Azure RBAC，绝不用管理员 kubeconfig。 | 防止非托管的本地集群凭据变成日常运维路径。 |
| GitHub Actions OIDC | 实验 01 创建联合凭据；工作流授予 `id-token: write` 并使用 `azure/login@v2`。 | 让 CI/CD 无需在 GitHub 中存放客户端密钥即可认证到 Azure。 |
| Workload Identity 与 UAMI | 实验 03 把 `system:serviceaccount:zavashop:orchestrator-sa` 联合到 UAMI；ACA 应用使用同一个用户分配标识。 | 让工作负载获得 Azure 身份，而无需内嵌密码或服务主体密钥。 |
| Key Vault 机密交付 | AKS 通过 Secrets Store CSI 读取 `GITHUB-TOKEN`；ACA 通过 `keyVaultUrl` 机密读取。 | 让 Copilot 凭据远离 Helm values、Bicep 参数、镜像和源码仓库。 |
| Defender for Cloud | 实验 01 把 `Containers` 和 `KeyVaults` 设为 `Standard`；实验 03 启用 AKS Defender 配置文件。 | 当容器与机密面未被安全基线覆盖时阻止发布。 |
| Azure Policy 附加组件 | 实验 03 建集群时启用，实验 04 断言。 | 在应用发布前保持集群与 AKS 登陆区基线一致。 |

---

## 🤖 背景：这个应用是怎么造出来的

完成实验并不需要你运行这些 Agent —— 但了解代码的来处仍有价值，因为扩展它时用的正是同一套工作模式。

本仓库中的每一件产物 —— 规格说明、Agent 代码、MCP 服务器、测试、Bicep、Helm、CI —— 都由一个**具名的 GitHub Copilot 自定义 Coding Agent** 编写，每个 Agent 拥有仓库的一个切片，并携带自己的工具、技能与拒绝规则。

```
       ┌─────────────────────── GitHub Copilot 多自定义 Agents ────────────────────────────┐
       │                                                                                  │
 Issue ─►  /requirements-analyst  ─►  specs/<slug>.md                                      │
                  │                                                                       │
                  ▼                                                                       │
          /mcp-builder  ───────►  src/mcp_servers/*                                        │
          /agent-builder  ─────►  src/agents/<specialist>/*                                │
          /orchestrator-architect ─► src/agents/orchestrator, src/shared, docker-compose   │
                  │                                                                       │
                  ▼                                                                       │
          /test-author  ───────►  tests/** （单元 · 集成 · 评测）                            │
                  │                                                                       │
                  ▼                                                                       │
          /deploy-engineer  ───►  infra/** + .github/workflows/** + ACR/ACA/AKS 发布        │
       └──────────────────────────────────────────────────────────────────────────────────┘
```

| 阶段 | Coding Agent | 拥有 | 文件 |
|---|---|---|---|
| 需求 | `/requirements-analyst` | 仅 `specs/*.md` —— 拒绝写代码 | [.github/agents/requirements-analyst.agent.md](.github/agents/requirements-analyst.agent.md) |
| MCP 实现 | `/mcp-builder` | `src/mcp_servers/*`（每轮一个服务器） | [.github/agents/mcp-builder.agent.md](.github/agents/mcp-builder.agent.md) |
| Agent 实现 | `/agent-builder` | `src/agents/<specialist>/*`（每轮一个专家） | [.github/agents/agent-builder.agent.md](.github/agents/agent-builder.agent.md) |
| 编排 | `/orchestrator-architect` | `src/agents/orchestrator/*`、`src/shared/*`、`docker-compose.yml` | [.github/agents/orchestrator-architect.agent.md](.github/agents/orchestrator-architect.agent.md) |
| 测试 | `/test-author` | 仅 `tests/**` —— 从不改 `src/` | [.github/agents/test-author.agent.md](.github/agents/test-author.agent.md) |
| 部署 | `/deploy-engineer` | `infra/**`、`.github/workflows/**` | [.github/agents/deploy-engineer.agent.md](.github/agents/deploy-engineer.agent.md) |

跨 Agent 共享的知识放在 [.github/skills/](.github/skills/)。[.github/prompts/](.github/prompts/) 中的工作流提示词把这些 Agent 串起来：`/feature-from-issue`、`/spec-to-code`、`/ship-it`。

> ⚠️ 不要混淆两个层次：
> - **应用 Agent**（`InventoryAgent`、`OrchestratorAgent` …）—— 你在这些实验中部署的运行时舰队。
> - **GitHub Copilot 自定义 Coding Agents**（`/requirements-analyst` …）—— 在开发期*编写*这些应用 Agent 的团队。

**如果你要扩展应用**（见实验 04 的 Day-2 步骤）：按照 [AGENTS.md](AGENTS.md) §1.1，调用其 `Owns` 一列匹配你所改路径的 `/<agent>`，而不是自由发挥地提示。

---

## 📚 Microsoft Learn 知识地图

以下按关注点分组。每个链接同时也嵌在使用它的那个实验里。

### 平台基础（实验 01）

- [Azure 容器注册表简介](https://learn.microsoft.com/zh-cn/azure/container-registry/container-registry-intro)
- [Azure Key Vault 概述](https://learn.microsoft.com/zh-cn/azure/key-vault/general/overview)
- [Azure 资源的托管标识](https://learn.microsoft.com/zh-cn/entra/identity/managed-identities-azure-resources/overview)
- [Microsoft Defender for Cloud](https://learn.microsoft.com/zh-cn/azure/defender-for-cloud/defender-for-cloud-introduction)
- [使用 OIDC 将 GitHub Actions 连接到 Azure](https://learn.microsoft.com/zh-cn/azure/developer/github/connect-from-azure-openid-connect)
- [工作负载标识联合](https://learn.microsoft.com/zh-cn/entra/workload-id/workload-identity-federation)

### Container Apps + Bicep（实验 02）

- [Azure Container Apps 概述](https://learn.microsoft.com/zh-cn/azure/container-apps/overview)
- [使用 Container Apps 和 Bicep 构建微服务](https://learn.microsoft.com/zh-cn/azure/container-apps/microservices-bicep)
- [ACR 任务](https://learn.microsoft.com/zh-cn/azure/container-registry/container-registry-tasks-overview)
- [Azure Container Apps 中的托管标识](https://learn.microsoft.com/zh-cn/azure/container-apps/managed-identity)
- [在 Container Apps 中引用 Key Vault 机密](https://learn.microsoft.com/zh-cn/azure/container-apps/manage-secrets)
- [Container Apps 运行状况探针](https://learn.microsoft.com/zh-cn/azure/container-apps/health-probes)
- [在 Container Apps 中设置缩放规则](https://learn.microsoft.com/zh-cn/azure/container-apps/scale-app)
- [Bicep 模块](https://learn.microsoft.com/zh-cn/azure/azure-resource-manager/bicep/modules)

### AKS、Entra ID 与 Helm（实验 03）

- [Azure Kubernetes 服务 (AKS) 概述](https://learn.microsoft.com/zh-cn/azure/aks/intro-kubernetes)
- [AKS 核心概念](https://learn.microsoft.com/zh-cn/azure/aks/core-aks-concepts)
- [AKS 体系结构指南](https://learn.microsoft.com/zh-cn/azure/architecture/reference-architectures/containers/aks-start-here)
- [AKS 登陆区加速器](https://learn.microsoft.com/zh-cn/azure/cloud-adoption-framework/scenarios/app-platform/aks/landing-zone-accelerator)
- [AKS 的 Microsoft Entra ID 集成](https://learn.microsoft.com/zh-cn/azure/aks/enable-authentication-microsoft-entra-id)
- [将 Azure RBAC 用于 Kubernetes 授权](https://learn.microsoft.com/zh-cn/azure/aks/manage-azure-rbac)
- [AKS 工作负载标识](https://learn.microsoft.com/zh-cn/azure/aks/workload-identity-overview)
- [AKS 上的 Secrets Store CSI 驱动](https://learn.microsoft.com/zh-cn/azure/aks/csi-secrets-store-driver)
- [Azure CNI Overlay 网络](https://learn.microsoft.com/zh-cn/azure/aks/azure-cni-overlay)
- [AKS 的 Azure Policy](https://learn.microsoft.com/zh-cn/azure/aks/use-azure-policy)
- [Microsoft Defender for Containers](https://learn.microsoft.com/zh-cn/azure/defender-for-cloud/defender-for-containers-introduction)
- [容器见解](https://learn.microsoft.com/zh-cn/azure/azure-monitor/containers/container-insights-overview)
- [AKS Helm 快速入门](https://learn.microsoft.com/zh-cn/azure/aks/quickstart-helm)

### 发布工程（实验 04）

- [Bicep `what-if` 部署](https://learn.microsoft.com/zh-cn/azure/azure-resource-manager/bicep/deploy-what-if)
- [从 GitHub Actions 部署 Bicep](https://learn.microsoft.com/zh-cn/azure/azure-resource-manager/bicep/deploy-github-actions)
- [部署与集群可靠性最佳实践](https://learn.microsoft.com/zh-cn/azure/aks/best-practices-app-cluster-reliability)
- [Container Apps 修订与流量](https://learn.microsoft.com/zh-cn/azure/container-apps/revisions)
- [容器见解查询示例](https://learn.microsoft.com/zh-cn/azure/azure-monitor/containers/container-insights-log-query)
- [AKS 安全基线](https://learn.microsoft.com/zh-cn/security/benchmark/azure/baselines/aks-security-baseline)

### 应用技术栈（延伸阅读）

- [Microsoft Agent Framework](https://learn.microsoft.com/zh-cn/agent-framework/)
- [使用自定义 Agent 定制 GitHub Copilot Chat](https://docs.github.com/zh/copilot/customizing-copilot/about-customizing-github-copilot-chat-responses)
- [Python 的 `DefaultAzureCredential`](https://learn.microsoft.com/zh-cn/python/api/overview/azure/identity-readme)
- [使用 OpenTelemetry 为 AI 应用提供可观测性](https://learn.microsoft.com/zh-cn/azure/azure-monitor/app/opentelemetry-overview)

---

## ✅ 前置条件

- 拥有订阅或资源组 **Owner** 权限的 Azure 订阅（你需要创建角色分配和联合凭据）
- 创建 **Microsoft Entra ID 组** 的权限
- Azure CLI ≥ 2.65（`az`），以及 `kubectl`、`helm`、`docker`、`git`、`jq`、`curl`、`gh`
- **`uv`** 与 Python **3.11+**
- 一个 **GitHub Copilot** 订阅，以及一个具备 Copilot 读取权限的细粒度 PAT
- 一个可以设置仓库机密与变量的 GitHub 账号

从 [实验 01](./labs/lab-01-deployment-foundation/README.zh.md) 开始 —— 它会在你花任何钱之前先校验以上全部条件。

---

## 📂 仓库结构

```
.
├── AGENTS.md                        # 面向 AI 编码 Agent 的团队规范
├── pyproject.toml                    # uv/ruff/pyright/pytest/poe 配置
├── docker-compose.yml                # 实验 01 使用的本地 9 服务舰队
├── .github/
│   ├── copilot-instructions.md      # 常驻 Copilot 上下文
│   ├── agents/                      # 6 个 Copilot 自定义 Coding Agent（*.agent.md）
│   ├── skills/                      # Agent 共享的知识
│   ├── prompts/                     # 工作流提示词（/feature-from-issue、/spec-to-code、/ship-it）
│   ├── instructions/                # 分域 *.instructions.md（python、k8s、agent-framework）
│   └── workflows/                   # CI 与 OIDC 联合的 CD
├── labs/
│   ├── lab-01-deployment-foundation/ # 接收交付、本地运行、搭建 Azure 基础
│   ├── lab-02-deploy-container-apps/ # ACR Tasks + Bicep -> Azure Container Apps
│   ├── lab-03-deploy-aks-cluster/    # AKS + Entra ID + Workload Identity + Helm
│   └── lab-04-deploy-production/     # 真实发布、评测、CD、Day-2 运维
├── specs/                           # Coding Agent 依据的规格说明
├── src/
│   ├── Dockerfile.base               # 应用 Agent 的共享基础镜像
│   ├── agents/                      # ZavaShop 应用 Agent（每个一个目录）
│   │   ├── orchestrator/             # AKS 编排器：/plan + /invoke
│   │   ├── inventory/                # ACA 专家：缺货处置
│   │   ├── supplier/                 # ACA 专家：采购单起草
│   │   ├── logistics/                # ACA 专家：发运规划
│   │   └── pricing/                  # ACA 专家：定价建议
│   ├── mcp_servers/                 # 提供 /mcp 的 FastMCP 工具服务器
│   │   ├── inventory/
│   │   ├── supplier/
│   │   ├── shipping/
│   │   └── pricing/
│   └── shared/                      # 设置、遥测、Copilot 客户端、A2A 服务工厂
├── infra/
│   ├── aks/                         # Helm Chart + Workload Identity 文档
│   └── aca/                         # ACA Bicep 模块 + deploy.sh + FQDN 导出
└── tests/                           # 单元 · 集成 · 线上评测
```

## 许可证

MIT
