# Declarative Setup: Argo CD Application CRD

In GitOps, we don't just manage our app code via Git; we manage our **Argo CD configuration** via Git too. This is often called "GitOps for GitOps." Instead of using the UI or CLI to create apps, we define them using the **Application Custom Resource Definition (CRD)**.

---

## 🏗️ 1. Why Declarative?

*   **Version Control**: Track changes to sync policies, destination clusters, and source paths.
*   **Disaster Recovery**: If your Argo CD server crashes, you can redeploy all your applications instantly by applying your "App manifests."
*   **Scalability**: Manage hundreds of applications without ever touching the UI.
*   **Audit Trail**: Know exactly who changed the deployment path or enabled auto-pruning.

---

## 📄 2. The Application Manifest Breakdown

Below is a production-ready YAML for an Argo CD Application.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-declarative-app
  namespace: argocd # Always created in the argocd namespace
spec:
  project: default # The Argo CD Project name

  # 1. THE SOURCE: Where is the code?
  source:
    repoURL: 'https://github.com/Saikiran121/gitops-argocd-project'
    targetRevision: HEAD # Branch, Tag, or Commit SHA
    path: 'nginx-app'    # Folder containing K8s manifests

  # 2. THE DESTINATION: Where to deploy?
  destination:
    server: 'https://kubernetes.default.svc' # Local cluster or external URL
    namespace: my-app-ns # Target namespace in the cluster

  # 3. THE SYNC POLICY: How to deploy?
  syncPolicy:
    automated:
      prune: true      # Delete resources removed from Git
      selfHeal: true   # Revert manual cluster changes
      allowEmpty: false # Don't allow 0 resources by accident
    syncOptions:
      - CreateNamespace=true # Create target namespace if it doesn't exist
      - PruneLast=true       # Prune at the end of the sync process
      - Validate=true        # Run 'kubectl apply --validate'
```

---

## 🔍 3. Field Depth Analysis

### A. The `source` Block
*   **`repoURL`**: The HTTPS or SSH link to your Git repository.
*   **`targetRevision`**: Defaults to `HEAD`. In production, you might pin this to a specific **Tag** (e.g., `v1.2.0`) to avoid accidental breaking changes.
*   **`path`**: The directory relative to the repository root. If your manifests are at the root, use `./`.

### B. The `destination` Block
*   **`server`**: Most common is `https://kubernetes.default.svc` for the same cluster where Argo CD is installed.
*   **`namespace`**: The logical boundary for your app. **Crucial**: If your manifest already has a namespace defined, this field must match it, or the manifest will take precedence.

### C. The `syncPolicy` Block
*   **`automated`**: Enabling this turns on the "Auto-Sync" loop.
*   **`syncOptions`**: These are modifiers. `CreateNamespace=true` is the most common, as it removes the step of manually creating namespaces via `kubectl`.

---

## 🛠️ 4. How to Apply

To create the application, you simply apply the YAML to the cluster where Argo CD is running:

```bash
kubectl apply -f application.yaml -n argocd
```

Argo CD will immediately pick up this object, create the link to Git, and start the reconciliation loop.

---

## 📄 5. How to List Applications

Once you have applied your manifests, you can verify your applications using either the Argo CD CLI or native Kubernetes commands.

### A. Using Argo CD CLI
This command provides a clean, formatted table of all managed applications.
```bash
argocd app list
```

### B. Using kubectl
Since Applications are Custom Resources (CRDs), you can use standard `kubectl` commands to view them.
```bash
# Short name
kubectl get apps -n argocd

# Full name
kubectl get applications -n argocd
```

---

## 🏁 Summary
The `Application` CRD is the heart of Argo CD. By moving from UI-based creation to Declarative YAML, you ensure that your deployment infrastructure is as resilient and auditable as your application code.
