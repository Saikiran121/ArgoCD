# Hands-on: Creating an Application via Argo CD UI

This guide provides a step-by-step walkthrough of creating your first application using the Argo CD Web Interface, with a deep-dive into how Argo CD manages your repository credentials under the hood.

---

## 1️⃣ Step 1: Initial Application Setup

1.  Log into your Argo CD UI.
2.  Click the **"+ NEW APP"** button.
3.  Fill in the **General** section:
    *   **Application Name**: `my-first-app`
    *   **Project**: `default`
    *   **Sync Policy**: `Manual` (Best for initial hands-on testing).

---

## 2️⃣ Step 2: Connecting the Repository

Argo CD needs access to your Git manifests. 

1.  Go to **Settings** (Gear icon) -> **Repositories**.
2.  Click **"CONNECT REPO USING HTTPS"**.
3.  Enter the following:
    *   **Repository URL**: `https://github.com/Saikiran121/ultimate-devops-project-demo`
    *   **Username/Password**: Optional (Only required if the repo is private).
4.  Once connected, you should see the connection status as ✅ **Successful**.

---

## 🔍 3️⃣ Deep Dive: Where are my Repo Secrets?

When you connect a repository via the UI, Argo CD stores those credentials as **Kubernetes Secrets** in the `argocd` namespace.

### List all Argo CD Secrets
```bash
kubectl -n argocd get secrets
```
**Example Output:**
```text
NAME                          TYPE     DATA   AGE
argocd-initial-admin-secret   Opaque   1      85m
argocd-secret                 Opaque   5      86m
repo-420675350                Opaque   4      62s
```
> [!NOTE]
> Secrets prefixed with `repo-` contain your Git repository information.

### Inspect a Repository Secret
To see exactly what Argo CD has stored (in Base64), run:
```bash
kubectl -n argocd get secrets repo-420675350 -o json
```
This JSON output contains the `data` block with fields like `url`, `name`, and `type`.

### Extract and Decode the URL
To verify the URL stored inside the secret without navigating the JSON manually:
```bash
kubectl -n argocd get secrets repo-420675350 -o json | jq .data.url -r | base64 -d
```
**Command Breakdown**:
*   `kubectl ... -o json`: Outputs the full secret in JSON format.
*   `jq .data.url -r`: Uses the `jq` tool to extract the Base64 string from the `url` field (`-r` removes quotes).
*   `base64 -d`: Decodes the string back into human-readable text.

---

## 4️⃣ Step 4: Finalizing Application Creation

Now that the repo is connected:

1.  Return to the **New App** screen.
2.  In the **Source** section:
    *   Select the **Repository URL** you just configured.
    *   Enter the **Path** (e.g., `./` or the folder containing your manifest).
3.  In the **Destination** section:
    *   Select the **Cluster URL** (Default: `https://kubernetes.default.svc`).
    *   Enter the **Namespace** where the app should be deployed.
4.  Click **CREATE**.

---

## 5️⃣ Step 5: Initial Sync & Verification

Once the application is created, you will notice that its status is 🟡 **OutOfSync** and its health is ⚪ **Missing**. This is because we selected **Manual Sync Policy**.

### 1. Observe the Current State
If you click on the application, you will see a list of resources that Argo CD *wants* to create but hasn't yet. For the **OpenTelemetry Demo**, this includes:
*   **Services**: `adservice`, `cartservice`, `checkoutservice`, etc.
*   **Deployments**: `accountingservice`, `frontend`, `paymentservice`, etc.
*   **ServiceAccount**: `opentelemetry-demo`.

### 2. Verify via CLI (Pre-Sync)
If you check your cluster now, the resources do not exist yet:
```bash
# Check namespaces
kubectl get ns

# Check pods in the target namespace (Assuming 'all' was your intent or a specific name)
kubectl get pods -n <your-namespace>
# Output: No resources found in <your-namespace> namespace.
```

### 3. Perform Manual Sync
To move from Git to the cluster:
1.  In the Argo CD UI, click the **"SYNC"** button at the top of the application page.
2.  A panel will appear listing all the resources to be synchronized.
3.  Ensure the desired resources are selected (default is all).
4.  Click **"SYNCHRONIZE"** at the bottom of the panel.

---

## 🛑 6️⃣ Step 6: Troubleshooting Sync Failure

Sometimes, a manual sync will ❌ **Fail**. One of the most common reasons for this in Argo CD is a **Missing Namespace**.

### 1. Identify the Error
If the sync fails, click on the red **"Failed"** status icon. You will likely see an error message similar to:
> `Namespace "abhishek" not found`

### 2. Resolution Options
You have two ways to fix this:
*   **Manual**: Create the namespace yourself using `kubectl create ns abhishek`.
*   **Argo CD Automated**: 
    1.  Click **"SYNC"** again.
    2.  In the Sync panel, look for the **"AUTO-CREATE NAMESPACE"** checkbox.
    3.  Check it and click **"SYNCHRONIZE"**.

### 3. Verification (Post-Success)
After enabling Auto-Create, Argo CD will provision the namespace and all resources. You can verify this using the following commands:

**Check Namespaces:**
```bash
kubectl get ns
```
*Expected Output:*
```text
NAME              STATUS   AGE
abhishek          Active   9s    <-- Created by Argo CD
argocd            Active   102m
default           Active   169m
```

**Check Resources in the New Namespace:**
```bash
kubectl get all -n abhishek
```
*Expected Output:*
```text
NAME                                                            READY   STATUS    RESTARTS   AGE
pod/opentelemetry-demo-accountingservice-85dc886fb6-vt6fh       1/1     Running   0          73s
pod/opentelemetry-demo-adservice-54dc6b6d96-v7gkr               1/1     Running   0          72s
...
service/opentelemetry-demo-adservice               ClusterIP   10.100.116.128   <none>        8080/TCP            75s
...
deployment.apps/opentelemetry-demo-accountingservice       1/1     1            1           76s
...
```

---

## 🔄 7️⃣ Step 7: The GitOps Lifecycle (Handling Changes)

The true power of Argo CD is revealed when you update your application. Let's say you want to upgrade your application from `v1` to `v2`.

### 1. Update the Manifest in GitHub
Locate your Deployment manifest in your Git repository and update the image tag:
```diff
- image: my-app:v1
+ image: my-app:v2
```
Push the change to your GitHub repository.

### 2. Drift Detection in Argo CD
By default, Argo CD polls Git every 3 minutes. If you want to see the change immediately, click the **"REFRESH"** button in the Argo CD UI.
*   **Status**: You will see the application status change from ✅ **Synced** to 🟡 **OutOfSync**.
*   **Reason**: Argo CD has detected that the "Live State" in the cluster (v1) no longer matches the "Desired State" in Git (v2).

### 3. Inspect the Diff
Before syncing, you can see exactly what changed:
1.  Click on the application to enter the details view.
2.  Click the **"APP DIFF"** tab.
3.  Argo CD will show a side-by-side comparison (diff) highlighting the change from `v1` to `v2`.

### 4. Synchronize the Change
To apply the new version:
1.  Click the **"SYNC"** button.
2.  Review the list of resources to be updated.
3.  Click **"SYNCHRONIZE"**.
*   **Observation**: You will see the Kubernetes controller in action. A **new ReplicaSet** will be created for `v2`, and the old one for `v1` will be scaled down once the new pods are healthy.

---

## ↩️ 8️⃣ Step 8: Manual Rollbacks & Pruning

If you discover a bug in your latest deployment (e.g., `v2` is crashing), Argo CD allows you to roll back to a known-good state instantly, even before you fix the code in Git.

### 1. Perform a Manual Rollback
1.  Click the **"HISTORY AND ROLLBACK"** button in the Application UI.
2.  You will see a list of every synchronization event that has ever occurred.
3.  Identify the stable version (e.g., your initial `v1` deployment).
4.  Click the **"Rollback"** button next to that version.
5.  Confirm by clicking **"OK"**.
*   **Result**: Argo CD will immediately re-deploy the manifests associated with that historical version.

### 2. What is "Prune" during Rollback?
While rolling back, you will see a checkbox for **Prune**.

> [!IMPORTANT]
> **Pruning** is the process of deleting resources from the cluster that are NOT defined in the version you are rolling back to.

*   **Scenario Without Prune**: If you added a new "config-map" in `v2` and you rollback to `v1` **without** pruning, that config-map will stay in your cluster, even though `v1` doesn't need it.
*   **Scenario With Prune**: If you check "Prune," Argo CD will delete that extra config-map, ensuring your cluster looks exactly like it did when `v1` was originally deployed.

> [!WARNING]
> Keep in mind that a manual rollback puts your application in an **OutOfSync** state because the cluster is now different from the current `main` branch in Git. To make the rollback permanent, you must eventually revert the change in Git as well.

---

## 🗑️ 9️⃣ Step 9: Deleting an Application

The final step in the lifecycle is removing the application. When you click **DELETE** in the Argo CD UI, what actually happens?

### 1. The Deletion Process
1.  Click the **"DELETE"** button on the application card or details page.
2.  You will be asked to type the application name to confirm.
3.  **Does it delete the resources in Kubernetes?**
    *   **Yes (Default - Cascading Delete)**: Argo CD uses **Finalizers** to ensure that all managed resources (Deployments, Services, Pods, etc.) are deleted *before* the application itself disappears from the UI.
    *   **No (Non-Cascading)**: If you chose to "Orphan" the resources, only the Argo CD metadata is deleted, and the apps keep running in Kubernetes.

### 2. Does it delete the Namespace?
This is a common point of confusion.
*   **Usually NO**: By default, Argo CD protects the namespace. Even if it deletes every Pod and Service inside, the **Namespace itself remains** Active in your cluster. This is to prevent accidental deletion of a namespace that might contain resources from other applications.
*   **The Exception**: If you used the **"Auto-Create Namespace"** feature AND your application is explicitly managing the Namespace resource in its manifests, then it *might* delete it if pruning is enabled. 

> [!TIP]
> To be safe, always assume the **namespace survives**. If you want a perfectly clean cluster after testing, you should manually run:
> `kubectl delete ns abhishek`

---

## 🏁 Summary
By using the UI, you've successfully managed the entire **Argo CD Lifecycle**:
1.  **Connected** a secure repository (and inspected the resulting K8s Secret).
2.  **Created** an application with a manual sync policy.
3.  **Troubleshot** initial failures using **Auto-Create Namespace**.
4.  **Synced** the desired state to the cluster.
5.  **Detected Drift** and synchronized manual changes from Git.
6.  Managed emergency **Rollbacks** with **Pruning**.
7.  **Cleaned up** the environment through **Cascading Deletes**.
