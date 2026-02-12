# Hands-on: Git Webhook Demo (Instant Reconciliation)

By default, Argo CD polls your Git repository every **3 minutes**. In this demo, we will bypass this delay by configuring a **GitHub Webhook**, enabling Argo CD to react to changes **instantly**.

---

## 🏗️ Step 1: Connect the Demo Repository

First, we need to connect the repository that will trigger the webhooks.

1.  Navigate to **Settings** -> **Repositories**.
2.  Click **"CONNECT REPO VIA HTTPS"**.
3.  Fill in the details:
    *   **Type**: `git`
    *   **Name**: `gitops-argocd-project`
    *   **Project**: `default`
    *   **Repository URL**: `https://github.com/Saikiran121/gitops-argocd-project`
4.  Click **CONNECT**.

---

## 🚀 Step 2: Create the Application

Create an application that targets this repository so we can observe the reconciliation behavior.

1.  Click **"NEW APP"**.
2.  **Application Name**: `git-webhook-app`.
3.  **Project**: `default`.
4.  **Sync Policy**: `Manual`.
5.  **Sync Options**: Check **Auto-Create Namespace** (Important!).
6.  **Repository URL**: Select `https://github.com/Saikiran121/gitops-argocd-project`.
7.  **Path**: `./nginx-app`.
8.  **Cluster URL**: `https://kubernetes.default.svc`.
9.  **Namespace**: `webhook`.
10. Click **CREATE**.
11. **Troubleshooting**: If you get the error *"existing application spec is different"*, it means a resource named `git-webhook-app` already exists in the Kubernetes cluster (even if you don't see it in the UI).
    *   **Solution 1**: Change the Application Name to something unique (e.g., `git-webhook-demo-sai`).
    *   **Solution 2**: Run `kubectl delete application git-webhook-app -n argocd` to clean it up before clicking Create again.

---

## ⏳ Step 3: Observe the "3-Minute Delay"

1.  Make a small change in your GitHub repository (e.g., update a replica count or a label).
2.  Commit and push the change.
3.  **Wait**: You will notice that Argo CD does **not** catch the change immediately. It will wait for its internal 180-second timer to expire before the status turns `OutOfSync`.

---

## ⚡ Step 4: Configure the GitHub Webhook

Now, let's tell GitHub to "ping" Argo CD the moment a push happens.

1.  In your GitHub repository, go to **Settings** -> **Webhooks**.
2.  Click **"Add webhook"**.
3.  **Payload URL**: `https://<YOUR_ARGOCD_SERVER_URL>/api/webhook`
    *   *Example*: `https://aadcf84d605e24fc881df3934388ca41-1594213995.us-east-1.elb.amazonaws.com/api/webhook`
4.  **Content type**: `application/json`.
5.  **Which events?**: Just the `push` event.
6.  Click **Add webhook**.

> [!IMPORTANT]
> Ensure your Argo CD server is reachable from the internet (e.g., via a LoadBalancer) so GitHub can send the POST request.

### ⚠️ Troubleshooting: Webhook failing with HTTPS?
If GitHub shows a red "danger" icon next to your webhook or the "Delivery" fails with a TLS/SSL error, it's because Argo CD uses a self-signed certificate by default.

**To fix this by switching to HTTP (Insecure Mode):**

*   **Option A: Using kubectl patch (Fastest)**:
    ```bash
    kubectl patch deployment argocd-server -n argocd --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--insecure"}]'
    ```

*   **Option B: Using kubectl edit**:
    1.  Edit the deployment: `kubectl edit deployment argocd-server -n argocd`
    2.  Add the `--insecure` flag to the `args` section:
        ```yaml
        args:
        - --insecure
        - --listen-port
        - "8080"
        ```

3.  **Update GitHub Payload URL**: Ensure your URL starts with `http://` instead of `https://`.

---

## ✅ Step 5: Verification (Instant Sync)

1.  Go back to your code and make another change (e.g., change the image tag).
2.  Push the change to GitHub.
3.  **Immediate Reaction**: Watch the Argo CD UI. The application status should move to **`OutOfSync`** instantly without you having to click "Refresh." This proves the webhook triggered the reconciliation loop.

---

## 🕒 Step 6: Alternative - Customizing Polling Interval (The "Timer")

If webhooks are not possible (e.g., your Argo CD is behind a private firewall and GitHub cannot reach it), you can manually reduce the 3-minute check by modifying the `argocd-cm` ConfigMap.

### 1. Edit the ConfigMap
```bash
kubectl edit cm argocd-cm -n argocd
```

### 2. Add the Timeout Setting
Add `timeout.reconciliation` under the `data` section.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Change from 3m (180s) to 60s
  timeout.reconciliation: 60s
```

### 3. Verification
You can verify that the `argocd-repo-server` has picked up the new timer:
```bash
kubectl describe pod argocd-repo-server -n argocd | grep -i RECONCILIATION
```

> [!WARNING]
> Setting this value too low (e.g., `< 10s`) can cause high CPU usage on the Argo CD server and put heavy load on your Git provider (GitHub/Bitbucket).

---

## 🏁 Summary
Congratulations! You've successfully optimized the **Reconciliation Loop**. Instead of a "Pull-based polling" (slow), you've implemented an "Event-driven update" (instant). This is the standard practice in production environments to ensure that CI/CD pipelines move as fast as the developers.
