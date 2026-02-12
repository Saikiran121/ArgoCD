# GitOps Principles: A Quick Summary

GitOps is built on four fundamental principles that ensure your infrastructure is automated, reliable, and auditable.

### 1. Declarative State
The entire system is described in a declarative format (e.g., YAML/Helm/Kustomize). You describe the **"Desired State"** (what you want) rather than the steps to get there.

### 2. Versioned & Immutable
The desired state is stored in Git, which serves as a versioned, immutable source of truth. Every change is tracked, and rollbacks are as simple as a `git revert`.

### 3. Automated Delivery
Once the desired state is defined in Git, it is automatically pulled into the environment. No manual intervention is required to trigger a deployment.

### 4. Continuous Reconciliation
Software agents (like Argo CD) continuously compare the **Actual State** (running in the cluster) with the **Desired State** (defined in Git). If they differ, the agent automatically "heals" the system to match Git.

---

## 🏗️ Deep Dive: GitOps Architecture

GitOps architecture is designed to maintain a feedback loop between your version control system and your runtime environment.

### 1. Core Components

*   **Git Repository**: The "Single Source of Truth." It contains all Kubernetes manifests, Helm charts, or Kustomize base/overlays.
*   **CI Pipeline**: Responsible for building code, running tests, and pushing Docker images to a registry. In GitOps, the CI pipeline **updates the Git repository** with the new image tag rather than deploying directly to the cluster.
*   **GitOps Agent (Operator)**: A controller (e.g., Argo CD or Flux) that lives *inside* the cluster. It has two main jobs:
    1.  **Syncing**: Pulling changes from Git.
    2.  **Reconciling**: Ensuring the cluster matches Git.
*   **Actual State**: The live resources running in your environment (Kubernetes namespaces, deployments, services).

### 2. The Reconciliation Loop (The Heart of GitOps)

The GitOps architecture operates on a continuous loop:
1.  **Observe**: The agent watches both the Git Repo (Desired State) and the Cluster (Actual State).
2.  **Diff**: It compares the two. If you manually change a service in the cluster, the agent detects this "Drift."
3.  **Act**: The agent automatically overwrites the manual change to restore the state defined in Git.

### 3. Delivery Models: Push vs. Pull

| Feature | Push Model (Traditional CI) | Pull Model (GitOps Standard) |
| :--- | :--- | :--- |
| **Agent Location** | External (Jenkins, GitHub Actions) | Internal (Inside the Cluster) |
| **Security** | CI needs "Admin" tokens for the cluster. | Cluster only needs "Read" access to Git. |
| **Drift Detection** | No. If someone changes the cluster, CI doesn't know. | Yes. Continuous reconciliation detects drift. |
| **Connectivity** | CI must be able to reach the cluster. | Cluster must be able to reach Git. |

> [!IMPORTANT]
> The **Pull Model** is preferred for GitOps because it adheres to "Zero Trust" security. The cluster does not expose any management ports to the outside world; it only pulls safe configuration from a trusted Git source.
