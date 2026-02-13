# Hands-on: Argo CD Sync Strategies Demo

In this lab, we will explore the practical behavior of different sync policies and options using the `health-check-app` we previously deployed.

---

## 🏗️ 1. Manual Sync Policy (The Default)

When an application is created without automated sync, it enters the "Gated" mode.

### ❓ Scenario A: Delete from Git
**Action**: Delete the `k8s/service.yaml` file from your repository and push the change.
**What happens?**
*   Argo CD detects the change and marks the app as **`OutOfSync`**.
*   **Is the Service deleted from Kubernetes?** **No.** The cluster remains in the last known good state.

### ❓ Scenario B: Delete from CLI
**Action**: Run `kubectl delete svc shapes-service -n health-check`.
**What happens?**
*   The Service is immediately gone from the cluster.
*   Argo CD detects this "Drift" and shows the resource as missing (`OutOfSync`).
*   **Does Argo CD recreate it?** **No.** Not until you manually click "Sync."

---

## 🚫 2. GitOps Best Practice: Why avoid CLI changes?

In a true GitOps methodology, your cluster is a **reflection** of your Git repository. 

*   **Auditability**: If you use `kubectl edit` or `kubectl delete`, there is no record of who changed what or why.
*   **Stability**: Manual changes skip the Continuous Integration (CI) checks and Peer Reviews.
*   **Consistency**: A change made via CLI will be lost if the pod is recreated or the cluster is redeployed.

> [!IMPORTANT]
> **Conclusion**: Manual cluster edits are considered "Infrastructure Debt" and should be avoided in production.

---

## 🤖 3. Enabling Full Automation

Now, let's turn on the "No-Touch" mode.

1.  In the Argo CD UI, open the **Application Details** for `health-check-app`.
2.  Click **APP SPEC** -> **EDIT** (or use the **SYNC POLICY** panel).
3.  Set **Sync Policy** to **Automated**.
4.  Enable these options:
    *   **PRUNE RESOURCES**: Allows Argo CD to delete resources that are removed from Git.
    *   **SELF HEAL**: Allows Argo CD to revert manual cluster changes.
5.  **Click SAVE**.

---

## 🛡️ 4. Demonstrating Self-Healing

1.  **Action**: Attempt to delete your Deployment manually via CLI:
    ```bash
    kubectl delete deployment php-random-shapes -n health-check
    ```
2.  **Observation**: Watch the Argo CD UI instantly. You will see the Deployment status flicker to `OutOfSync` and then immediately back to `Synced`.
3.  **Result**: Kubernetes recreated the pods because Argo CD noticed the deletion and reapplied the desired state from Git. **The cluster self-healed.**

---

## 🧹 5. Demonstrating Auto-Pruning

1.  **Action**: In your Git repository, delete the `k8s/deployment.yaml` file.
2.  **Push**: Commit and push the deletion to GitHub.
3.  **Observation**: Argo CD detects the "missing" manifest in Git. 
4.  **Result**: Because **PRUNE RESOURCES** is enabled, Argo CD immediately issues a delete command for the `php-random-shapes` Deployment in the cluster. **The cluster is perfectly synced with the empty Git state.**

---

## 🏁 Summary
*   **Manual**: Safe for sensitive rollouts; requires human intervention.
*   **Self-Healing**: Acts as an automated security guard against manual drift.
*   **Auto-Pruning**: Keeps the cluster clean by mirroring repository deletions.
