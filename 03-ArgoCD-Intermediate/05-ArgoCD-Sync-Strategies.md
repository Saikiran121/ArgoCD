# Argo CD Sync Strategies: Manual vs. Automated

The **Sync Strategy** is the core decision-maker of your GitOps lifecycle. It determines how and when Argo CD moves the state from Git into your Kubernetes cluster.

---

## 🏗️ 1. Manual Sync Strategy (Gated)

In this model, Argo CD identifies drift (the "Yellow" OutOfSync status) but **wait for a human** to click the "Sync" button.

### 🏢 When to use (Organization Examples):
*   **Banking & Finance**: Where every production change must be approved by a CAB (Change Advisory Board).
*   **Highly Sensitive Infra**: Database migrations or stateful workloads where a person needs to monitor logs during the rollout.
*   **Blue/Green Testing**: When you want to verify the new version in "OutOfSync" diff view before committing to the live cluster.

### 🛠️ Example CLI:
```bash
argocd app sync my-app # Triggers the manual sync
```

---

## 🤖 2. Automated Sync Strategy (No-Touch)

Argo CD periodically polls Git and **instantly applies** changes as soon as they are detected.

### 🚀 When to use (Organization Examples):
*   **SaaS/Tech Startups**: For "Continuous Deployment" to Dev/Staging environments.
*   **Stateless Apps**: Microservices that are safe to update automatically without human intervention.
*   **Disaster Recovery**: Rebuilding an environment from scratch instantly by re-applying all manifests.

---

## ⚙️ 3. Core Sync Options

When you enable automation, you get several powerful modifiers:

| Option | Description | Real-World Use Case |
| :--- | :--- | :--- |
| **Prune** | Deletes resources in K8s that were removed from Git. | Prevents "Zombies" (stale services) from consuming cluster resources. |
| **Self-Heal** | Automatically overwrites manual cluster changes with Git's state. | Stops "Snowflake" clusters created by engineers running `kubectl edit`. |
| **Allow Empty** | Allows the application to have 0 resources. | Useful for "Teardown" scenarios where you want to empty a namespace. |
| **Validate** | Runs `kubectl apply --validate` before syncing. | Catches schema errors (e.g., misspelled YAML keys) before they crash the pod. |

---

## 🧙 4. Advanced Sync Patterns

### A. Replace vs. Apply
*   **Apply (Default)**: Strategic merge patch (`kubectl apply`). Preserves fields managed by others (like HPA).
*   **Replace**: Deletes and recreates the resource (`kubectl replace`). Used for resources that don't support patching (e.g., certain Jobs or specialized CRDs).

### B. Server-Side Apply (SSA)
Enabling SSA allows Argo CD to manage **very large manifests** (like massive ConfigMaps or CRDs) that exceed the standard `last-applied-configuration` annotation limit (64KB).

### C. Prune Propagation Policies
Determines how child resources are deleted:
*   **Foreground**: Deletes the owner and then the dependents (safe).
*   **Background**: Deletes the owner instantly; dependents are cleaned up later.
*   **Orphan**: Deletes the owner but **keeps** the dependents (Rare, use with caution).

---

## 🧹 5. Auto Pruning: The "Garbage Collector" of GitOps

**Auto Pruning** is the process where Argo CD automatically deletes Kubernetes resources that are no longer defined in the source Git repository.

### 🔍 How it Works
1.  **Detection**: Argo CD compares the cluster state with Git. It finds a resource (e.g., a Service) in the cluster that has been deleted from Git.
2.  **Action**: If `Prune=true` is enabled in the sync policy, Argo CD issues a delete command to the Kubernetes API.
3.  **Result**: The cluster perfectly mirrors the current state of Git, leaving no "leftover" or "zombie" resources.

### 🏢 Corporate Scenarios

#### **A. Microservice Decommissioning**
*   **Problem**: When a service is retired, engineers often forget to delete its Ingress, Secrets, or ServiceAccounts manually.
*   **GitOps Solution**: By deleting the service folder from Git, **Auto Pruning** ensures every single related resource is wiped clean from the cluster. This prevents "Resource Leakage" and keeps the cluster performant.

#### **B. Marketing/Campaign Environments**
*   **Problem**: A marketing agency creates short-lived landing pages for 48-hour sales. Manual cleanup is tedious and prone to human error.
*   **GitOps Solution**: The agency uses a folder-per-campaign structure. Once the sale ends, deleting the folder from Git automatically removes the entire production environment.

### ⚠️ Safety Features: "Prune Last"
In complex applications, deleting resources in the wrong order can cause errors. You can use the `PruneLast=true` option to tell Argo CD:
*"Apply all new/updated resources first, and only once everything is stable, go back and delete the old ones."*

### 🛠️ Example
**Git State**: `deployment.yaml` removed from `/apps/my-app`
**Sync Status**: `OutOfSync` (Prune Required)
**Argo CD Action**: `DELETE Deployment/my-app`
**Cluster State**: Pods are terminated; Service stays if it's still in Git.

---

## 🛡️ 6. Self Heal: Guarding the Source of Truth

**Self Heal** is the feature that protects your cluster from "Manual Drift." It ensures that any changes made directly to the cluster (using `kubectl edit` or `kubectl apply`) are automatically reverted to match the state in Git.

### 🔍 How it Works
1.  **Drift Detection**: An engineer manually changes the replica count of a Deployment from `3` to `5` using `kubectl`.
2.  **Comparison**: Argo CD detects that the live state (`5`) no longer matches the desired state in Git (`3`).
3.  **Restoration**: If `SelfHeal=true` is enabled, Argo CD immediately overwrites the cluster change with the Git value.
4.  **Result**: The Deployment is scaled back to `3` within seconds.

### 🏢 Corporate Scenarios

#### **A. Preventing "Snowflake" Clusters**
*   **Problem**: In an emergency, an engineer might "hotfix" a LoadBalancer or ConfigMap manually. Later, nobody remembers this change, making the cluster impossible to reproduce if it crashes.
*   **GitOps Solution**: **Self Heal** forces engineers to make all changes in Git. If they try a hotfix, it gets reverted, prompting them to do it the "correct way." This ensures cluster consistency across Dev, Staging, and Prod.

#### **B. Security & Compliance (Zero Trust)**
*   **Problem**: A malicious actor (or a misconfigured script) gains access to the cluster and tries to open a security hole (e.g., adding a wide-open Ingress rule).
*   **GitOps Solution**: **Self Heal** acts as an automated security guard. As soon as the unauthorized cluster change is detected, Argo CD resets it to the secure state defined in the Git repository.

### ⚠️ Pro-Tip: Manual Gates for Production
While **Self Heal** is amazing for automation, some organizations disable it for **Production Database Migrations**. They want the "OutOfSync" warning so they can manually verify the cluster state before the final sync.

### 🛠️ Example
**Git State**: `replicas: 3`
**Cluster Action**: `kubectl scale deployment/my-app --replicas=10`
**Argo CD Status**: Briefly `OutOfSync`, then `Syncing`.
**Final Result**: Pods scale back to `3` automatically.

---

## 🏁 Summary: The Hybrid Strategy
Most mature organizations use a **Hybrid Pattern**:
1.  **Lower Environments** (Dev/QA): Full **Automated Sync** + **Self-Heal** + **Prune**.
2.  **Production**: **Manual Sync** (or Automated Sync triggered via a Webhook from a Git Approval flow) to maintain safety and audit compliance.
