# 实验 04 — 部署到生产：Bicep + Helm + GitHub Actions

> ⏱ **快速通道约 30 分钟 · 完整实验约 75 分钟** · 用 **Bicep** 在两个平面上执行完整的生产发布，用 Helm 把编排器落到 AKS，做冒烟测试，对真实 Azure 环境运行黄金评测集，最后把整套流程交给一条无密钥的 CI 流水线。

🇬🇧 [English](./README.md)

### ⏱ 想在 2 小时内跑完整个系列？

| | 步骤 | 预算 |
|---|---|---|
| **核心** | 1、2、4、5、6、7、9 | 约 30 分钟 |
| *可选* | 3 what-if 预览 · 8 可观测性 · 10 Day-2 局部发布 · 11 成本与伸缩姿态 | 约 45 分钟 |

> 即使走快速通道也不要跳过最后的 **资源清理** —— AKS 集群和 Container Apps 环境是按小时计费的。

## 场景故事

一切就绪。ACR 有十个镜像。Container Apps 跑着八个服务。AKS 已带着 Entra ID、Azure RBAC、Workload Identity 预配完成，Helm Chart 也已渲染并验证过。

现在来真的 —— 就像周五下午拿到变更窗口时那样：

1. **门禁** —— 登陆区管控项不达标就拒绝部署。
2. **固定** —— 两个平面共用同一个不可变 git SHA。
3. **预览** —— 在任何变更前对 Bicep 执行 `what-if`。
4. **发布** —— ACA 平面用 Bicep，AKS 平面用 Helm。
5. **验证** —— 冒烟 `/healthz`、执行 `POST /plan`、运行评测集。
6. **自动化** —— 用 OIDC 从 GitHub Actions 执行同样的发布，零密钥。
7. **运维** —— Day-2 局部发布、回滚、拆除。

---

## 本实验相关的 Microsoft Learn 知识

- [Bicep `what-if` 部署](https://learn.microsoft.com/zh-cn/azure/azure-resource-manager/bicep/deploy-what-if) —— 永远不能省略的预览步骤。
- [从 GitHub Actions 部署 Bicep](https://learn.microsoft.com/zh-cn/azure/azure-resource-manager/bicep/deploy-github-actions) —— `deploy.yml` 中的 CD 模式。
- [使用 OIDC 将 GitHub Actions 连接到 Azure](https://learn.microsoft.com/zh-cn/azure/developer/github/connect-from-azure-openid-connect) —— 仓库中不存放客户端密钥。
- [使用 Helm 将应用程序部署到 AKS](https://learn.microsoft.com/zh-cn/azure/aks/quickstart-helm) —— `helm upgrade --install` 的语义。
- [部署与集群可靠性最佳实践](https://learn.microsoft.com/zh-cn/azure/aks/best-practices-app-cluster-reliability) —— 探针、PDB、发布策略。
- [Container Apps 修订与流量](https://learn.microsoft.com/zh-cn/azure/container-apps/revisions) —— 局部更新的行为。
- [Container Apps 零停机部署](https://learn.microsoft.com/zh-cn/azure/container-apps/revisions-manage) —— 单修订与多修订模式。
- [容器见解查询示例](https://learn.microsoft.com/zh-cn/azure/azure-monitor/containers/container-insights-log-query) —— 可观测性检查中使用的 KQL。
- [AKS 安全基线](https://learn.microsoft.com/zh-cn/security/benchmark/azure/baselines/aks-security-baseline) —— 门禁断言的管控项。

---

## 前置条件

```bash
cd /path/to/AKS-Lab-GitHubCopilot
source .env.lab
source .env.fqdns

kubectl config current-context      # 你的 AKS 集群
az account show --query name -o tsv
```

---

## 步骤 1 — 执行登陆区门禁

本实验遵循的规范禁止在下列任一条件不满足时发布。全部执行；任何不匹配都终止部署。

```bash
echo "== AKS 管控项 =="
az aks show -g $RG -n $AKS --query aadProfile.managed -o tsv
az aks show -g $RG -n $AKS --query aadProfile.enableAzureRBAC -o tsv
az aks show -g $RG -n $AKS --query addonProfiles.azurepolicy.enabled -o tsv
az aks show -g $RG -n $AKS \
  --query securityProfile.defender.securityMonitoring.enabled -o tsv
az aks show -g $RG -n $AKS --query oidcIssuerProfile.enabled -o tsv
az aks show -g $RG -n $AKS \
  --query securityProfile.workloadIdentity.enabled -o tsv

echo "== Defender 计划 =="
az security pricing show -n Containers --query pricingTier -o tsv
az security pricing show -n KeyVaults  --query pricingTier -o tsv

echo "== 身份 =="
az role assignment list --assignee "$UAMI_PRINCIPAL_ID" --all \
  --query "[].{role:roleDefinitionName, scope:scope}" -o table
az identity federated-credential list \
  --identity-name $UAMI -g $RG \
  --query "[].{name:name, subject:subject}" -o table
```

期望的角色表：

| 角色 | 范围 |
|---|---|
| `AcrPull` | ACR |
| `Contributor` | 资源组 |
| `Key Vault Secrets User` | Key Vault |
| `Azure Kubernetes Service Cluster User Role` | AKS 集群 |

期望的联合凭据：`gha-aks-lab-main` 和 `fc-orchestrator`。

不可变标签门禁 —— 必须无输出：

```bash
grep -rn ':latest' infra/ .github/workflows/ || echo "no :latest — ok"
```

代码质量门禁：

```bash
git status --porcelain          # 为空
uv run poe check                # 全绿
```

---

## 步骤 2 — 固定发布 SHA

```bash
export GIT_SHA=$(git rev-parse --short HEAD)
export ACR_LOGIN_SERVER="${ACR}.azurecr.io"
echo "releasing $GIT_SHA"
```

确认十个镜像在该标签下都存在（如果实验 02 是在这个 commit 上执行的就一定存在）：

```bash
for repo in base orchestrator inventory supplier logistics pricing \
            inventory-mcp supplier-mcp shipping-mcp pricing-mcp; do
  printf '%-18s ' "$repo"
  az acr repository show-tags -n $ACR --repository zavashop/$repo -o tsv \
    | grep -qx "$GIT_SHA" && echo "ok" || echo "MISSING"
done
```

如有缺失，重新构建：

```bash
az acr build -r $ACR --platform linux/amd64 \
  -t zavashop/base:$GIT_SHA -f src/Dockerfile.base .

for s in inventory supplier shipping pricing; do
  az acr build -r $ACR --platform linux/amd64 \
    -t zavashop/${s}-mcp:$GIT_SHA -f src/mcp_servers/${s}/Dockerfile .
done

for s in orchestrator inventory supplier logistics pricing; do
  az acr build -r $ACR --platform linux/amd64 \
    -t zavashop/${s}:$GIT_SHA \
    --build-arg BASE_IMAGE="$ACR_LOGIN_SERVER/zavashop/base:$GIT_SHA" \
    -f src/agents/${s}/Dockerfile .
done
```

---

## 步骤 3 — 用 `what-if` 预览 ACA 平面 *（可选）*

绝不盲目应用 Bicep。先预览编排器的下游依赖：

```bash
az deployment group what-if \
  -g $RG \
  -f infra/aca/agent.bicep \
  -p name=inventory \
  -p envId="$CAE_ID" \
  -p image="$ACR_LOGIN_SERVER/zavashop/inventory:$GIT_SHA" \
  -p registryServer="$ACR_LOGIN_SERVER" \
  -p uamiId="$UAMI_RESOURCE_ID" \
  -p uamiClientId="$UAMI_CLIENT_ID" \
  -p keyVaultName="$KV" \
  -p targetPort=8000 \
  -p exposeIngress=true \
  -p minReplicas=1 \
  -p mcpEndpoints="{\"ZAVA_INVENTORY_MCP_URL\":\"$ZAVA_INVENTORY_MCP_URL\"}"
```

阅读差异：

- `properties.template.containers[0].image` 上的 `~ Modify` —— SHA 变了，符合预期。
- `configuration.secrets` 上出现带字面值的 `~ Modify` —— **停止**，这是泄漏。
- 任何 `- Delete` —— **停止**，说明缺少参数。

---

## 步骤 4 — 用 Bicep 发布 ACA 平面

```bash
DEPLOY_SHA=$GIT_SHA bash infra/aca/deploy.sh
```

脚本按依赖顺序部署全部八个应用并重写 `.env.fqdns`。重新加载：

```bash
source .env.fqdns
cat .env.fqdns
```

确认每个应用都切到了新 SHA：

```bash
for app in inventory-mcp supplier-mcp shipping-mcp pricing-mcp \
           inventory supplier logistics pricing; do
  printf '%-16s ' "$app"
  az containerapp show -g $RG -n $app \
    --query 'properties.template.containers[0].image' -o tsv
done
```

每一行都必须以 `:$GIT_SHA` 结尾。

冒烟全部八个：

```bash
for app in inventory-mcp supplier-mcp shipping-mcp pricing-mcp \
           inventory supplier logistics pricing; do
  fqdn=$(az containerapp show -g $RG -n $app \
    --query properties.configuration.ingress.fqdn -o tsv)
  printf '%-16s ' "$app"
  curl -fsS -o /dev/null -w '%{http_code}\n' "https://$fqdn/readyz"
done
```

---

## 步骤 5 — 用 Helm 发布 AKS 平面

同一个 SHA、同一个身份、不同的运行时。

```bash
helm upgrade --install zavashop infra/aks/helm/zavashop \
  -n zavashop --create-namespace \
  --set image.repository="$ACR_LOGIN_SERVER/zavashop/orchestrator" \
  --set image.tag="$GIT_SHA" \
  --set workloadIdentity.clientId="$UAMI_CLIENT_ID" \
  --set keyVault.name="$KV" \
  --set tenantId="$AZURE_TENANT_ID" \
  --set a2a.inventory="$ZAVA_INVENTORY_URL" \
  --set a2a.supplier="$ZAVA_SUPPLIER_URL" \
  --set a2a.logistics="$ZAVA_LOGISTICS_URL" \
  --set a2a.pricing="$ZAVA_PRICING_URL" \
  --wait --timeout 10m
```

观察发布过程：

```bash
kubectl -n zavashop rollout status deploy/orchestrator --timeout=300s
kubectl -n zavashop get pods -o wide
kubectl -n zavashop get svc orchestrator
```

验证身份链路已落地：

```bash
kubectl -n zavashop get sa orchestrator-sa -o yaml \
  | grep azure.workload.identity/client-id

kubectl -n zavashop get secretproviderclass -o name

kubectl -n zavashop exec deploy/orchestrator -- \
  ls -l /mnt/secrets-store
```

确认没有令牌泄漏进 Pod 规格：

```bash
kubectl -n zavashop get deploy orchestrator -o yaml \
  | grep -i 'GITHUB_TOKEN'
```

唯一命中的应该是读取 CSI 文件的那条 shell 命令。

---

## 步骤 6 — 对线上系统做冒烟测试

```bash
export ORCH_IP=$(kubectl -n zavashop get svc orchestrator \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

while [ -z "$ORCH_IP" ]; do
  sleep 10
  export ORCH_IP=$(kubectl -n zavashop get svc orchestrator \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
done

echo "orchestrator at http://$ORCH_IP"
```

```bash
curl -fsS "http://$ORCH_IP/healthz"
curl -fsS "http://$ORCH_IP/readyz"
```

真正的业务调用 —— 一次请求打通 AKS → 4 个 ACA 专家 → 4 个 ACA MCP 服务器 → Copilot `gpt-5.5`：

```bash
curl -fsS -X POST "http://$ORCH_IP/plan" \
  -H 'content-type: application/json' \
  -d '{
        "goal": "Store 101 will stock out of SKU ZS-1042 by Friday.",
        "sku": "ZS-1042",
        "store_id": "store-101"
      }' | tee /tmp/plan.json
```

断言计划完整：

```bash
python - <<'PY'
import json
plan = json.load(open("/tmp/plan.json"))
for key in ("stock_view", "po_view", "shipping_view",
            "price_view", "summary", "next_actions"):
    assert plan.get(key), f"missing or empty: {key}"
print("plan complete:", len(plan["next_actions"]), "next actions")
PY
```

---

## 步骤 7 — 对真实 Azure 环境运行黄金评测集

交付的仓库自带评测框架和黄金场景集。这是对**部署**的验收测试，而不只是对代码的。

```bash
sed -n '1,40p' tests/evals/run_evals.py
wc -l tests/evals/scenarios.jsonl
```

```bash
ZAVA_ENDPOINT="http://$ORCH_IP" \
ZAVA_EVAL_LATENCY_BUDGET=400 \
uv run python -m tests.evals.run_evals
```

期望输出结尾：

```
failures=0
```

如果只是延迟不达标而内容通过，说明集群是健康的、只是预算太紧 —— 提高 `ZAVA_EVAL_LATENCY_BUDGET` 并记录真实数值。如果是内容失败，说明某个专家或 MCP 服务器配置有误，检查 `kubectl -n zavashop logs deploy/orchestrator --tail=100`。

---

## 步骤 8 — 可观测性检查 *（可选）*

```bash
kubectl -n zavashop logs deploy/orchestrator --tail=50
```

结构化的 `structlog` JSON 行携带 `agent.name`、`agent.run_id` 和 `agent.span_id`。

从容器见解查询同样的数据：

```bash
az monitor log-analytics query \
  -w $(az monitor log-analytics workspace show -g $RG -n $LAW \
        --query customerId -o tsv) \
  --analytics-query "ContainerLogV2
    | where PodName startswith 'orchestrator'
    | project TimeGenerated, LogMessage
    | order by TimeGenerated desc
    | take 20" \
  -o table
```

ACA 日志：

```bash
az containerapp logs show -g $RG -n inventory --tail 50
```

---

## 步骤 9 — 启用 CD 流水线

你刚才手工做的一切都写在 `.github/workflows/deploy.yml` 里。

```bash
sed -n '1,60p' .github/workflows/deploy.yml
```

让它无密钥的关键要素：

| 要素 | 原因 |
|---|---|
| `permissions: id-token: write` | 允许作业申请 OIDC 令牌 |
| `azure/login@v2` 配合 `client-id`/`tenant-id`/`subscription-id` | 通过实验 01 的联合凭据交换 OIDC 令牌 |
| `actions/setup-python@v5`（3.13）在 `astral-sh/setup-uv@v3` **之前** | `uv sync` 需要解释器已存在 |
| `GIT_SHA=${{ github.sha }}` | 与手工发布相同的不可变标签规则 |

确认实验 01 步骤 14 的机密与变量已就位：

```bash
gh secret list --repo "$GH_REPO_SLUG"
gh variable list --repo "$GH_REPO_SLUG"
```

触发它：

```bash
gh workflow run deploy.yml --repo "$GH_REPO_SLUG" --ref main
gh run watch --repo "$GH_REPO_SLUG"
```

绿色的运行结果意味着队友无需在自己电脑上持有任何 Azure 凭据就能发布 ZavaShop。

---

## 步骤 10 — Day-2：局部发布 *（可选）*

真实的变更很少同时涉及十个服务。改一个提示词，只发布那一个镜像。

```bash
# 做一处小而安全的修改
$EDITOR src/agents/pricing/prompts.py

git add -A
git commit -m "feat(pricing): tighten markdown guidance"
export NEW_SHA=$(git rev-parse --short HEAD)
```

只构建并发布 `pricing`：

```bash
az acr build -r $ACR --platform linux/amd64 \
  -t zavashop/pricing:$NEW_SHA \
  --build-arg BASE_IMAGE="$ACR_LOGIN_SERVER/zavashop/base:$GIT_SHA" \
  -f src/agents/pricing/Dockerfile .

az containerapp update -g $RG -n pricing \
  --image "$ACR_LOGIN_SERVER/zavashop/pricing:$NEW_SHA"
```

观察修订切换：

```bash
az containerapp revision list -g $RG -n pricing \
  --query "[].{name:name, active:properties.active, \
              traffic:properties.trafficWeight, \
              image:properties.template.containers[0].image}" -o table
```

再跑一次计划，确认没有其他回归：

```bash
curl -fsS -X POST "http://$ORCH_IP/plan" \
  -H 'content-type: application/json' \
  -d '{"goal":"Post-change verification","sku":"ZS-1042","store_id":"store-101"}' \
  | python -c 'import json,sys; print(json.load(sys.stdin)["summary"])'
```

不满意就回滚：

```bash
az containerapp update -g $RG -n pricing \
  --image "$ACR_LOGIN_SERVER/zavashop/pricing:$GIT_SHA"
```

AKS 侧回滚：

```bash
helm history zavashop -n zavashop
helm rollback zavashop -n zavashop
kubectl -n zavashop rollout status deploy/orchestrator
```

---

## 步骤 11 — 成本与伸缩姿态 *（可选）*

```bash
kubectl -n zavashop get hpa 2>/dev/null
kubectl top nodes 2>/dev/null
kubectl top pods -n zavashop 2>/dev/null

az containerapp show -g $RG -n pricing \
  --query properties.template.scale -o json
```

值得记住的生产调优（实验中不要应用）：

- 测量冷启动预算后，把 ACA 的 `minReplicas` 设为 `0`。
- 在用户节点池上启用 AKS 集群自动缩放器。
- 启用节点自动升级前，为编排器添加 `PodDisruptionBudget`。

---

## ✅ 达成以下条件即完成实验 04

- [ ] 步骤 1 的所有门禁通过，且 `grep -rn ':latest'` 无输出。
- [ ] 十个镜像在 `$GIT_SHA` 下均存在。
- [ ] `what-if` 只显示镜像标签变更，未显示任何机密值。
- [ ] 8 个 Container App 全部运行 `:$GIT_SHA` 且 `/readyz` 返回 `200`。
- [ ] `helm upgrade --install` 成功且编排器发布完成。
- [ ] `orchestrator-sa` 带有 Workload Identity 注解，Pod 能看到 `/mnt/secrets-store/GITHUB-TOKEN`。
- [ ] `POST /plan` 返回了全部六个计划字段。
- [ ] `tests.evals.run_evals` 报告 `failures=0`。
- [ ] 容器见解能返回编排器的日志行。
- [ ] `deploy.yml` 在 GitHub Actions 中无客户端密钥地跑绿。
- [ ] 局部发布产生了新的 ACA 修订并成功回滚。

---

## 拆除

所有资源都在一个资源组里。

```bash
source .env.lab
az group delete -n $RG --yes --no-wait
```

同时删除生命周期长于资源组的身份对象：

```bash
az ad group delete --group $AKS_ADMINS_GROUP_ID
```

本地清理：

```bash
kubectl config delete-context $AKS 2>/dev/null
rm -f .env.lab .env.fqdns
```

完成后请在 <https://github.com/settings/personal-access-tokens> 轮换那个细粒度 PAT。

---

## 故障排查

| 现象 | 处理 |
|---|---|
| `helm upgrade` 超时 | `kubectl -n zavashop describe pod -l app=orchestrator` —— 通常是镜像拉取或 CSI 挂载问题。 |
| Pod 报 `CreateContainerConfigError` | `SecretProviderClass` 失败。检查 `keyVault.name` 和 `tenantId`。 |
| Pod 报 `ImagePullBackOff` | `--attach-acr` 未生效，或标签不存在。重新执行 `az aks update --attach-acr $ACR`。 |
| LoadBalancer IP 一直是 `<pending>` | 公网 IP 配额不足。`kubectl -n zavashop describe svc orchestrator`。 |
| `/plan` 返回 502 | `.env.fqdns` 中某个专家 FQDN 过期。重新 source 并重跑 `helm upgrade`。 |
| `/plan` 首次调用非常慢 | Copilot SDK 预热加上 ACA 冷启动。先请求各专家的 `/healthz`。 |
| 评测只在延迟上失败 | 提高 `ZAVA_EVAL_LATENCY_BUDGET` 并记录实测值。 |
| 评测在内容上失败 | `kubectl -n zavashop logs deploy/orchestrator --tail=100`；检查每个专家的 MCP URL。 |
| GitHub Actions 报 `AADSTS70021` | 联合凭据 subject 不匹配 —— 必须是 `repo:OWNER/REPO:ref:refs/heads/main`。 |
| GitHub Actions 对 ACR 报 `could not be found` | UAMI 缺少资源组上的 `Contributor`（实验 01 步骤 8）。 |
| CI 中 `uv sync` 失败 | `actions/setup-python` 必须在 `astral-sh/setup-uv` 之前执行。 |
