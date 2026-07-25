# Lab 02 — Deploy to Container Apps: Containerize the Fleet & Ship with Bicep

> ⏱ ~60 min · Build 10 container images from the delivered source, push them to ACR, and deploy the 4 MCP servers + 4 specialist agents to **Azure Container Apps** using a single reusable **Bicep** module.

🇨🇳 [中文版](./README.zh.md)

## The story

Lab 01 gave you a foundation and a working local Compose fleet. But Compose is
not a deployment target — ZavaShop needs the bursty parts of the system to cost
nothing at 3 a.m. and absorb the 5 p.m. replenishment peak.

Eight of the nine services fit that shape: the four **specialist agents**
(inventory, supplier, logistics, pricing) and the four **MCP tool servers**.
They are stateless, HTTP-triggered, and idle most of the day. Azure Container
Apps is the right home: no cluster to operate, KEDA HTTP scaling, managed
identity, and Key Vault-backed secrets.

The orchestrator is different — it is long-lived and fronted by a load balancer.
That is Lab 03 and Lab 04.

By the end of this lab, ZavaShop has **8 live Container Apps** answering
`/healthz`, `/readyz`, and `/invoke`, with the Copilot token delivered from Key
Vault and never written into a Bicep parameter.

---

## Microsoft Learn knowledge for this lab

- [Azure Container Apps overview](https://learn.microsoft.com/azure/container-apps/overview) — the runtime model for the 8 bursty services.
- [Microservices with Container Apps and Bicep](https://learn.microsoft.com/azure/container-apps/microservices-bicep) — the parameterized module pattern reused 8 times.
- [ACR Tasks](https://learn.microsoft.com/azure/container-registry/container-registry-tasks-overview) — `az acr build` gives daemon-free native `linux/amd64` builds.
- [Managed identity in Azure Container Apps](https://learn.microsoft.com/azure/container-apps/managed-identity) — how the app pulls from ACR without a registry password.
- [Reference secrets from Key Vault in Container Apps](https://learn.microsoft.com/azure/container-apps/manage-secrets) — the `keyVaultUrl` + `identity` secret shape.
- [Health probes in Container Apps](https://learn.microsoft.com/azure/container-apps/health-probes) — liveness on `/healthz`, readiness on `/readyz`.
- [Set scaling rules in Container Apps](https://learn.microsoft.com/azure/container-apps/scale-app) — the HTTP concurrency rule.
- [Bicep `what-if` deployments](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deploy-what-if) — preview before you apply.
- [Bicep modules](https://learn.microsoft.com/azure/azure-resource-manager/bicep/modules) — why one `agent.bicep` serves 8 apps.

---

## What you deploy

| # | Container App | Image | Port | Ingress | Env it needs |
|---|---|---|---|---|---|
| 1 | `inventory-mcp` | `zavashop/inventory-mcp` | 8080 | external | — |
| 2 | `supplier-mcp` | `zavashop/supplier-mcp` | 8080 | external | — |
| 3 | `shipping-mcp` | `zavashop/shipping-mcp` | 8080 | external | — |
| 4 | `pricing-mcp` | `zavashop/pricing-mcp` | 8080 | external | — |
| 5 | `inventory` | `zavashop/inventory` | 8000 | external | `ZAVA_INVENTORY_MCP_URL` |
| 6 | `supplier` | `zavashop/supplier` | 8000 | external | `ZAVA_SUPPLIER_MCP_URL` |
| 7 | `logistics` | `zavashop/logistics` | 8000 | external | `ZAVA_SHIPPING_MCP_URL` |
| 8 | `pricing` | `zavashop/pricing` | 8000 | external | `ZAVA_PRICING_MCP_URL` |

Order matters: MCP servers first, because the specialists need their FQDNs.

---

## Prerequisites

```bash
cd /path/to/AKS-Lab-GitHubCopilot
source .env.lab

echo "$RG / $ACR / $CAE / $KV"
az account show --query name -o tsv
```

If `.env.lab` is missing, go back to
[Lab 01](../lab-01-deployment-foundation/README.md).

---

## Step 1 — Understand the image layout before you build

The delivered app builds **one shared base image** plus one thin image per
service. Read the two Dockerfiles first.

```bash
sed -n '1,60p' src/Dockerfile.base
sed -n '1,40p' src/agents/inventory/Dockerfile
sed -n '1,40p' src/mcp_servers/inventory/Dockerfile
```

Key facts you must carry into ACR:

| Fact | Why it matters |
|---|---|
| `src/Dockerfile.base` installs every Python dependency | Service images only copy source — builds after the first are fast. |
| Agent Dockerfiles take a `BASE_IMAGE` build arg | You must pass the **ACR-qualified** base tag, not the local one. |
| Runtime user is `zava`, uid `10001`, non-root | Matches the AKS `podSecurityContext` in Lab 03. |
| Agents listen on `8000`, MCP servers on `8080` | Drives `targetPort` in Bicep. |

Naming convention for every image in this lab series:

```
$ACR.azurecr.io/zavashop/<service>:<short-git-sha>
```

**Never `:latest`.** Lab 04's deploy gate greps `infra/` and the workflows for
that string and fails the build if it appears.

---

## Step 2 — Pin the deployment SHA

Everything you build and deploy in this lab is stamped with one immutable tag.

```bash
git status --porcelain      # must be empty
export GIT_SHA=$(git rev-parse --short HEAD)
echo "deploying $GIT_SHA"
```

If `git status` is not empty, commit or stash first. A dirty tree means the tag
no longer identifies what is running.

---

## Step 3 — Build the base image with ACR Tasks

`az acr build` uploads the build context to Azure and builds there. No local
Docker daemon, no architecture mismatch on Apple Silicon.

```bash
az acr build -r $ACR \
  --platform linux/amd64 \
  -t zavashop/base:$GIT_SHA \
  -f src/Dockerfile.base .
```

Verify:

```bash
az acr repository show-tags -n $ACR \
  --repository zavashop/base -o tsv
```

---

## Step 4 — Build the four MCP server images

MCP images do not depend on the base image.

```bash
for service in inventory supplier shipping pricing; do
  az acr build -r $ACR \
    --platform linux/amd64 \
    -t zavashop/${service}-mcp:$GIT_SHA \
    -f src/mcp_servers/${service}/Dockerfile .
done
```

---

## Step 5 — Build the five agent images

Agent images consume the ACR-qualified base image.

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

> The orchestrator image is built here even though it deploys in Lab 04. One
> SHA, one build pass, both planes.

### Acceptance

```bash
az acr repository list -n $ACR -o tsv | sort
```

Expect exactly these ten repositories:

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

## Step 6 — Create the Container Apps environment

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

Wait for it:

```bash
az containerapp env show -g $RG -n $CAE \
  --query properties.provisioningState -o tsv   # Succeeded
```

> Prefer no log ingestion cost in a throwaway lab? Replace the three
> `--logs-*` flags with `--logs-destination none`. Everything else is
> unchanged, but `az containerapp logs show` will return nothing.

---

## Step 7 — Read the Bicep module

Open `infra/aca/agent.bicep`. It is deployed **eight times** with different
parameters. Walk through the five things it encodes:

```bash
sed -n '1,40p' infra/aca/agent.bicep
```

| Block | What it does | Why it is written that way |
|---|---|---|
| `identity` | Attaches the Lab 01 UAMI as `UserAssigned` | One identity for ACR pull *and* Key Vault read |
| `configuration.secrets` | `keyVaultUrl: https://<kv>/secrets/GITHUB-TOKEN` with `identity: uamiId` | The token is never a Bicep parameter, never in a deploy log |
| `configuration.registries` | `server` + `identity` (no password) | ACR admin user stays disabled |
| `containers[0].probes` | Liveness `/healthz`, readiness `/readyz` | The delivered `make_app()` already exposes both |
| `template.scale` | `minReplicas` param, `maxReplicas: 10`, HTTP concurrency `30` | KEDA HTTP rule from the Learn doc |

Two parameters you will set per app:

- `targetPort` — `8080` for MCP servers, `8000` for agents.
- `mcpEndpoints` — an object of extra env vars, e.g.
  `{ "ZAVA_INVENTORY_MCP_URL": "https://.../mcp" }`.

Validate the template compiles before deploying anything:

```bash
az bicep build --file infra/aca/agent.bicep --stdout > /dev/null \
  && echo "bicep ok"
```

> **Why `minReplicas: 1` in this lab?** Scale-to-zero is the ACA superpower, but
> a cold start plus a Copilot SDK call can exceed the smoke-test budget in Lab
> 04. The lab pins `minReplicas=1` for reliable demos. Production sets it to `0`
> after cold-start budgets are measured.

---

## Step 8 — Deploy one MCP server by hand

Do the first one manually so you can see exactly what the module produces.

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

Check it:

```bash
export INVENTORY_MCP_FQDN=$(az containerapp show \
  -g $RG -n inventory-mcp \
  --query properties.configuration.ingress.fqdn -o tsv)

curl -fsS "https://$INVENTORY_MCP_FQDN/healthz"
curl -fsS "https://$INVENTORY_MCP_FQDN/readyz"
```

Both must return `{"status":...}`. If the app is stuck, read the logs:

```bash
az containerapp logs show -g $RG -n inventory-mcp --tail 50
```

---

## Step 9 — Preview the rest with `what-if`

Before deploying the remaining seven, preview one to build the habit:

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

Read the `+ Create` block. Confirm no secret value is printed — only the
`keyVaultUrl` reference.

---

## Step 10 — Deploy all eight with `deploy.sh`

`infra/aca/deploy.sh` runs the same `az deployment group create` eight times in
dependency order and writes every resulting URL to `.env.fqdns`.

```bash
sed -n '1,40p' infra/aca/deploy.sh
```

What it does:

1. Sources `.env.lab` and requires `RG`, `ACR`, `CAE`, `KV`, `UAMI_*`.
2. Resolves `GIT_SHA` from `DEPLOY_SHA`, `GIT_SHA`, or `git rev-parse`.
3. Deploys `inventory-mcp`, `supplier-mcp`, `shipping-mcp`, `pricing-mcp`.
4. Reads back each MCP FQDN and builds the four `.../mcp` URLs.
5. Deploys the four specialists, injecting their MCP URL via `mcpEndpoints`.
6. Writes `.env.fqdns` with `ZAVA_*_URL` entries for Lab 04's Helm values.

Syntax-check, then run:

```bash
bash -n infra/aca/deploy.sh && echo "shell syntax ok"

DEPLOY_SHA=$GIT_SHA bash infra/aca/deploy.sh
```

Re-running is safe — each deployment is an idempotent ARM update.

---

## Step 11 — Verify the eight apps

```bash
az containerapp list -g $RG -o table
```

All eight must show `Succeeded`.

```bash
cat .env.fqdns
```

Expect 8 `ZAVA_*_URL` lines plus 4 `ZAVA_*_MCP_URL` lines.

Probe every service:

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

Eight `200`s.

---

## Step 12 — Call a specialist end to end

This proves the whole chain: ACA ingress → agent → Copilot SDK (`gpt-5.5`) →
MCP server over HTTP.

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

You should get a JSON response referencing stock levels for `ZS-1042`.
First call may take 20–40 s while the Copilot SDK warms up.

Repeat for `supplier`, `logistics`, and `pricing`.

---

## Step 13 — Confirm the secret never touched a parameter

```bash
az containerapp show -g $RG -n inventory \
  --query properties.configuration.secrets -o json
```

You should see a `keyVaultUrl` and an `identity`, and **no** `value` field.

```bash
az deployment group show -g $RG -n aca-inventory-mcp \
  --query properties.parameters -o json | grep -i token
```

No output means no token was ever passed as a parameter.

---

## Step 14 — Understand the scaling behaviour

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

Try a production-shaped variant on one app only:

```bash
az containerapp update -g $RG -n pricing --min-replicas 0
az containerapp revision list -g $RG -n pricing -o table
```

Then set it back so Lab 04's evals stay fast:

```bash
az containerapp update -g $RG -n pricing --min-replicas 1
```

---

## ✅ Lab 02 done when…

- [ ] `az acr repository list -n $ACR` shows all ten `zavashop/*` repositories.
- [ ] Every image tag equals the short git SHA; `:latest` appears nowhere.
- [ ] `az containerapp env show` reports `Succeeded`.
- [ ] `az containerapp list -g $RG -o table` shows 8 apps, all `Succeeded`.
- [ ] All 8 `/healthz` probes return `200`.
- [ ] `POST /invoke` on `inventory` returns a `ZS-1042` answer.
- [ ] `.env.fqdns` exists and is git-ignored.
- [ ] No Container App secret exposes a literal `value`.

Next → [Lab 03 — Deploy the AKS Cluster](../lab-03-deploy-aks-cluster/README.md)

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `az acr build` fails with `denied` | Re-run `az login`; confirm `Contributor` on the RG from Lab 01 Step 8. |
| Image build fails on `BASE_IMAGE` | You passed the local tag. It must be `$ACR.azurecr.io/zavashop/base:$GIT_SHA`. |
| Container App stuck `Activating` | `az containerapp logs show -g $RG -n <app> --tail 100`. Usually a bad image tag. |
| `Failed to pull image` | UAMI lacks `AcrPull`, or you passed a tag that was never pushed. |
| Secret resolution error on `GITHUB-TOKEN` | UAMI lacks `Key Vault Secrets User`, or the secret name is not exactly `GITHUB-TOKEN`. |
| `/invoke` returns 500 | The Copilot token is invalid or expired. Re-seed Key Vault, then `az containerapp revision restart`. |
| `/invoke` times out | Increase `ZAVA_COPILOT_TIMEOUT_SECONDS`, or warm the app by hitting `/healthz` first. |
| `deploy.sh` errors `missing UAMI_RESOURCE_ID` | `.env.lab` is stale. Re-run Lab 01 Step 13. |
