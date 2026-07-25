# Lab 03 — Deploy the AKS Cluster: Entra ID, Workload Identity & Helm

> ⏱ **Fast track ~35 min · Full lab ~75 min** · Provision a landing-zone-aware AKS cluster, wire Microsoft Entra ID + Azure RBAC, enable Workload Identity and the Key Vault CSI driver, then read and dry-render the delivered Helm chart. The real rollout happens in Lab 04.

🇨🇳 [中文版](./README.zh.md)

### ⏱ Running the whole series in 2 hours?

| | Steps | Budget |
|---|---|---|
| **Core** | 2, 3, 4, 5, 6, 7, 8, 9, 10, 14, 15 + Part D | ~35 min |
| *Optional* | 1 cluster shape · 11 chart anatomy · 12 values · 13 security posture · 16 server-side dry run | ~40 min |

> Start Step 2 first — cluster creation runs ~8 min unattended. Read the optional
> steps while it provisions and you get them for free.

## The story

Eight of ZavaShop's nine services now live on Container Apps. The ninth — the
**orchestrator** — cannot.

It is the store manager of the fleet: long-lived, multi-replica, fronted by a
load balancer, spread across availability zones, and holding a Copilot token
that must never appear in an environment variable in a template. It needs a
Kubernetes-shaped runtime with Azure-native identity.

This lab is about **understanding AKS well enough to run it**, not about
`kubectl apply`. You will:

1. Build a cluster with the landing-zone controls turned on from minute one.
2. Prove that humans authenticate with **Microsoft Entra ID**, authorize with
   **Azure RBAC**, and never touch an admin kubeconfig.
3. Give the orchestrator pod an Azure identity with **Workload Identity** —
   no secret in the cluster.
4. Project `GITHUB-TOKEN` from Key Vault with the **Secrets Store CSI driver**.
5. Read the delivered Helm chart and render it offline until you can predict
   every object it creates.

At the end, the cluster is production-shaped and the chart is verified — but
nothing is deployed yet. That deliberate gap is Lab 04.

---

## Microsoft Learn knowledge for this lab

- [Azure Kubernetes Service (AKS) overview](https://learn.microsoft.com/azure/aks/intro-kubernetes) — what AKS manages for you and what stays yours.
- [AKS core concepts](https://learn.microsoft.com/azure/aks/core-aks-concepts) — control plane, node pools, namespaces, workloads.
- [AKS architecture guidance](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-start-here) — the current production reference architecture.
- [AKS landing zone accelerator](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/app-platform/aks/landing-zone-accelerator) — the design areas this cluster maps to.
- [Microsoft Entra ID integration with AKS](https://learn.microsoft.com/azure/aks/enable-authentication-microsoft-entra-id) — cluster authentication without local accounts.
- [Use Azure RBAC for Kubernetes Authorization](https://learn.microsoft.com/azure/aks/manage-azure-rbac) — Kubernetes authorization through Azure role assignments.
- [AKS Workload Identity](https://learn.microsoft.com/azure/aks/workload-identity-overview) — federating a Kubernetes service account to a UAMI.
- [Secrets Store CSI driver on AKS](https://learn.microsoft.com/azure/aks/csi-secrets-store-driver) — mounting Key Vault secrets into pods.
- [Azure CNI Overlay networking](https://learn.microsoft.com/azure/aks/azure-cni-overlay) — the IP-efficient network plugin used here.
- [Azure Policy for AKS](https://learn.microsoft.com/azure/aks/use-azure-policy) — cluster guardrails and compliance reporting.
- [Microsoft Defender for Containers](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-containers-introduction) — runtime threat detection for the cluster.
- [Container Insights](https://learn.microsoft.com/azure/azure-monitor/containers/container-insights-overview) — node, pod, and container telemetry.
- [AKS Helm quickstart](https://learn.microsoft.com/azure/aks/quickstart-helm) — the chart workflow used by `infra/aks/helm/zavashop`.
- [Pod security context](https://learn.microsoft.com/azure/aks/developer-best-practices-pod-security) — the non-root, read-only-root-filesystem baseline.

---

## Prerequisites

```bash
cd /path/to/AKS-Lab-GitHubCopilot
source .env.lab

echo "$RG / $AKS / $ACR / $KV / $LAW_ID"
echo "$AKS_ADMINS_GROUP_ID"
```

Lab 02 must be complete: the eight Container Apps are live and `.env.fqdns`
exists. The orchestrator image `zavashop/orchestrator:<sha>` is already in ACR.

---

## Part A — Build the cluster

### Step 1 — Decide the cluster shape before typing *(optional)*

Every flag in the next command answers one landing-zone design question.

| Design area | Decision | Flag |
|---|---|---|
| Compute | 2 nodes, `Standard_D4s_v6` — enough for 2 orchestrator replicas plus system pods | `--node-count`, `--node-vm-size` |
| Identity (authN) | Humans sign in with Microsoft Entra ID | `--enable-aad` |
| Identity (authZ) | Kubernetes RBAC delegated to Azure role assignments | `--enable-azure-rbac` |
| Admin access | Only members of the Lab 01 Entra group | `--aad-admin-group-object-ids` |
| Workload identity | Pods federate to the UAMI via an OIDC issuer | `--enable-oidc-issuer`, `--enable-workload-identity` |
| Secrets | Key Vault projected via CSI | `--enable-addons azure-keyvault-secrets-provider` |
| Governance | Policy add-on for guardrails | `--enable-addons azure-policy` |
| Monitoring | Container Insights → Lab 01 workspace | `--enable-addons monitoring`, `--workspace-resource-id` |
| Networking | Azure CNI Overlay to conserve IPs | `--network-plugin azure --network-plugin-mode overlay` |
| Registry | Kubelet pulls from ACR with managed identity | `--attach-acr` |

### Step 2 — Create the cluster

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

This takes 5–10 minutes. While it runs, read Part B below.

If your subscription has no `Standard_D4s_v6` quota, substitute
`Standard_D4s_v5` or `Standard_D2s_v5` (2 nodes minimum).

### Step 3 — Enable the Defender profile

```bash
az aks update \
  -g $RG -n $AKS \
  --enable-defender \
  --defender-config \
      logAnalyticsWorkspaceResourceId=$LAW_ID
```

### Step 4 — Capture cluster facts into `.env.lab`

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

## Part B — Identity: how humans and pods get in

Two completely separate paths. Confusing them is the most common AKS mistake.

```
Humans                                   Pods
──────                                   ────
Entra ID sign-in                         ServiceAccount token
   │                                        │
   ▼                                        ▼
Azure RBAC role on the cluster           Federated credential on the UAMI
   │                                        │
   ▼                                        ▼
kubectl (no local admin account)         DefaultAzureCredential → Key Vault
```

### Step 5 — Grant human access through Azure RBAC

```bash
az role assignment create \
  --assignee $AKS_ADMINS_GROUP_ID \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --scope $AKS_ID
```

Grant the UAMI the minimum CI needs — a kubeconfig, nothing more:

```bash
az role assignment create \
  --assignee-object-id $UAMI_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $AKS_ID
```

> CI never gets `RBAC Cluster Admin`. Lab 04's workflow only needs to run
> `helm upgrade` in one namespace.

### Step 6 — Get a kubeconfig the right way

```bash
az aks get-credentials -g $RG -n $AKS --overwrite-existing
kubectl get nodes -o wide
```

The first `kubectl` command triggers an Entra ID device-code sign-in. That is
the proof that authentication is federated.

```bash
kubectl auth can-i create deployment -n zavashop
kubectl auth can-i '*' '*' --all-namespaces
```

**Never** run `az aks get-credentials --admin`. It bypasses Entra ID entirely
and Lab 04's gate treats it as a failure.

### Step 7 — Create the namespace and service account

```bash
kubectl create namespace zavashop \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl label namespace zavashop \
  project=zavashop lab=03 --overwrite
```

The Helm chart creates `orchestrator-sa` itself (see
`templates/serviceaccount.yaml`), but the federated credential must exist
before any pod starts.

### Step 8 — Federate the service account to the UAMI

```bash
az identity federated-credential create \
  --name fc-orchestrator \
  --identity-name $UAMI \
  --resource-group $RG \
  --issuer "$AKS_OIDC" \
  --subject system:serviceaccount:zavashop:orchestrator-sa \
  --audiences api://AzureADTokenExchange
```

The subject is `system:serviceaccount:<namespace>:<serviceaccount>`. If the
chart's namespace or SA name changes, this must change too.

```bash
az identity federated-credential list \
  --identity-name $UAMI -g $RG \
  --query "[].{name:name, subject:subject}" -o table
```

You should see two rows: `gha-aks-lab-main` (Lab 01) and `fc-orchestrator`.

### Step 9 — Verify the CSI driver is running

```bash
kubectl get pods -n kube-system \
  -l app=secrets-store-csi-driver

kubectl get pods -n kube-system \
  -l app=secrets-store-provider-azure
```

If the add-on was not enabled at create time:

```bash
az aks enable-addons \
  -g $RG -n $AKS \
  --addons azure-keyvault-secrets-provider \
  --enable-secret-rotation
```

### Step 10 — Prove the identity chain works end to end

Test before Helm, so a failure here is unambiguous.

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

A directory listing containing `GITHUB-TOKEN` means Workload Identity, the
federated credential, the Key Vault RBAC grant, and the CSI driver are all
correct.

Clean up the smoke objects — the chart brings its own:

```bash
kubectl -n zavashop delete pod wif-smoke
kubectl -n zavashop delete secretproviderclass wif-smoke
```

> If the pod stays `ContainerCreating`, describe it:
> `kubectl -n zavashop describe pod wif-smoke`. The CSI error message names
> the exact failing step.

---

## Part C — Helm: read the chart before you run it

### Step 11 — Chart anatomy *(optional)*

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

Four Kubernetes objects, one per concern:

| Template | Object | The AKS concept it demonstrates |
|---|---|---|
| `serviceaccount.yaml` | `ServiceAccount` | Workload Identity annotation `azure.workload.identity/client-id` |
| `secretproviderclass.yaml` | `SecretProviderClass` | Key Vault → CSI projection of `GITHUB-TOKEN` |
| `deployment.yaml` | `Deployment` | Replicas, probes, security context, topology spread, CSI volume |
| `service.yaml` | `Service` | `LoadBalancer` publishing port 80 → container 8000 |

### Step 12 — Required values *(optional)*

```bash
cat infra/aks/helm/zavashop/values.yaml
```

Six values have no default and **must** be supplied:

| Value | Source |
|---|---|
| `image.repository` | `$ACR.azurecr.io/zavashop/orchestrator` |
| `image.tag` | the git SHA from Lab 02 |
| `workloadIdentity.clientId` | `$UAMI_CLIENT_ID` |
| `keyVault.name` | `$KV` |
| `tenantId` | `$AZURE_TENANT_ID` |
| `a2a.{inventory,supplier,logistics,pricing}` | `.env.fqdns` from Lab 02 |

The chart uses Helm's `required` function, so a missing value fails the render
rather than deploying a broken pod.

### Step 13 — Read the Deployment's security posture *(optional)*

```bash
sed -n '15,50p' infra/aks/helm/zavashop/templates/deployment.yaml
```

Five hardening decisions to recognize:

1. `azure.workload.identity/use: "true"` pod label — required for token injection.
2. `podSecurityContext`: `runAsNonRoot`, uid/gid/fsGroup `10001` — matches `Dockerfile.base`.
3. `securityContext`: `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, all capabilities dropped.
4. `topologySpreadConstraints` on `topology.kubernetes.io/zone` with `maxSkew: 1`.
5. The container reads the token from the CSI mount at startup, so the value never appears in a `kubectl get pod -o yaml`.

> `readOnlyRootFilesystem: true` is why the chart mounts an `emptyDir` at
> `/home/zava/.cache` — the Copilot SDK needs a writable cache path.

### Step 14 — Lint

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

Expected: `1 chart(s) linted, 0 chart(s) failed`.

### Step 15 — Render with your real values

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

Inspect what would be created:

```bash
grep '^kind:' /tmp/zavashop-rendered.yaml | sort -u
```

```
kind: Deployment
kind: SecretProviderClass
kind: Service
kind: ServiceAccount
```

Assert the three properties Lab 04's gate depends on:

```bash
grep -c 'image: ' /tmp/zavashop-rendered.yaml          # 1
grep 'image: ' /tmp/zavashop-rendered.yaml             # ends with :$GIT_SHA
grep -i 'latest' /tmp/zavashop-rendered.yaml           # no output
```

Confirm no secret value leaked into the manifest:

```bash
grep -i 'GITHUB_TOKEN' /tmp/zavashop-rendered.yaml
```

The only match should be the `cat /mnt/secrets-store/GITHUB-TOKEN` shell
command in the container args — never a literal token.

### Step 16 — Server-side dry run *(optional)*

This validates against the live API server, including admission and the Azure
Policy add-on, without creating anything.

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

Every line must end in `(server dry run)`. Stop here — Lab 04 performs the
actual `helm upgrade --install`.

---

## Part D — Verify the landing-zone controls

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

Check the Azure Policy add-on is actually evaluating:

```bash
kubectl get pods -n kube-system -l app=azure-policy
kubectl get constrainttemplates 2>/dev/null | head
```

Check Container Insights is receiving data (allow ~10 minutes after create):

```bash
kubectl get pods -n kube-system -l dsName=ama-logs-ds
```

The full identity + secret handoff checklist lives in
[infra/aks/wif/README.md](../../infra/aks/wif/README.md).

---

## ✅ Lab 03 done when…

- [ ] `kubectl get nodes` returns 2 `Ready` nodes after an Entra ID sign-in.
- [ ] `aadProfile.enableAzureRBAC` is `true`.
- [ ] The Entra group holds `Azure Kubernetes Service RBAC Cluster Admin`; the UAMI holds only `Cluster User`.
- [ ] `oidcIssuerProfile.enabled` and `securityProfile.workloadIdentity.enabled` are `true`.
- [ ] Azure Policy add-on and Defender security monitoring are `true`.
- [ ] Container Insights points at the Lab 01 workspace.
- [ ] The `fc-orchestrator` federated credential exists with subject `system:serviceaccount:zavashop:orchestrator-sa`.
- [ ] The `wif-smoke` pod listed `GITHUB-TOKEN` from the CSI mount, and was deleted.
- [ ] `helm lint` returns `0 chart(s) failed`.
- [ ] `helm template` renders exactly `Deployment`, `Service`, `ServiceAccount`, `SecretProviderClass`.
- [ ] `kubectl apply --dry-run=server` passes.
- [ ] Nothing is actually deployed yet.

Next → [Lab 04 — Deploy to Production](../lab-04-deploy-production/README.md)

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `az aks create` fails on quota | Try `Standard_D4s_v5`/`Standard_D2s_v5`, or a different region. |
| `az aks create` hangs > 15 min | Check `az aks show --query provisioningState` in another shell; delete and retry if `Failed`. |
| `kubectl` returns `Unauthorized` | Re-run `az aks get-credentials --overwrite-existing`; confirm you are in the Entra admin group. |
| `kubectl` returns `Forbidden` | Azure RBAC is working. Confirm the group role assignment on `$AKS_ID` and wait for propagation. |
| `wif-smoke` stuck `ContainerCreating` | `kubectl -n zavashop describe pod wif-smoke` — the CSI event names the failure. |
| CSI error `failed to get objectType:secret` | UAMI lacks `Key Vault Secrets User`, or the vault name/tenant id is wrong. |
| CSI error `no matching federated identity record` | Subject mismatch. It must be `system:serviceaccount:zavashop:orchestrator-sa`. |
| `helm lint` fails on `required` | You omitted one of the six mandatory values in Step 12. |
| `--dry-run=server` rejected by policy | Azure Policy is enforcing a constraint. Read the message and adjust the cluster policy, not the chart's security context. |
