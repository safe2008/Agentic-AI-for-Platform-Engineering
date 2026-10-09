# Context

Shared vocabulary for setting up and running the labs in this repository. [`SETUP.md`](./SETUP.md) and the lab READMEs use these terms.

## Glossary

**Book repo**
This repository: the companion code for *Agentic AI for Platform Engineering*. Labs refer to its local clone as `$BOOK_REPO`. It contains lab sources and manifests that you copy elsewhere; nothing deploys from it directly.

**Components repo**
The GitHub repository (`backstage-components`) that is the single GitOps source of truth for workloads in Chapters 5 and 7. Labs refer to its local clone as `$COMPONENTS_REPO`. Each top-level folder is one deployable unit with `k8s/` manifests and an `argocd/application.yaml` marker.

**Backstage app**
The developer portal you scaffold in Chapter 5 (`my-backstage-app`), referred to as `$BACKSTAGE_APP`. It lives outside both repositories, runs on your machine, and opens PRs against the components repo.

**Lab app**
The example workload (`my-first-app`, namespace `my-first-app`) that is created from the Backstage web-app template and that the Chapter 7 agents diagnose and repair.

**Agent runtime**
The Strands-based agent service (`agent-runtime`, namespace `agent-platform`) from Chapter 7. It hosts the Platform Coordinator and its specialist agents, and is deployed from the `agent-platform/` folder of the components repo.

**Shared cluster**
The single local kind cluster (`agentic-platform`, kube context `kind-agentic-platform`) that every chapter runs on.

**Discovery ApplicationSet**
The ArgoCD ApplicationSet (`backstage-app-discovery`) that turns each components-repo folder with an `argocd/application.yaml` into an ArgoCD Application.

**Read-only token**
The GitHub token in `argocd/github-creds` that ArgoCD uses to read the components repo.

**Read/write token**
A separate GitHub token (`$GITHUB_RW_TOKEN`) that Backstage and the GitOps Change Agent use to push branches and open PRs against the components repo.
