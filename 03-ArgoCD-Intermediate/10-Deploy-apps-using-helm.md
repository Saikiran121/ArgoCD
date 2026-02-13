# Deploying Applications using Helm in Argo CD

Argo CD has first-class support for **Helm**. Unlike traditional Helm deployments, Argo CD doesn't require "Tiller" (Helm v2) or even the Helm binary on your local machine (though it's useful for testing). Argo CD internally runs `helm template` to generate Kubernetes manifests and then applies them to the cluster.

---

## 🏗️ 1. Two Deployment Models

### **A. Helm Chart in Git (Local Chart)**
The chart source code lives in a subfolder of your Git repository.
*   **Best for**: Custom applications where you own the Helm chart.
*   **Workflow**: You update the chart in Git, and Argo CD detects the change.

### **B. Helm Repository (Public/Private)**
The chart is pulled from a dedicated Helm registry (e.g., Bitnami, Google, or Artifact Hub).
*   **Best for**: Third-party tools (Prometheus, Grafana, Nginx Ingress).
*   **Workflow**: Argo CD monitors the Helm repo for new versions.

---

## 📄 2. Declarative Helm Application

Here is a manifest for an application sourced from a **Helm Repository**.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-ingress
  namespace: argocd
spec:
  project: default
  source:
    chart: ingress-nginx # The chart name
    repoURL: https://kubernetes.github.io/ingress-nginx # The Helm Repo URL
    targetRevision: 4.10.0 # The chart version
    
    # PARAMETER OVERRIDES
    helm:
      valueFiles:
        - values.yaml
      parameters:
        - name: "controller.service.type"
          value: "LoadBalancer"
  
  destination:
    server: "https://kubernetes.default.svc"
    namespace: ingress-nginx
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 🛠️ 3. Managing Values & Overrides

Argo CD provides three ways to override Helm values:

1.  **Values Files**: Point to one or more `values.yaml` files inside your Git repo.
2.  **Parameters**: Use the `parameters:` block (equivalent to `helm install --set`).
3.  **Values Block**: For small overrides, you can paste raw YAML directly into the `values:` field in the Application manifest.

---

## 🚀 4. CLI Commands for Helm

You can create a Helm-based app entirely from the CLI.

### **Example: Deploying from a Helm Repo**
```bash
argocd app create my-helm-app \
    --repo https://charts.bitnami.com/bitnami \
    --helm-chart redis \
    --revision 18.x.x \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace redis-ns \
    --parameter global.storageClass=standard
```

### **Example: Deploying from Git (Local Chart)**
```bash
argocd app create local-helm-app \
    --repo https://github.com/my-org/my-repo.git \
    --path charts/my-app \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace dev
```

---

## 🔍 5. Viewing Helm Details in UI

When you open a Helm-based application in the Argo CD UI:
1.  Click on **APP DETAILS**.
2.  Go to the **PARAMETERS** tab.
3.  You will see a visual list of all the values being overridden. You can even edit them directly in the UI (which will then be out-of-sync with your declarative manifest).

---

## 🎯 6. Best Practices

*   **Version Pinning**: Always pin `targetRevision` to a specific version (e.g., `1.2.3`) instead of `HEAD` or leaving it blank. This prevents unexpected breaking changes when an upstream chart updates.
*   **Value File Order**: If you use multiple value files, the last one in the list takes precedence.
*   **Avoid `--set` for Complex Logic**: For large configurations, always use a `values.yaml` file in Git for better auditability.

---

## 🏁 Summary
Helm is the "Package Manager" for Kubernetes, and Argo CD is the "Automation Engine." Together, they provide a robust, version-controlled way to deploy complex applications with environment-specific overrides.
