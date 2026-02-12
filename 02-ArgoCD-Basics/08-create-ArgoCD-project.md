# Argo CD: Managing Projects & Security Boundaries

In a multi-tenant environment, you don't want every team to have full access to every cluster or repository. Argo CD uses **AppProjects** to create logical security boundaries.

---

## 1. Viewing Existing Projects

By default, Argo CD comes with a `default` project that has no restrictions.

```bash
argocd project list
```

### Explaining the Default Project Output:
*   **DESTINATIONS (`*,*`)**: Applications in this project can be deployed to **any** cluster and **any** namespace.
*   **SOURCES (`*`)**: Can pull code from **any** Git repository.
*   **CLUSTER-RESOURCE-WHITELIST (`*/*`)**: Can create **any** cluster-level resource (Nodes, ClusterRoles, etc.).
*   **NAMESPACE-RESOURCE-BLACKLIST (`<none>`)**: No namespaced resources (Pods, Services) are blocked.

---

## 2. Creating a Custom Project (Hands-on)

To restrict a specific team (e.g., `special-project`), we create a new project with guardrails.

**UI Steps:**
1.  Navigate to **Settings** -> **Projects**.
2.  Click **"+ NEW PROJECT"**.
3.  Name: `special-project` | Description: `Project with security restrictions`.

---

## 3. Defining Security Policies (In-Depth)

### 📍 Source Repositories
This restricts which Git repositories the applications in this project can use.
*   **Config**: Add `https://github.com/Saikiran121/solar-system`.
*   **Result**: If someone tries to create an app using a different repo, Argo CD will block it.

### 🎯 Destinations
This restricts where the code can be deployed.
*   **Server**: `https://kubernetes.default.svc` (Local cluster).
*   **Namespace**: `solar-system`.
*   **Result**: Apps in this project are "trapped" in the `solar-system` namespace. They cannot accidentally overwrite resources in `production` or `kube-system`.

---

## 🛡️ 4. Resource Allow/Deny Lists (The "Firewall")

This is the most powerful part of AppProjects. It controls **what type** of objects can be created.

| List Type | Scope | Example Use Case |
| :--- | :--- | :--- |
| **Cluster Resource Allow List** | Cluster-wide | Allow persistent volumes but block ClusterRoles. |
| **Cluster Resource Deny List** | Cluster-wide | Explicitly block `Node` or `Namespace` modifications. |
| **Namespace Resource Allow List** | Inside Namespace | Allow `Deployment` and `Service` but block `Ingress`. |
| **Namespace Resource Deny List** | Inside Namespace | Block `ResourceQuota` so devs can't change their own limits. |

---

## 5. CLI Inspection & YAML Structure

You can view the resulting security policy in YAML format:

```bash
argocd project get special-project -o yaml
```

### Key YAML Fields:
```yaml
spec:
  # Blocks this team from creating Cluster-level RBAC
  clusterResourceBlacklist:
  - group: '""'
    kind: ClusterRole
  
  # Only allows these specific targets
  destinations:
  - name: in-cluster
    namespace: solar-system
    server: https://kubernetes.default.svc
    
  # Only allows this specific source of truth
  sourceRepos:
  - https://github.com/Saikiran121/solar-system
```

---

## ⚠️ 6. Policy Enforcement: Testing the "Error"

What happens if we try to bypass these rules?

**Scenario**: Try to create an application in `special-project` using a **different** Git repo than the one we whitelisted.

**CLI Command:**
```bash
argocd app create rogue-app \
    --project special-project \
    --repo https://github.com/some-other-repo/malicious-code.git \
    --path ./ \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace solar-system
```

**Resulting Error:**
> `application repo 'https://github.com/some-other-repo/malicious-code.git' is not permitted in project 'special-project'`

### 7. Resource-level Violations (Sync Failures)

Even if your repository and destination are permitted, Argo CD will block the creation of specific **kinds** of resources if they aren't on your allow-list.

**Scenario**: You try to deploy a Helm chart that includes a `ClusterRole` and a `Namespace` creation, but your project is configured only for standard namespaced resources.

**The Resulting Sync Failure:**

When you click **SYNC**, the operation will fail with a `SyncFailed` status. If you look at the individual resource messages:

| RESOURCE KIND | NAME | STATUS | MESSAGE |
| :--- | :--- | :--- | :--- |
| `ClusterRole` | `pod-metadata-reader` | ❌ **SyncFailed** | `resource rbac.authorization.k8s.io:ClusterRole is not permitted in project special-project` |
| `Namespace` | `special-pod` | ❌ **SyncFailed** | `resource :Namespace is not permitted in project special-project` |

> [!CAUTION]
> This is a critical security feature. It prevents a "rogue" developer or a compromised Git repository from escalating privileges (e.g., by creating a `ClusterRoleBinding` to give themselves admin rights) or interfering with other namespaces.

### 8. How to Solve "Resource Not Permitted" Errors

There are two ways to solve this, depending on your goal:

#### Option A: Whitelist the Resource (Administrator Approach)
If you decide that the team *should* be allowed to create these resources, you must update the AppProject.

1.  Go to **Settings** -> **Projects** -> `special-project`.
2.  Scroll to **Cluster Resource Allow List** (for ClusterRole/Namespace).
3.  Click **EDIT** and add the specific resource KIND (e.g., `Namespace` or `ClusterRole`).
4.  Alternatively, via CLI:
    ```bash
    argocd project allow-cluster-resource special-project "" Namespace
    argocd project allow-cluster-resource special-project rbac.authorization.k8s.io ClusterRole
    ```

#### Option B: Clean Up the Manifest (Developer Approach)
If the project security policy is correct and you *shouldn't* be creating these resources:

1.  Remove the `Namespace` and `ClusterRole` manifests from your Git repository.
2.  Push to GitHub and click **REFRESH** in Argo CD.
3.  ARGO CD will now only see permitted resources (like Deployments and Services) and the sync will succeed.

#### Option C: Selective Sync (Quick Workaround)
If your repository contains multiple resources and only one is blocked (e.g., a `ClusterRole`), you can still deploy the rest of the application without changing your code or project settings.

1.  Click the **"SYNC"** button in the Argo CD UI.
2.  In the list of resources, **uncheck** the forbidden resource (e.g., `rbac.authorization.k8s.io/ClusterRole/pod-metadata-reader`).
3.  Click **"SYNCHRONIZE"**.
4.  Argo CD will deploy all the checked (permitted) resources and ignore the unchecked one, allowing your `Deployment` and `Service` to go live without errors.

---

## 🏁 Summary
**AppProjects** are the cornerstone of Argo CD security. By defining allowed sources, destinations, and resource types, you can safely scale Argo CD across hundreds of teams without worrying about cross-tenant interference or security breaches. You've now seen how Argo CD enforces these rules at two levels:
1.  **Creation Level**: Blocking the creation of the Application itself.
2.  **Sync Level**: Blocking the deployment of specific forbidden resources within a permitted application.
