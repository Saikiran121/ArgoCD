# Hands-on: Creating an Application via Argo CD CLI

While the UI is great for visualization, the **Argo CD CLI** is essential for automation, CI/CD pipelines, and power users. This guide covers how to manage the entire application lifecycle from your terminal.

---

## 🛠️ 1. Install the Argo CD CLI

First, download the latest stable binary and move it to your system path.

```bash
# Download the binary
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

# Make it executable
chmod +x argocd

# Move it to a global path
sudo mv argocd /usr/local/bin/

# Verify installation
argocd version --client
```

---

## 🔐 2. Login to the Argo CD Server

You need the DNS address of your Argo CD API server. If you are on AWS, this is typically your **LoadBalancer** URL.

```bash
# Get your LoadBalancer URL if you don't have it
kubectl get svc -n argocd argocd-server

# Login (use --insecure if you haven't configured SSL)
argocd login <LOAD_BALANCER_URL> \
  --username admin \
  --password <YOUR_PASSWORD> \
  --insecure
```

---

## 🚀 3. Create an Application

We will create an application named `solar-system` targeting a specific Git repository and Kubernetes cluster.

```bash
argocd app create solar-system \
    --repo https://github.com/Saikiran121/solar-system \
    --path ./k8s \
    --dest-namespace solar-system \
    --dest-server https://kubernetes.default.svc
```

### Breakdown of the Command:
*   **`--repo`**: The source of truth (Git URL).
*   **`--path`**: The folder inside the repo containing manifests.
*   **`--dest-namespace`**: Where the pods will be deployed.
*   **`--dest-server`**: The API endpoint of the target cluster.

---

## 📊 4. List and Inspect Applications

Once created, you can see the status of your applications.

```bash
argocd app list
```

**Understanding the Output:**
*   **`STATUS: OutOfSync`**: The cluster is empty, but Git has manifests.
*   **`HEALTH: Missing`**: No pods are running yet.
*   **`SYNCPOLICY: Manual`**: Argo CD is waiting for your command to deploy.

---

## 🔄 5. Synchronizing (Deployment)

To move the manifests from Git to the cluster, we run the `sync` command.

### Sync a Single App
```bash
argocd app sync solar-system
```

### Can we sync multiple applications at once?
**Yes!** You have two main ways to batch sync:
1.  **Sync All**: To sync every application managed by this Argo CD instance:
    ```bash
    argocd app sync --all
    ```
2.  **Sync by Labels**: If you have many apps and only want to sync the "production" ones:
    ```bash
    argocd app sync -l app.kubernetes.io/instance=prod-group
    ```

---

## ↩️ 6. Rollbacks

If a deployment fails, you can revert to a previous state instantly.

```bash
# View deployment history
argocd app history solar-system

# Rollback to a specific ID (e.g., ID 0)
argocd app rollback solar-system 0
```

---

## 🗑️ 7. Deleting an Application

To remove the application and its managed resources from the cluster:

```bash
argocd app delete solar-system
```
> [!NOTE]
> This performs a **cascading delete** by default, meaning all Kubernetes resources (Deployments, Services) are removed along with the Argo CD metadata.

---

## 🏁 Summary
Managing Argo CD via the CLI provides a fast and scriptable way to handle your GitOps environment. From installation to emergency rollbacks, the CLI offers the same power as the UI with the speed of the terminal.
