# Deployment Guide — PR Merge Order & Secrets Setup

This guide tells you exactly what to do with the three open pull requests and when to run your secrets bootstrap, so you end up with a working CI/CD pipeline and a cleanly deployed Azure Container App.

---

## 1. What each open PR contains

| PR | Branch | File(s) changed | Summary |
|----|--------|-----------------|---------|
| [#1](https://github.com/niklasedren/TechWorkshop-L300-AI-Apps-and-agents/pull/1) | `copilot/add-github-actions-workflow` | `.github/workflows/deploy.yml` *(new)* | Full CI/CD workflow: checks out code, logs in to Azure, builds the Docker image from `./src`, pushes to ACR tagged with the commit SHA, updates the Container App image, and injects the four AI endpoint env vars via Container App secrets (`secretref:`). |
| [#2](https://github.com/niklasedren/TechWorkshop-L300-AI-Apps-and-agents/pull/2) | `copilot/add-setup-secrets-documentation` | `.gitignore` | Adds `SETUP_SECRETS.md` to `.gitignore` so the local bootstrap file is never accidentally committed (earlier version). |
| [#3](https://github.com/niklasedren/TechWorkshop-L300-AI-Apps-and-agents/pull/3) | `copilot/update-setup-secrets-guide` | `.gitignore` | Same `.gitignore` change as PR #2, but paired with the **updated** `SETUP_SECRETS.md` that has corrected endpoint values (`gpt_endpoint` host-only, `phi_4_endpoint` with `/models` suffix) and a full 9-step guide including success criteria and troubleshooting. |

---

## 2. Overlap between PR #2 and PR #3

**Both PRs make the same change to `.gitignore`** — they add the line `SETUP_SECRETS.md`. The only difference is the comment above the entry:

- PR #2: `# Local-only secrets bootstrap guide (never commit this file)`
- PR #3: `# Local-only setup guide (contains resource names — never commit)`

**⚠️ Merging both would cause a merge conflict in `.gitignore`.**

**Verdict: merge only PR #3, close PR #2.**

PR #3 is strictly better:
- It corresponds to the **more complete `SETUP_SECRETS.md`** with corrected endpoint paths.
- It was created after PR #2 and explicitly supersedes it.
- Closing PR #2 avoids any conflict risk.

---

## 3. Step-by-step guide — do these in order

### Step 1 — Close PR #2 (avoid future conflict)

On GitHub, open [PR #2](https://github.com/niklasedren/TechWorkshop-L300-AI-Apps-and-agents/pull/2) and click **Close pull request** (do NOT merge it). Leave a comment such as: *"Superseded by PR #3 which contains the updated SETUP_SECRETS.md."*

---

### Step 2 — Merge PR #3 (`.gitignore` update)

On GitHub, open [PR #3](https://github.com/niklasedren/TechWorkshop-L300-AI-Apps-and-agents/pull/3), mark it **Ready for review** (it is currently a draft), and **merge** it.

This adds `SETUP_SECRETS.md` to `.gitignore` so you can safely create that file locally without risking an accidental commit.

> **No secrets, no workflow, no deployment risk** — this is purely a one-line `.gitignore` change.

---

### Step 3 — Create `SETUP_SECRETS.md` locally and run the bootstrap

After PR #3 merges, pull the latest `main` branch locally:

```bash
git checkout main && git pull origin main
```

Create the file `SETUP_SECRETS.md` at the repo root (it will be gitignored). You can use the content from the original chat session or recreate it with the commands below — paste the following into **Azure Cloud Shell** (https://shell.azure.com) or your local Bash terminal:

```bash
# ── Step 3a: Authenticate ────────────────────────────────────────────────────
az account set --subscription "MCAPS-Hybrid-REQ-146331-2026-niklasedren"
gh auth login   # choose: GitHub.com → HTTPS → web browser (one-time only)

# ── Step 3b: Set variables ───────────────────────────────────────────────────
SUB_ID=$(az account show --query id -o tsv)
RG=rg-techworkshop-l300-ai-agents
ACR=bcgeequgk5bsucosureg
CONTAINER_APP=app-bcgeequgk5bsu
IMAGE_NAME=techworkshop-app
REPO=niklasedren/TechWorkshop-L300-AI-Apps-and-agents

# ── Step 3c: AZURE_CREDENTIALS (service principal) ──────────────────────────
az ad sp create-for-rbac \
  --name "gh-actions-techworkshop" \
  --role contributor \
  --scopes /subscriptions/$SUB_ID/resourceGroups/$RG \
  --sdk-auth > azure-creds.json

gh secret set AZURE_CREDENTIALS --repo $REPO < azure-creds.json
rm azure-creds.json   # ← delete immediately; never commit this file

# ── Step 3d: ACR credentials ─────────────────────────────────────────────────
az acr update -n $ACR --admin-enabled true

ACR_LOGIN_SERVER=$(az acr show   -n $ACR --query loginServer             -o tsv)
ACR_USERNAME=$(az acr credential show -n $ACR --query username           -o tsv)
ACR_PASSWORD=$(az acr credential show -n $ACR --query "passwords[0].value" -o tsv)

echo -n "$ACR_LOGIN_SERVER" | gh secret set ACR_LOGIN_SERVER --repo $REPO
echo -n "$ACR_USERNAME"     | gh secret set ACR_USERNAME     --repo $REPO
echo -n "$ACR_PASSWORD"     | gh secret set ACR_PASSWORD     --repo $REPO

# ── Step 3e: Resource group / app / image ────────────────────────────────────
echo -n "$RG"            | gh secret set AZURE_RESOURCE_GROUP --repo $REPO
echo -n "$CONTAINER_APP" | gh secret set CONTAINER_APP_NAME   --repo $REPO
echo -n "$IMAGE_NAME"    | gh secret set IMAGE_NAME           --repo $REPO

# ── Step 3f: Four AI Foundry endpoint secrets ────────────────────────────────
echo -n "https://aif-bcgeequgk5bsu.services.ai.azure.com/api/projects/proj-bcgeequgk5bsu" \
  | gh secret set FOUNDRY_ENDPOINT   --repo $REPO

echo -n "https://aif-bcgeequgk5bsu.services.ai.azure.com" \
  | gh secret set gpt_endpoint       --repo $REPO

echo -n "https://aif-bcgeequgk5bsu.services.ai.azure.com" \
  | gh secret set embedding_endpoint --repo $REPO

echo -n "https://aif-bcgeequgk5bsu.services.ai.azure.com/models" \
  | gh secret set phi_4_endpoint     --repo $REPO

# ── Step 3g: Allow Container App to pull images from ACR ─────────────────────
az containerapp registry set \
  -n $CONTAINER_APP \
  -g $RG \
  --server $ACR_LOGIN_SERVER \
  --username $ACR_USERNAME \
  --password $ACR_PASSWORD

# ── Step 3h: Verify — you should see all 11 secrets ──────────────────────────
gh secret list --repo $REPO
```

**Expected output of `gh secret list`** (order may vary):

```
ACR_LOGIN_SERVER
ACR_PASSWORD
ACR_USERNAME
AZURE_CREDENTIALS
AZURE_RESOURCE_GROUP
CONTAINER_APP_NAME
FOUNDRY_ENDPOINT
IMAGE_NAME
embedding_endpoint
gpt_endpoint
phi_4_endpoint
```

---

### Step 4 — Merge PR #1 (the CI/CD workflow)

Once **all 11 secrets are confirmed**, merge [PR #1](https://github.com/niklasedren/TechWorkshop-L300-AI-Apps-and-agents/pull/1):

1. Open PR #1 on GitHub.
2. Mark it **Ready for review** (it is currently a draft).
3. Click **Merge pull request**.

> **Why last?** PR #1 adds the workflow file. Merging it to `main` is itself a push to `main`, which immediately triggers the workflow. If secrets are not yet in place when this push happens, the first workflow run will fail.

---

### Step 5 — Monitor the first deployment

After merging PR #1, go to the **Actions** tab in the repository:

```
https://github.com/niklasedren/TechWorkshop-L300-AI-Apps-and-agents/actions
```

You will see a run called **"Build and deploy to Azure Container Apps"** triggered by the merge commit. Click into it and watch the steps:

| Step | What to expect |
|------|---------------|
| Checkout repository | ✅ Fast, no credentials needed |
| Azure login | ✅ Uses `AZURE_CREDENTIALS` service principal |
| Log in to ACR | ✅ Uses `ACR_LOGIN_SERVER` / `ACR_USERNAME` / `ACR_PASSWORD` |
| Build and push Docker image | ✅ Takes 2–5 min; tag = commit SHA |
| Update Container App secrets and deploy | ✅ Sets 4 endpoint secrets then updates the app revision |

If any step turns red, click it to expand the logs. Common fixes:

| Error | Likely cause | Fix |
|-------|-------------|-----|
| `AZURE_CREDENTIALS: secret not found` | Secret name typo or not set | Re-run `gh secret set AZURE_CREDENTIALS` |
| `unauthorized: authentication required` (ACR) | ACR admin not enabled or wrong creds | Re-run Step 3d |
| `ContainerAppNotFound` | Wrong `CONTAINER_APP_NAME` or `AZURE_RESOURCE_GROUP` | Verify names with `az containerapp list -g $RG -o table` |
| `REGISTRY_ERROR` | Container App can't pull image | Re-run Step 3g (`az containerapp registry set`) |

---

### Step 6 — Verify the Container App is running the new revision

```bash
az containerapp revision list \
  -n app-bcgeequgk5bsu \
  -g rg-techworkshop-l300-ai-agents \
  --query "[].{revision:name, active:properties.active, image:properties.template.containers[0].image}" \
  -o table
```

The most recent revision should:
- Show `True` in the **active** column.
- Have an image tag matching the merge commit SHA (e.g., `bcgeequgk5bsucosureg.azurecr.io/techworkshop-app:17f83158...`).

---

## 4. Final expected state

After completing all six steps above:

- **`.gitignore`** — includes `SETUP_SECRETS.md` (from PR #3)
- **`.github/workflows/deploy.yml`** — present and active (from PR #1)
- **GitHub Actions secrets** — all 11 configured
- **GitHub Actions** — green check on the first workflow run
- **Azure Container App** — running the latest revision with the new image and the four AI Foundry env vars injected via `secretref:`
- **PR #2** — closed (not merged)
- **Local `SETUP_SECRETS.md`** — present on your machine, gitignored, never pushed

---

## 5. Caveats & conflict notes

| Scenario | Risk | Mitigation |
|----------|------|-----------|
| Merging both PR #2 and PR #3 | **Merge conflict** in `.gitignore` (duplicate `SETUP_SECRETS.md` entries) | Close PR #2 before merging PR #3 |
| Merging PR #1 before secrets are set | First workflow run fails immediately | Set all 11 secrets first (Step 3), then merge PR #1 (Step 4) |
| `SETUP_SECRETS.md` accidentally committed | Resource names visible in git history | Already prevented by the `.gitignore` entry from PR #3; double-check with `git status` |
| `azure-creds.json` left on disk | Service principal JSON exposed | The bootstrap script deletes it with `rm azure-creds.json` immediately after use |
| Service principal scope too broad | Excessive Azure permissions | SP is scoped to the resource group only (`/subscriptions/$SUB_ID/resourceGroups/$RG`) |
