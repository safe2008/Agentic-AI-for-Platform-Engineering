# Setup Runbook

A step-by-step path through every hands-on lab in *Agentic AI for Platform Engineering*, in book order. Each step lists:

- **Check**: how to tell whether the step is already done, so you can rerun this runbook safely
- **Do**: what to run (the lab README stays the source of truth for full commands)
- **Verify**: the expected result
- **Status**: progress on this machine

Terms such as *book repo*, *components repo*, and *Backstage app* are defined in [`CONTEXT.md`](./CONTEXT.md).

## Scope

| Chapter | Labs | In this runbook |
| --- | --- | --- |
| 1–3, 6 | Theory only | No |
| 4 — Organizational knowledge | RAG pipeline, MCP server | Yes |
| 5 — Conversational interfaces | Backstage, GitOps, chat assistant, actions | Yes |
| 7 — Operational agents | Agent runtime, diagnostics, GitOps agent, Backstage chat, Langfuse | Yes |
| 8 — Security labs | OPA, signed identity, tenant isolation | **No.** The ebook describes these labs (p. 206), but `chapter-08/` has not been published in this repository or upstream. |
| 9 — Future directions | Memory graph | Yes |

## Deviations from the book

> [!IMPORTANT]
> **Model provider: Anthropic API, not Bedrock.** Chapters 4 and 7 use `LLM_PROVIDER=anthropic` with `ANTHROPIC_API_KEY`, which the labs document as an alternative.
>
> - Chapter 5's chat plugin has no Anthropic provider, so it uses its `openai` provider pointed at Anthropic's OpenAI-compatible endpoint. If tool calls misbehave in Chapter 5 Labs 3–4, fall back to Bedrock for Chapter 5 only.
> - Every Chapter 7 lab copies a fresh `agent-platform/k8s/deployment.yaml` that sets `LLM_PROVIDER: bedrock`. Change it back to `anthropic` before each commit.

> [!IMPORTANT]
> **Two GitHub tokens instead of one.**
>
> - The **read-only token** (`argocd/github-creds`) is used only by ArgoCD.
> - A separate **read/write token** (`$GITHUB_RW_TOKEN`) is used by Backstage and by the Chapter 7 `gitops-mcp-github` Secret. Chapter 7 Lab 0 copies the token out of `argocd/github-creds`; use `$GITHUB_RW_TOKEN` there instead.
>
> This follows the book's own production note to keep ArgoCD's credential read-only.

> [!NOTE]
> **Secrets are created by you, in your own terminal.** Every command that writes a Kubernetes Secret reads values from your environment, so keys never appear in a chat transcript or shell output.

## One-time environment

Run these in every terminal you use for the labs:

```bash
export BOOK_REPO=/Users/catered/github.com/Agentic-AI-for-Platform-Engineering
export COMPONENTS_REPO=/Users/catered/github.com/backstage-components
export BACKSTAGE_APP=/Users/catered/github.com/my-backstage-app
export GITHUB_USERNAME=safe2008
export KIND_PLATFORM=linux/arm64          # linux/amd64 on Intel/AMD
export ANTHROPIC_API_KEY=...              # from console.anthropic.com
export GITHUB_RW_TOKEN=...                # see step 5.2
kubectl config use-context kind-agentic-platform
```

> [!WARNING]
> This kubeconfig also contains EKS contexts. Confirm `kubectl config current-context` prints `kind-agentic-platform` before running any step.

---

## Step 0 — Shared cluster

Lab README: [`00-cluster-setup/`](./00-cluster-setup/README.md)

- **Check:** `kind get clusters` lists `agentic-platform`, and `kubectl -n argocd get deploy` shows 6 available deployments.
- **Do:** `cd 00-cluster-setup && ./setup-cluster.sh`. ArgoCD is installed with `kubectl apply --server-side`, because the ApplicationSet CRD is too large for client-side apply.
- **Verify:** `./verify.sh` passes.
- **Status:** ✅ Done.

---

## Chapter 4 — RAG and MCP over platform knowledge

Lab READMEs: [`chapter-04/`](./chapter-04/README.md)

### 4.1 Prerequisites and corpus (Labs 0–1)

- **Check:** `python3.11 --version` works, and `chapter-04/1-knowledge-corpus/corpus/` exists.
- **Do:** Nothing to run; read Lab 1 for how the corpus is structured.
- **Status:** ✅ Done.

### 4.2 RAG pipeline (Lab 2)

- **Check:** `kubectl -n rag-demo get pods` shows pgvector running, and ingestion reports 18 chunks from 6 documents.
- **Do:** Follow Lab 2 Steps 1–3, then for Step 4 use Option B:
  ```bash
  export LLM_PROVIDER=anthropic   # ANTHROPIC_API_KEY already exported
  ```
- **Verify:** Asking "why does a pod end up in CrashLoopBackOff and how do I fix it?" returns a generated answer that cites `runbooks/crashloopbackoff.md`.
- **Status:** 🟡 Retrieval works. Generation is pending `ANTHROPIC_API_KEY` (Bedrock previously denied `bedrock:InvokeModel`).

### 4.3 MCP server (Lab 3)

- **Check:** `kubectl -n broken-demo get pods` shows `payment-processor` in CrashLoopBackOff.
- **Do:** Follow Lab 3 Steps 1–3.
- **Verify:** The `list_deployments` tool reports `payment-processor` as unhealthy.
- **Status:** ✅ Done.

---

## Chapter 5 — AI-powered Backstage

Lab READMEs: [`chapter-05/`](./chapter-05/README.md)

### 5.1 Components repo and ArgoCD discovery (Lab 1, Parts 2–3)

- **Check:** `gh repo view safe2008/backstage-components` succeeds, and the `backstage-app-discovery` ApplicationSet reports `ErrorOccurred=False`:
  ```bash
  kubectl -n argocd get applicationset backstage-app-discovery \
    -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
  ```
- **Do:**
  - Create the repo with a README (`gh repo create safe2008/backstage-components --private --add-readme`).
  - Apply `chapter-05/1-backstage-setup/files/argocd-applicationset.yaml`.
  - Create `argocd/github-creds` from the **read-only token** (Part 3, Step 5).
- **Verify:** The ApplicationSet conditions show no errors.
- **Troubleshooting:**
  - `Repository not found`: the repo doesn't exist, or the Secret is missing.
  - `Write access to repository not granted`: the token's repository list doesn't include the repo.
  - After fixing the token, restart `argocd-repo-server` and `argocd-applicationset-controller` to clear the cached failure.
  - `dns: A record lookup error: lookup argocd-repo-server ... i/o timeout` after a cluster or Docker restart: the ArgoCD controllers started before CoreDNS was ready and stay stuck. Restart them, wait for the repo server to be ready, then refresh the ApplicationSet:
    ```bash
    kubectl -n argocd rollout restart deploy/argocd-repo-server \
      deploy/argocd-applicationset-controller statefulset/argocd-application-controller
    kubectl -n argocd rollout status deploy/argocd-repo-server
    kubectl -n argocd annotate applicationset backstage-app-discovery \
      argocd.argoproj.io/application-set-refresh=true --overwrite
    ```
- **Status:** ✅ Done.

### 5.2 Read/write token

- **Check:** `GITHUB_TOKEN=$GITHUB_RW_TOKEN gh api repos/safe2008/backstage-components --jq .permissions.push` prints `true`.
- **Do:** Create a fine-grained token limited to `safe2008/backstage-components`, with the permissions from Lab 1 Part 2, Step 2:
  - Contents and Pull requests: read and write
  - Issues: read and write
  - Actions and Commit statuses: read-only

  Export it as `GITHUB_RW_TOKEN`.
- **Status:** ⬜ To do (you).

### 5.3 Backstage app (Lab 1, Part 1)

- **Check:** `$BACKSTAGE_APP/package.json` exists.
- **Do:**
  ```bash
  cd /Users/catered/github.com
  npx @backstage/create-app@0.7.9 --path my-backstage-app   # pin 0.7.9: later versions break Labs 2 and 4
  ```
  Replace the `resolutions` block in `package.json` with `chapter-05/1-backstage-setup/files/package-resolutions.json`, then run `yarn install`. Put the GitHub integration token in `app-config.local.yaml` as `${GITHUB_RW_TOKEN}`.
- **Verify:** `NODE_OPTIONS=--no-node-snapshot yarn start` serves `http://localhost:3000`.
- **Status:** 🟡 Verify passes: `localhost:3000` returns 200, and the backend on `localhost:7007` is up. Confirmed:
  - `@backstage/create-app` is pinned to 0.7.9.
  - The `resolutions` block matches `package-resolutions.json`.
  - `yarn install` has run.
  - `app-config.local.yaml` reads the token from `${GITHUB_RW_TOKEN}`.

  Still open: confirm `GITHUB_RW_TOKEN` was exported in the terminal that ran `yarn start` (5.2). The backend's environment can't be inspected from outside, so restart it with the token exported if unsure. The `webapp-template` files are in `examples/` but not yet registered in `app-config.yaml` (step 5.4).

### 5.4 Web-app template and the first app (Lab 1, Parts 4–5)

- **Check:** `kubectl -n argocd get application my-first-app` shows Synced and Healthy.
- **Do:** Copy `files/webapp-template/` to `$BACKSTAGE_APP/examples/`, and register it plus the components-repo catalog location in `app-config.yaml`. Then create an app from the template in the Backstage UI, and merge the PR it opens.
- **Verify:** ArgoCD deploys the new app, and its pods run.
- **Status:** 🟡 `my-first-app` already exists: it was rendered from the template by hand (components repo PR #1) and is Synced and Healthy. To exercise the Backstage flow without clashing, create a second app from the template (for example `my-second-app`).

### 5.5 AI chat assistant (Lab 2)

- **Do:** Follow Steps 1–5. In Step 6, configure the `general` agent with the OpenAI-compatible provider instead of `bedrock`:
  ```yaml
  langgraph:
    messagesMaxTokens: 150000
    openai:
      apiKey: ${ANTHROPIC_API_KEY}
      modelName: claude-sonnet-4-5
      baseUrl: https://api.anthropic.com/v1/
  ```
  Skip Step 7 (AWS credentials).
- **Verify:** The Chat Assistant answers a general platform-engineering question.
- **Status:** ⬜ To do.

### 5.6 Catalog actions (Lab 3) and Kubernetes actions (Lab 4)

- **Do:** Follow both labs as written.
- **Verify:**
  - The assistant finds `my-first-app` in the catalog.
  - The assistant lists pods in `my-first-app`.
- **Status:** ⬜ To do.

---

## Chapter 7 — Operational agents

Lab READMEs: [`chapter-07/`](./chapter-07/README.md)

Every Chapter 7 lab ships through the components repo. The flow is **branch → PR → you review and merge → ArgoCD refresh**:

```bash
kubectl -n argocd annotate applicationset backstage-app-discovery \
  argocd.argoproj.io/application-set-refresh=true --overwrite
```

Before every commit, run this to confirm the provider is still Anthropic:

```bash
grep -A1 LLM_PROVIDER "$COMPONENTS_REPO/agent-platform/k8s/deployment.yaml"   # must say anthropic
```

### 7.0 Credentials (Lab 0)

- **Check:** `kubectl -n agent-platform get secret agent-llm-credentials gitops-mcp-github` lists both Secrets.
- **Do (you, in your own terminal):**
  ```bash
  kubectl create namespace agent-platform --dry-run=client -o yaml | kubectl apply -f -
  kubectl -n agent-platform create secret generic agent-llm-credentials \
    --from-literal=ANTHROPIC_API_KEY="$ANTHROPIC_API_KEY" \
    --dry-run=client -o yaml | kubectl apply -f -
  kubectl -n agent-platform create secret generic gitops-mcp-github \
    --from-literal=GITHUB_PERSONAL_ACCESS_TOKEN="$GITHUB_RW_TOKEN" \
    --dry-run=client -o yaml | kubectl apply -f -
  ```
- **Verify:** `"$BOOK_REPO/00-cluster-setup/verify.sh" --github-secret argocd/github-creds` passes.
- **Status:** ⬜ To do (you).

### 7.1 Agent runtime (Lab 1)

- **Check:** `kubectl -n agent-platform get pods` shows `agent-runtime-*` Running 1/1.
- **Do:**
  - Build and load `agent-runtime:0.1.0`.
  - Copy `agent-platform/` into the components repo, then open and merge the PR.
  - Merge the provider PR (components repo PR #3).
- **Verify:** `POST /invoke` answers using the identity from `configmap-identity.yaml`, and `save_to_memory` writes `/state/memory/MEMORY.md`.
- **Status:** 🟡 The image is loaded into kind, and the manifests are merged (PR #2). The pod is in `CreateContainerConfigError` until the 7.0 Secrets exist and PR #3 (Anthropic provider) is merged.

### 7.2 Cluster diagnostics (Lab 2)

- **Do:**
  - Load `quay.io/containers/kubernetes_mcp_server:v0.0.66`, and build and load `agent-runtime:0.7.2`.
  - Copy the manifests and set the provider back to `anthropic`.
  - Open the PR, then merge it.
- **Verify:** The `cluster-mcp` and `agent-runtime:0.7.2` rollouts complete, and asking it to "Inspect my-first-app" returns an evidence-based health report.
- **Status:** ⬜ To do.

### 7.3 GitOps agent and collaboration (Lab 3)

- **Do:**
  - Load `ghcr.io/github/github-mcp-server:v1.12.1`, and build and load `agent-runtime:0.7.4`.
  - Copy the manifests, replace `YOUR_USERNAME`, and set the provider back to `anthropic`.
  - Open the PR, then merge it.
  - Open and merge the deliberate `[lab] break my-first-app image` PR.
- **Verify:** The agent diagnoses the broken image tag and opens an `[agent]` repair PR. After you merge that PR, `my-first-app` is healthy again.
- **Status:** ⬜ To do.

### 7.4 Backstage chat collaboration (Lab 4)

Requires 5.5–5.6.

- **Do:**
  - Copy `strands-proxy-agent.ts` into the Backstage backend and register it.
  - Point `genai.agents.general` at `strands-proxy` (`http://localhost:18080`).
  - Create `agent-backstage-read` (you), and add the static `externalAccess` token.
- **Verify:** In one chat conversation, the assistant finds `my-first-app` in the catalog, diagnoses it, and proposes a PR when the evidence supports one.
- **Status:** ⬜ To do.

### 7.5 Langfuse traces and audit log (Lab 5)

- **Do:**
  - Build and load `agent-runtime:0.7.5`.
  - Create the `langfuse-db` and `langfuse-app` Secrets (you).
  - Copy the `langfuse/` and `agent-platform/` manifests, set the provider back to `anthropic`, open the PR, then merge it.
  - Create Langfuse API keys in the UI, then create `agent-langfuse-keys` (you).
- **Verify:**
  - Langfuse shows `coordinator.invoke`, `diagnostics.invoke` and `gitops.invoke` traces for session `lab5-observability`.
  - The audit-chain check prints `Chain OK`.
- **Status:** ⬜ To do.

---

## Chapter 9 — Living memory graph

Lab README: [`chapter-09/memory-graph/`](./chapter-09/memory-graph/README.md)

- **Check:** `chapter-09/memory-graph/files/mcp/.venv` exists.
- **Do:** Follow Steps 1–3. `requirements.txt` pins `mcp<2`, because mcp 2.x renamed `FastMCP`.
- **Verify:** `search_memory("prompt injection")` returns results.
- **Status:** ✅ Done.
