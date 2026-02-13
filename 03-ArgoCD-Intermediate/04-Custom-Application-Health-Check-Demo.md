# Hands-on: Custom Application Health Check Demo

In this demo, we will learn how to teach Argo CD to detect "Application-Level" failures that standard Kubernetes health checks miss. We will use the **PHP Random Shapes** application.

---

## 🏗️ Step 1: Create the "Health-Check" Application

1.  Navigate to the Argo CD UI.
2.  Click **"NEW APP"** and fill in:
    *   **Application Name**: `health-check-app`
    *   **Project**: `default`
    *   **Sync Policy**: `Manual`
    *   **Sync Options**: Check **Auto-Create Namespace**
    *   **Repository URL**: `https://github.com/Saikiran121/gitops-argocd-project`
    *   **Path**: `./php-random-shapes-project/k8s`
    *   **Cluster URL**: `https://kubernetes.default.svc`
    *   **Namespace**: `health-check`
3.  Click **CREATE** and then **SYNC**.

---

## 🛑 Step 2: Simulate an "Application Failure"

Our PHP app has a built-in "Degraded" mode. If the triangles are set to **white**, the app displays a warning, but the pods stay "Running" (Healthy).

1.  Modify your `k8s/configmap.yaml`:
    ```yaml
    data:
      TRIANGLE_COLOR: "white"  # Change from red to white
    ```
2.  **Commit and Push** the change to GitHub.
3.  In Argo CD, click **REFRESH** (or wait for the webhook) and then **SYNC**.

### 🧐 The Problem:
*   The App UI now shows a **"⚠️ DEGRADED MODE"** warning.
*   **Argo CD status?** It shows **✅ Healthy**.
*   **Why?** Because the pods are running and the readiness probes pass. Argo CD doesn't know that "white triangles" mean the app is in a bad state.

---

## ⚡ Step 3: Implement the Custom Health Check (Lua)

We will now tell Argo CD how to check the **ConfigMap** to determine the application's true health.

1.  **Edit the `argocd-cm` ConfigMap**:
    ```bash
    kubectl edit cm argocd-cm -n argocd
    ```

2.  **Add the Lua Script**:
    Add the following under the `data` section. This script checks the `moving-shapes-colors` ConfigMap for the `TRIANGLE_COLOR` value.

    ```yaml
    data:
      resource.customizations.health.ConfigMap: |
        hs = {}
        if obj.metadata.name == "moving-shapes-colors" then
          if obj.data.TRIANGLE_COLOR == "white" then
            hs.status = "Degraded"
            hs.message = "Triangles are white! This is a forbidden state."
            return hs
          end
        end
        hs.status = "Healthy"
        return hs
    ```

---

## ✅ Step 4: Verification

1.  Go back to the Argo CD UI.
2.  You will notice that the **ConfigMap** resource (`moving-shapes-colors`) now shows a **Red X (Degraded)** status.
3.  The overall **Application Health** now shows **❌ Degraded**.

### 🎯 Why this is powerful:
You have just successfully integrated "Business Logic" into your infrastructure. You can now use this status to:
*   **Prevent Auto-Promotions** in CI/CD.
*   **Trigger Alerts** before users report issues.
*   **Stop Rollouts** if the configuration is invalid.

---

## 🚀 Step 5: Advanced Rule - Forbidden "Red" Colors

Now, let's suppose our organization has a policy: **No shape should ever be Red.** We will update our Lua script to handle this new business rule.

1.  **Edit the `argocd-cm` ConfigMap**:
    ```bash
    kubectl edit cm argocd-cm -n argocd
    ```

2.  **Update the Lua Script**:
    Replace the previous script with this more generic check that specifically looks for the "red" value.

    ```yaml
    data:
      resource.customizations.health.ConfigMap: |
        hs = {}
        hs.status = "Healthy"
        if obj.data.TRIANGLE_COLOR == "red" then
          hs.status = "Degraded"
          hs.message = "Use a different COLOR for TRIANGLE"
        end
        return hs
    ```

3.  **Trigger the Failure**: 
    Go to your `health-check/configmap.yaml` in Git and set:
    ```yaml
    TRIANGLE_COLOR: "red"
    ```
    Push the change to GitHub.

---

## 🔎 Step 6: Verifying the "Red" Violation

1.  In Argo CD, click **REFRESH**.
2.  The application health will immediately flip to **❌ Degraded**.
3.  Click on the **ConfigMap** (`moving-shapes-colors`) to see the custom health details:
    *   **STATUS**: `Synced`
    *   **HEALTH**: `Degraded`
    *   **HEALTH DETAILS**: `Use a different COLOR for TRIANGLE`

---

## 🛠️ Step 7: The Final Fix (GitOps Resolution)

To bring the application back to a healthy state, we follow the proper GitOps workflow:

1.  **Edit Git**: Change the color in `configmap.yaml` from `red` to a safe color (e.g., `green`).
2.  **Push**: Push the update to your repository.
3.  **Argo CD Sync**:
    *   Click **REFRESH** (it will show `OutOfSync`).
    *   Click **SYNC**.
4.  **Success**: The application status will turn **✅ Healthy**.

> [!IMPORTANT]
> **Why do I still see red triangles in the browser?**
> Even though Argo CD shows "Healthy," the running PHP pod injected the environment variables when it first started. Since Kubernetes doesn't automatically restart pods when a ConfigMap changes, you need to manually restart the deployment to pick up the new colors:
> ```bash
> kubectl rollout restart deployment php-random-shapes -n health-check
> ```

---

## 🏁 Summary
By using **Lua Scripts**, you can extend Argo CD to understand any resource in your cluster. This provides the ultimate "Source of Truth" validation, ensuring that if it's not working for the user or violating business rules, it shouldn't show as green in Argo CD.
