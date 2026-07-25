# Lab 01 — Deployment Foundation: Take Delivery of the Fleet & Lay the Azure Groundwork

> ⏱ ~60 min · **No code generation in this lab series.** The multi-agent application already exists under `src/`. You clone it, verify it runs, and build the Azure foundation it will be deployed onto.

🇨🇳 [中文版](./README.zh.md)

## The story

You are the platform engineer on call at **ZavaShop**, a global retailer with 500+ stores.

Last sprint, a team of **GitHub Copilot Custom Coding Agents** delivered a complete multi-agent supply-chain application. It is already in this repository:

- 5 Microsoft Agent Framework agents (1 orchestrator + 4 specialists)
- 4 FastMCP tool servers
- Unit, integration, and golden-eval tests
- Dockerfiles, a Helm chart, an ACA Bicep module, and a CD workflow

Nobody wrote a line of it by hand. Now the business wants it **live on Azure by the end of the week**.

Your job across these four labs is not to build the app. It is to answer one question:

> *"I just received a multi-agent project produced by custom Coding Agents. How do I get it onto Azure — fast, securely, and repeatably?"*

| Lab | What you do |
|---|---|
| **01 (this lab)** | Clone the delivered app, prove it runs locally, provision the shared Azure foundation |
| **02** | Containerize the fleet and deploy 8 services to **Azure Container Apps with Bicep** |
| **03** | Learn and provision **AKS** — Entra ID, Azure RBAC, Workload Identity, Helm |
| **04** | The **real end-to-end deployment**: Bicep + Helm + GitHub Actions OIDC, smoke, evals, Day-2 |

---

## What you were handed

This is the application you are deploying. Read it — do not rewrite it.

### Runtime services (9 containers + 1 base image)

| Service | Source | Runtime | Port | Routes |
|---|---|---|---|---|
| `orchestrator` | `src/agents/orchestrator/` | AKS (Lab 03/04) | 8000 | `/healthz` `/readyz` `/invoke` `/plan` |
| `inventory` | `src/agents/inventory/` | ACA (Lab 02) | 8000 | `/healthz` `/readyz` `/invoke` |
| `supplier` | `src/agents/supplier/` | ACA (Lab 02) | 8000 | `/healthz` `/readyz` `/invoke` |
| `logistics` | `src/agents/logistics/` | ACA (Lab 02) | 8000 | `/healthz` `/readyz` `/invoke` |
| `pricing` | `src/agents/pricing/` | ACA (Lab 02) | 8000 | `/healthz` `/readyz` `/invoke` |
| `inventory-mcp` | `src/mcp_servers/inventory/` | ACA (Lab 02) | 8080 | `/healthz` `/readyz` `/mcp` |
| `supplier-mcp` | `src/mcp_servers/supplier/` | ACA (Lab 02) | 8080 | `/healthz` `/readyz` `/mcp` |
| `shipping-mcp` | `src/mcp_servers/shipping/` | ACA (Lab 02) | 8080 | `/healthz` `/readyz` `/mcp` |
| `pricing-mcp` | `src/mcp_servers/pricing/` | ACA (Lab 02) | 8080 | `/healthz` `/readyz` `/mcp` |
| `base` | `src/Dockerfile.base` | shared layer | — | — |

### The business scenario baked into the code

> *"SKU `ZS-1042` will stock out at `store-101` by Friday. What should we do?"*

`POST /plan` on the orchestrator fans out to all four specialists over A2A HTTP, each specialist calls its own MCP server, and the orchestrator returns a structured `Plan` with `stock_view`, `po_view`, `shipping_view`, `price_view`, `summary`, and `next_actions`.

### Shape of the delivered repository

```
src/
├── Dockerfile.base                 # python:3.11-slim, non-root uid 10001
├── shared/                         # Settings, telemetry, Copilot client, FastAPI factory
├── agents/<name>/                  # agent.py · tools.py · prompts.py · server.py · Dockerfile
└── mcp_servers/<name>/             # server.py · models.py · store.py · Dockerfile
tests/                              # unit · integration · tests/evals (golden set)
infra/
├── aca/agent.bicep + deploy.sh     # Lab 02
└── aks/helm/zavashop + aks/wif     # Lab 03 / 04
.github/workflows/deploy.yml        # Lab 04
docker-compose.yml                  # local full-fleet run
```

---

## Microsoft Learn knowledge for this lab

- [Azure Container Registry introduction](https://learn.microsoft.com/azure/container-registry/container-registry-intro) — one registry shared by ACA and AKS.
- [Azure Key Vault overview](https://learn.microsoft.com/azure/key-vault/general/overview) — where the Copilot `GITHUB-TOKEN` lives.
- [Managed identities for Azure resources](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview) — one UAMI for AKS pods, ACA apps, and GitHub Actions.
- [Workload Identity Federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation) — secret-less auth for CI and for pods.
- [Configure GitHub Actions OIDC with Azure](https://learn.microsoft.com/azure/developer/github/connect-from-azure-openid-connect) — the `repo:OWNER/REPO:ref:refs/heads/main` subject format.
- [Microsoft Defender for Containers](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-containers-introduction) — the container security baseline the deploy gates check.
- [Azure Monitor Container Insights](https://learn.microsoft.com/azure/azure-monitor/containers/container-insights-overview) — the workspace AKS will attach to in Lab 03.
- [AKS landing zone accelerator](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/app-platform/aks/landing-zone-accelerator) — the design areas this foundation maps to.

---

## What you provision in this lab

| Resource | Name pattern | Used by |
|---|---|---|
| Resource group | `rg-zavashop-lab` | everything |
| Container registry | `acrzavashop<rand>` | Lab 02 + Lab 03 + Lab 04 |
| Log Analytics workspace | `law-zava-<rand>` | Lab 03 (Container Insights, Defender) |
| User-assigned managed identity | `uami-zavashop` | ACA apps, AKS pods, GitHub Actions |
| Entra ID security group | `zavashop-aks-admins` | Lab 03 human cluster access |
| Key Vault | `kv-zava-<rand>` + `GITHUB-TOKEN` | every workload |
| Defender for Cloud plans | `Containers`, `KeyVaults` | Lab 04 deploy gate |
| `.env.lab` | repo root, git-ignored | Labs 02–04 |

> AKS is **not** created here — it is Lab 03. The Container Apps environment is **not** created here — it is Lab 02.

---

## Step 0 — Tooling check

```bash
az version                  # >= 2.65
kubectl version --client
helm version                # >= 3.14
docker version
python --version            # 3.11+ (CI runs 3.13)
uv --version
git --version
```

Install what is missing:

```bash
# macOS
brew install azure-cli kubectl helm uv git

# Windows
winget install --id Microsoft.AzureCLI
winget install --id Kubernetes.kubectl
winget install --id Helm.Helm
winget install --id astral-sh.uv
```

Add the Azure CLI extensions the later labs need:

```bash
az extension add -n containerapp --upgrade
az provider register -n Microsoft.App --wait
az provider register -n Microsoft.ContainerService --wait
```

---

## Step 1 — Clone the delivered application

```bash
git clone https://github.com/microsoft/AKS-Lab-GitHubCopilot.git
cd AKS-Lab-GitHubCopilot
```

Install the Python environment exactly as CI does:

```bash
uv sync
```

Confirm the delivered fleet is intact:

```bash
ls src/agents          # inventory logistics orchestrator pricing supplier
ls src/mcp_servers     # inventory pricing shipping supplier
ls infra/aca           # agent.bicep deploy.sh
ls infra/aks/helm/zavashop
```

---

## Step 2 — 10-minute code tour

Read these six files in order. You will not modify them, but every deployment
decision in Labs 02–04 traces back to one of them.

| # | File | What to notice |
|---|---|---|
| 1 | `src/shared/settings.py` | Every runtime knob is a `ZAVA_*` env var. This is the entire deployment contract. |
| 2 | `src/shared/server.py` | `make_app()` gives every service `/healthz`, `/readyz`, `/invoke`. Probes come free. |
| 3 | `src/agents/orchestrator/server.py` | `POST /plan` fans out to the four specialists and assembles the `Plan`. |
| 4 | `src/agents/orchestrator/a2a.py` | Specialists are reached by **URL**, never by Python import — that is why they can live on ACA. |
| 5 | `src/mcp_servers/inventory/server.py` | FastMCP on port `8080`, path `/mcp`, plus custom `/healthz` + `/readyz` routes. |
| 6 | `src/Dockerfile.base` | Multi-stage, non-root `zava` uid `10001`, `EXPOSE 8000`. |

The deployment contract you must satisfy in Azure:

| Env var | Consumed by | Value in Azure |
|---|---|---|
| `GITHUB_TOKEN` | all agents | Key Vault `GITHUB-TOKEN` (CSI on AKS, `secretRef` on ACA) |
| `ZAVA_COPILOT_MODEL` | all agents | `gpt-5.5` |
| `ZAVA_COPILOT_TIMEOUT_SECONDS` | all agents | `120` |
| `ZAVA_<NAME>_MCP_URL` | specialists | ACA MCP FQDN + `/mcp` (Lab 02) |
| `ZAVA_<NAME>_A2A_URL` | orchestrator | ACA specialist FQDN + `/invoke` (Lab 04) |
| `AZURE_CLIENT_ID` | all workloads | UAMI client id (Workload Identity) |

---

## Step 3 — Run the quality gate

The delivered code ships green. Prove it before you touch Azure — Lab 04's
deploy workflow refuses to roll if this is red.

```bash
uv run poe check
```

This runs `ruff check` → `ruff format --check` → `pyright` (strict) → `pytest`.

Expected tail:

```
=== N passed in Xs ===
```

If it fails, fix the environment (usually a stale `uv sync`) before continuing.
Do not "fix" the application code — it is the delivered artifact.

---

## Step 4 — Get a GitHub Copilot token

Every agent in this repo uses the **GitHub Copilot SDK** with `model="gpt-5.5"`.
No Azure OpenAI deployment is required. Authentication is a GitHub token read
from `GITHUB_TOKEN`.

1. Go to <https://github.com/settings/personal-access-tokens/new>.
2. Create a **fine-grained PAT** with the **Copilot (read)** permission.
3. Do **not** grant repo write.

```bash
export GITHUB_TOKEN="<your-fine-grained-PAT>"
```

Smoke-test it:

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

> ⚠️ Never commit the token. It goes into Key Vault in Step 10 and is projected
> into workloads by CSI (AKS) and `secretRef` (ACA).

---

## Step 5 — Run the whole fleet locally (recommended)

This is the fastest way to understand what you are about to deploy. Docker
Compose starts all 9 services with the same images Azure will run.

```bash
export GIT_SHA=$(git rev-parse --short HEAD)

docker compose --profile build build zavashop-base
docker compose build
docker compose up -d
```

Watch them go healthy:

```bash
docker compose ps
```

Exercise the same call Lab 04 will make against AKS:

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

Tear it down:

```bash
docker compose down -v
```

---

## Step 6 — Sign in to Azure and set naming variables

```bash
az login --use-device-code
az account set --subscription "<your-subscription-id>"
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

> `AKS` and `CAE` are only names here. The resources are created in Lab 03 and
> Lab 02 respectively — but the names are written into `.env.lab` now so the
> later labs stay copy-paste friendly.

---

## Step 7 — Resource group, container registry, Log Analytics

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

> `--admin-enabled false` is deliberate. Labs 02–04 pull images with the managed
> identity, never with a registry username and password.

```bash
az monitor log-analytics workspace create \
  -g $RG -n $LAW -l $LOCATION \
  --tags project=zavashop lab=01

export LAW_ID=$(az monitor log-analytics workspace show \
  -g $RG -n $LAW --query id -o tsv)
```

---

## Step 8 — The single managed identity

One user-assigned managed identity (UAMI) is the identity for **ACA apps**,
**AKS pods**, and **GitHub Actions**. Creating it once is what makes the rest of
the labs secret-less.

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

Grant the two roles it needs today:

```bash
export ACR_ID=$(az acr show -n $ACR -g $RG --query id -o tsv)
export RG_ID=$(az group show -n $RG --query id -o tsv)

# 1. Data plane: pull images (used by ACA and the AKS kubelet)
az role assignment create \
  --assignee-object-id $UAMI_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role AcrPull \
  --scope $ACR_ID

# 2. Control plane: az acr build, ACA Bicep deploys, AKS updates
az role assignment create \
  --assignee-object-id $UAMI_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role Contributor \
  --scope $RG_ID
```

> 💡 **Both are required.** `AcrPull` is data-plane only. Without `Contributor`
> (or any role granting `Microsoft.ContainerRegistry/registries/read`), CI fails
> with the misleading error *"The resource with name '<acr>' could not be
> found"*. The AKS-specific Cluster User role is added in Lab 03 once the
> cluster exists.

---

## Step 9 — Entra ID group for human cluster operators

Lab 03 turns on Microsoft Entra ID + Azure RBAC for AKS. Create the group that
will hold the humans now.

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

> If your tenant blocks group creation, ask an Entra administrator for a group
> object id and continue with
> `export AKS_ADMINS_GROUP_ID="<group-object-id>"`.

---

## Step 10 — Key Vault and the `GITHUB-TOKEN` secret

```bash
az keyvault create -n $KV -g $RG -l $LOCATION \
  --enable-rbac-authorization \
  --tags project=zavashop lab=01

export KV_ID=$(az keyvault show -n $KV -g $RG --query id -o tsv)
```

Grant the UAMI read access and yourself write access:

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

Seed the Copilot token (RBAC propagation can take a minute — retry if it 403s):

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

The secret name `GITHUB-TOKEN` is hard-coded in `infra/aca/agent.bicep` and in
`infra/aks/helm/zavashop/templates/secretproviderclass.yaml`. Do not rename it.

---

## Step 11 — Defender for Cloud baseline

Lab 04's deploy gate refuses to roll unless these two plans are `Standard`.

```bash
az provider register -n Microsoft.Security --wait

az security pricing create -n Containers --tier Standard
az security pricing create -n KeyVaults --tier Standard
```

Verify:

```bash
az security pricing show -n Containers --query pricingTier -o tsv   # Standard
az security pricing show -n KeyVaults  --query pricingTier -o tsv   # Standard
```

> If a central policy blocks these in your lab subscription, record the existing
> assignment and continue. Lab 04 explains how to document the exception rather
> than bypass the gate.

---

## Step 12 — GitHub Actions federated credential

Lab 04 runs the same rollout from GitHub Actions with **no client secret**.
Federate the UAMI to your fork's `main` branch now.

```bash
export GH_REPO_SLUG="OWNER/REPO"   # e.g. contoso/AKS-Lab-GitHubCopilot

az identity federated-credential create \
  --name gha-aks-lab-main \
  --identity-name $UAMI \
  --resource-group $RG \
  --issuer https://token.actions.githubusercontent.com \
  --subject "repo:${GH_REPO_SLUG}:ref:refs/heads/main" \
  --audiences api://AzureADTokenExchange
```

Verify the subject exactly matches — a typo here surfaces later as
`AADSTS70021: no matching federated identity record found`.

```bash
az identity federated-credential list \
  --identity-name $UAMI -g $RG \
  --query "[].{name:name, subject:subject}" -o table
```

---

## Step 13 — Persist the foundation to `.env.lab`

Every later lab starts with `source .env.lab`.

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

Make sure it can never be committed:

```bash
grep -qxF '**/.env.lab' .gitignore \
  || echo '**/.env.lab' >> .gitignore

grep -qxF '**/.env.fqdns' .gitignore \
  || echo '**/.env.fqdns' >> .gitignore

git ls-files --error-unmatch .env.lab >/dev/null 2>&1 \
  && echo "ERROR: .env.lab is tracked — rotate the PAT and git rm --cached it"
```

> `.env.lab` holds resource names and identity ids — **not** the live
> `GITHUB_TOKEN`. The token lives only in your shell and in Key Vault.

---

## Step 14 — Register GitHub repo secrets and variables

Needed only for Lab 04's CD pipeline, but set them now while the values are in
your shell.

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

These names match `.github/workflows/deploy.yml` exactly.

---

## ✅ Verification

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

Lab 01 is done when:

- [ ] `uv run poe check` is green.
- [ ] The Copilot SDK smoke test printed a reply from `gpt-5.5`.
- [ ] `POST /plan` returned a structured plan from the local Compose fleet.
- [ ] `rg-zavashop-lab` contains ACR, Key Vault, Log Analytics, and the UAMI.
- [ ] The UAMI holds `AcrPull` (ACR scope) and `Contributor` (RG scope).
- [ ] `GITHUB-TOKEN` exists in Key Vault.
- [ ] Defender `Containers` and `KeyVaults` plans return `Standard`.
- [ ] The `gha-aks-lab-main` federated credential exists with the right subject.
- [ ] `.env.lab` exists at the repo root and is git-ignored.

Next → [Lab 02 — Deploy to Container Apps](../lab-02-deploy-container-apps/README.md)

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `uv run poe check` fails on import | Re-run `uv sync`; confirm Python 3.11+. |
| Copilot SDK returns 401 | The PAT lacks the Copilot permission, or your account has no Copilot seat. |
| Copilot SDK returns 403 on `gpt-5.5` | Your Copilot plan may not include `gpt-5.5`. Check <https://github.com/settings/copilot>. |
| `az keyvault secret set` returns 403 | RBAC has not propagated. Wait 60 s and retry. |
| `docker compose up` fails on `BASE_IMAGE` | Run `docker compose --profile build build zavashop-base` first, with `GIT_SHA` exported. |
| `az ad group create` denied | Tenant policy. Ask an Entra admin for the group object id. |
| `az security pricing create` denied | Central Defender policy. Record the existing assignment and continue. |
| `.env.lab` accidentally committed | Rotate the PAT immediately, then `git rm --cached .env.lab`. |
