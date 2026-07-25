# Lab 04 — Deploy to Production: Bicep + Helm + GitHub Actions

> ⏱ **Fast track ~30 min · Full lab ~75 min** · Run the full production rollout with **Bicep** across both planes, land the orchestrator on AKS with Helm, smoke-test it, run the golden eval suite against live Azure, then hand the whole thing to a secret-less CI pipeline.

🇨🇳 [中文版](./README.zh.md)

### ⏱ Running the whole series in 2 hours?

| | Steps | Budget |
|---|---|---|
| **Core** | 1, 2, 4, 5, 6, 7, 9 | ~30 min |
| *Optional* | 3 what-if preview · 8 observability · 10 Day-2 partial rollout · 11 cost & scale posture | ~45 min |

> Do not skip **Teardown** at the end, even on the fast track — the AKS cluster
> and Container Apps environment bill by the hour.

## The story

Everything is in place. ACR holds ten images. Container Apps runs eight
services. AKS is provisioned with Entra ID, Azure RBAC, Workload Identity, and
a Helm chart you have already rendered and validated.

Now do it for real — the way you would on a Friday afternoon with a change
window open:

1. **Gate** — refuse to deploy unless the landing-zone controls hold.
2. **Pin** — one immutable git SHA for both planes.
3. **Preview** — `what-if` on Bicep before anything changes.
4. **Roll** — Bicep for the ACA plane, Helm for the AKS plane.
5. **Prove** — smoke `/healthz`, exercise `POST /plan`, run the eval suite.
6. **Automate** — the same rollout from GitHub Actions with OIDC, no secrets.
7. **Operate** — Day-2 partial rollout, rollback, teardown.

---

## Microsoft Learn knowledge for this lab

- [Bicep `what-if` deployments](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deploy-what-if) — the preview step you never skip.
- [Deploy Bicep from GitHub Actions](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deploy-github-actions) — the CD pattern in `deploy.yml`.
- [Connect GitHub Actions to Azure with OIDC](https://learn.microsoft.com/azure/developer/github/connect-from-azure-openid-connect) — no client secret in the repo.
- [Deploy an application with Helm to AKS](https://learn.microsoft.com/azure/aks/quickstart-helm) — `helm upgrade --install` semantics.
- [Deployment and cluster reliability best practices](https://learn.microsoft.com/azure/aks/best-practices-app-cluster-reliability) — probes, PDBs, rollout strategy.
- [Container Apps revisions and traffic](https://learn.microsoft.com/azure/container-apps/revisions) — how a partial update behaves.
- [Container Apps zero-downtime deployment](https://learn.microsoft.com/azure/container-apps/revisions-manage) — single vs. multiple revision mode.
- [Container Insights query examples](https://learn.microsoft.com/azure/azure-monitor/containers/container-insights-log-query) — the KQL used in the observability check.
- [AKS security baseline](https://learn.microsoft.com/security/benchmark/azure/baselines/aks-security-baseline) — the controls the gate asserts.

---

## Prerequisites

```bash
cd /path/to/AKS-Lab-GitHubCopilot
source .env.lab
source .env.fqdns

kubectl config current-context      # your AKS cluster
az account show --query name -o tsv
```

---

## Step 1 — Run the landing-zone gate

The spec these labs follow forbids a rollout unless every one of these is true.
Run them all; any mismatch stops the deployment.

```bash
echo "== AKS controls =="
az aks show -g $RG -n $AKS --query aadProfile.managed -o tsv
az aks show -g $RG -n $AKS --query aadProfile.enableAzureRBAC -o tsv
az aks show -g $RG -n $AKS --query addonProfiles.azurepolicy.enabled -o tsv
az aks show -g $RG -n $AKS \
  --query securityProfile.defender.securityMonitoring.enabled -o tsv
az aks show -g $RG -n $AKS --query oidcIssuerProfile.enabled -o tsv
az aks show -g $RG -n $AKS \
  --query securityProfile.workloadIdentity.enabled -o tsv

echo "== Defender plans =="
az security pricing show -n Containers --query pricingTier -o tsv
az security pricing show -n KeyVaults  --query pricingTier -o tsv

echo "== Identity =="
az role assignment list --assignee "$UAMI_PRINCIPAL_ID" --all \
  --query "[].{role:roleDefinitionName, scope:scope}" -o table
az identity federated-credential list \
  --identity-name $UAMI -g $RG \
  --query "[].{name:name, subject:subject}" -o table
```

Expected role table:

| Role | Scope |
|---|---|
| `AcrPull` | the ACR |
| `Contributor` | the resource group |
| `Key Vault Secrets User` | the Key Vault |
| `Azure Kubernetes Service Cluster User Role` | the AKS cluster |

Expected federated credentials: `gha-aks-lab-main` and `fc-orchestrator`.

Immutable-tag gate — must print nothing:

```bash
grep -rn ':latest' infra/ .github/workflows/ || echo "no :latest — ok"
```

Code-quality gate:

```bash
git status --porcelain          # empty
uv run poe check                # green
```

---

## Step 2 — Pin the release SHA

```bash
export GIT_SHA=$(git rev-parse --short HEAD)
export ACR_LOGIN_SERVER="${ACR}.azurecr.io"
echo "releasing $GIT_SHA"
```

Confirm all ten images exist at this tag (they do if Lab 02 ran on this commit):

```bash
for repo in base orchestrator inventory supplier logistics pricing \
            inventory-mcp supplier-mcp shipping-mcp pricing-mcp; do
  printf '%-18s ' "$repo"
  az acr repository show-tags -n $ACR --repository zavashop/$repo -o tsv \
    | grep -qx "$GIT_SHA" && echo "ok" || echo "MISSING"
done
```

If anything is missing, rebuild it:

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

## Step 3 — Preview the ACA plane with `what-if` *(optional)*

Never apply Bicep blind. Preview the orchestrator's downstream dependency first:

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

Read the diff:

- `~ Modify` on `properties.template.containers[0].image` — the SHA changed. Expected.
- Any `~ Modify` on `configuration.secrets` showing a literal value — **stop**, that is a leak.
- `- Delete` on anything — **stop**, a parameter is missing.

---

## Step 4 — Roll the ACA plane with Bicep

```bash
DEPLOY_SHA=$GIT_SHA bash infra/aca/deploy.sh
```

The script deploys all eight apps in dependency order and rewrites
`.env.fqdns`. Reload it:

```bash
source .env.fqdns
cat .env.fqdns
```

Confirm every app moved to the new SHA:

```bash
for app in inventory-mcp supplier-mcp shipping-mcp pricing-mcp \
           inventory supplier logistics pricing; do
  printf '%-16s ' "$app"
  az containerapp show -g $RG -n $app \
    --query 'properties.template.containers[0].image' -o tsv
done
```

Every line must end in `:$GIT_SHA`.

Smoke all eight:

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

## Step 5 — Roll the AKS plane with Helm

Same SHA, same identity, different runtime.

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

Watch the rollout:

```bash
kubectl -n zavashop rollout status deploy/orchestrator --timeout=300s
kubectl -n zavashop get pods -o wide
kubectl -n zavashop get svc orchestrator
```

Verify the identity plumbing landed:

```bash
kubectl -n zavashop get sa orchestrator-sa -o yaml \
  | grep azure.workload.identity/client-id

kubectl -n zavashop get secretproviderclass -o name

kubectl -n zavashop exec deploy/orchestrator -- \
  ls -l /mnt/secrets-store
```

Confirm no token leaked into the pod spec:

```bash
kubectl -n zavashop get deploy orchestrator -o yaml \
  | grep -i 'GITHUB_TOKEN' 
```

The only hit should be the shell command that reads the CSI file.

---

## Step 6 — Smoke the live system

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

The real business call — AKS → 4 ACA specialists → 4 ACA MCP servers →
Copilot `gpt-5.5`, all in one request:

```bash
curl -fsS -X POST "http://$ORCH_IP/plan" \
  -H 'content-type: application/json' \
  -d '{
        "goal": "Store 101 will stock out of SKU ZS-1042 by Friday.",
        "sku": "ZS-1042",
        "store_id": "store-101"
      }' | tee /tmp/plan.json
```

Assert the plan is complete:

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

## Step 7 — Run the golden eval suite against live Azure

The delivered repo ships an eval harness and a golden scenario set. This is the
acceptance test for the deployment, not just for the code.

```bash
sed -n '1,40p' tests/evals/run_evals.py
wc -l tests/evals/scenarios.jsonl
```

```bash
ZAVA_ENDPOINT="http://$ORCH_IP" \
ZAVA_EVAL_LATENCY_BUDGET=400 \
uv run python -m tests.evals.run_evals
```

Expected tail:

```
failures=0
```

If latency fails but content passes, the cluster is healthy and the budget is
tight — raise `ZAVA_EVAL_LATENCY_BUDGET` and record the real number. If content
fails, a specialist or MCP server is misconfigured; check
`kubectl -n zavashop logs deploy/orchestrator --tail=100`.

---

## Step 8 — Observability check *(optional)*

```bash
kubectl -n zavashop logs deploy/orchestrator --tail=50
```

Structured `structlog` JSON lines carry `agent.name`, `agent.run_id`, and
`agent.span_id`.

Query the same data from Container Insights:

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

ACA logs:

```bash
az containerapp logs show -g $RG -n inventory --tail 50
```

---

## Step 9 — Enable the CD pipeline

Everything you just did by hand exists in `.github/workflows/deploy.yml`.

```bash
sed -n '1,60p' .github/workflows/deploy.yml
```

The pieces that make it secret-less:

| Piece | Why |
|---|---|
| `permissions: id-token: write` | Lets the job request an OIDC token |
| `azure/login@v2` with `client-id`/`tenant-id`/`subscription-id` | Exchanges the OIDC token via the Lab 01 federated credential |
| `actions/setup-python@v5` (3.13) **before** `astral-sh/setup-uv@v3` | `uv sync` needs an interpreter present |
| `GIT_SHA=${{ github.sha }}` | Same immutable-tag rule as your manual rollout |

Confirm the secrets and variables from Lab 01 Step 14 are present:

```bash
gh secret list --repo "$GH_REPO_SLUG"
gh variable list --repo "$GH_REPO_SLUG"
```

Trigger it:

```bash
gh workflow run deploy.yml --repo "$GH_REPO_SLUG" --ref main
gh run watch --repo "$GH_REPO_SLUG"
```

A green run means a teammate can ship ZavaShop without an Azure credential on
their laptop.

---

## Step 10 — Day-2: partial rollout *(optional)*

Real changes rarely touch all ten services. Change one prompt and ship only
that image.

```bash
# make a small, safe edit
$EDITOR src/agents/pricing/prompts.py

git add -A
git commit -m "feat(pricing): tighten markdown guidance"
export NEW_SHA=$(git rev-parse --short HEAD)
```

Build and roll only `pricing`:

```bash
az acr build -r $ACR --platform linux/amd64 \
  -t zavashop/pricing:$NEW_SHA \
  --build-arg BASE_IMAGE="$ACR_LOGIN_SERVER/zavashop/base:$GIT_SHA" \
  -f src/agents/pricing/Dockerfile .

az containerapp update -g $RG -n pricing \
  --image "$ACR_LOGIN_SERVER/zavashop/pricing:$NEW_SHA"
```

Observe the revision swap:

```bash
az containerapp revision list -g $RG -n pricing \
  --query "[].{name:name, active:properties.active, \
              traffic:properties.trafficWeight, \
              image:properties.template.containers[0].image}" -o table
```

Re-run the plan to confirm nothing else regressed:

```bash
curl -fsS -X POST "http://$ORCH_IP/plan" \
  -H 'content-type: application/json' \
  -d '{"goal":"Post-change verification","sku":"ZS-1042","store_id":"store-101"}' \
  | python -c 'import json,sys; print(json.load(sys.stdin)["summary"])'
```

Roll back if you dislike the result:

```bash
az containerapp update -g $RG -n pricing \
  --image "$ACR_LOGIN_SERVER/zavashop/pricing:$GIT_SHA"
```

AKS-side rollback:

```bash
helm history zavashop -n zavashop
helm rollback zavashop -n zavashop
kubectl -n zavashop rollout status deploy/orchestrator
```

---

## Step 11 — Cost and scale posture *(optional)*

```bash
kubectl -n zavashop get hpa 2>/dev/null
kubectl top nodes 2>/dev/null
kubectl top pods -n zavashop 2>/dev/null

az containerapp show -g $RG -n pricing \
  --query properties.template.scale -o json
```

Production tuning to note (do not apply in the lab):

- Set ACA `minReplicas: 0` once cold-start budgets are measured.
- Enable the AKS cluster autoscaler on the user node pool.
- Add a `PodDisruptionBudget` for the orchestrator before enabling node auto-upgrade.

---

## ✅ Lab 04 done when…

- [ ] Every gate in Step 1 passed, and `grep -rn ':latest'` printed nothing.
- [ ] All ten images exist at `$GIT_SHA`.
- [ ] `what-if` showed only image-tag modifications and no secret values.
- [ ] All 8 Container Apps run `:$GIT_SHA` and return `200` on `/readyz`.
- [ ] `helm upgrade --install` succeeded and the orchestrator rollout completed.
- [ ] `orchestrator-sa` carries the Workload Identity annotation and the pod sees `/mnt/secrets-store/GITHUB-TOKEN`.
- [ ] `POST /plan` returned all six plan fields.
- [ ] `tests.evals.run_evals` reported `failures=0`.
- [ ] Container Insights returns orchestrator log lines.
- [ ] `deploy.yml` ran green from GitHub Actions with no client secret.
- [ ] The partial rollout produced a new ACA revision and rolled back cleanly.

---

## Teardown

Everything lives in one resource group.

```bash
source .env.lab
az group delete -n $RG --yes --no-wait
```

Also remove the identity artifacts that outlive the group:

```bash
az ad group delete --group $AKS_ADMINS_GROUP_ID
```

Local cleanup:

```bash
kubectl config delete-context $AKS 2>/dev/null
rm -f .env.lab .env.fqdns
```

Rotate the fine-grained PAT at
<https://github.com/settings/personal-access-tokens> when you are done.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `helm upgrade` times out | `kubectl -n zavashop describe pod -l app=orchestrator` — usually image pull or CSI mount. |
| Pod `CreateContainerConfigError` | The `SecretProviderClass` failed. Check `keyVault.name` and `tenantId`. |
| Pod `ImagePullBackOff` | `--attach-acr` was not applied, or the tag does not exist. Re-run `az aks update --attach-acr $ACR`. |
| LoadBalancer IP stays `<pending>` | Public IP quota. `kubectl -n zavashop describe svc orchestrator`. |
| `/plan` returns 502 | A specialist FQDN in `.env.fqdns` is stale. Re-source and re-run `helm upgrade`. |
| `/plan` is very slow on first call | Copilot SDK warm-up plus ACA cold start. Hit `/healthz` on each specialist first. |
| Evals fail on latency only | Raise `ZAVA_EVAL_LATENCY_BUDGET` and record the measured number. |
| Evals fail on content | `kubectl -n zavashop logs deploy/orchestrator --tail=100`; check each specialist's MCP URL. |
| GitHub Actions `AADSTS70021` | Federated credential subject mismatch — must be `repo:OWNER/REPO:ref:refs/heads/main`. |
| GitHub Actions `could not be found` on ACR | UAMI is missing `Contributor` on the RG (Lab 01 Step 8). |
| `uv sync` fails in CI | `actions/setup-python` must run before `astral-sh/setup-uv`. |
