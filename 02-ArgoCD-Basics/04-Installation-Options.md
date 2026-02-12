# Argo CD: In-Depth Installation Options

Choosing the right installation model for Argo CD is critical for balancing operational simplicity, security, and scalability. Argo CD offers several flavors, each suited for specific environment types.

---

## 🏗️ 1. Core (Headless) Installation
The **Core** installation is a lightweight, "headless" version of Argo CD. 

*   **What it is**: It omits the API Server and the Web UI. It installs only the Application Controller and the Repository Server.
*   **How to manage**: Interaction is strictly through Kubernetes CRDs (using `kubectl`) or the Argo CD CLI in "Core" mode.
*   **Key Characteristics**:
    *   **No SSO/RBAC**: Security is managed entirely via standard Kubernetes RBAC.
    *   **No Multi-cluster**: Designed to manage the local cluster where it is installed.
*   **When to use**:
    *   **Edge Clusters**: Low-resource environments where a UI overhead is unwanted.
    *   **GitOps-Only Workflows**: Teams that use purely automated tools and never need a manual dashboard.
    *   **Internal Tools**: When Argo CD is used as an internal library for other higher-level platforms.

---

## 🏢 2. Multi-Tenant Installation
This is the standard, full-featured installation used by the vast majority of organizations.

*   **What it is**: It includes all core components plus the **API Server**, **Web UI**, and **Dex** for SSO.
*   **Key Characteristics**:
    *   **Centralized Management**: One Argo CD instance can manage hundreds of remote clusters.
    *   **Enterprise Security**: Integrates with OIDC (Okta/AD/GitHub) and provides fine-grained RBAC via AppProjects.
    *   **Developer Experience**: Provides a rich dashboard for visualizing resources and logs.
*   **When to use**:
    *   **Platform Engineering**: When a central team provides GitOps-as-a-Service to multiple developer teams.
    *   **Standard Production**: Any environment where human visibility and auditability are required.

---

## ⚡ 3. Non-HA vs. High Availability (HA)

Within the Multi-Tenant model, you must choose between stability and resource usage.

### Multi-Tenant: Non-HA (Default)
*   **Architecture**: Single replicas for the API Server, Repo Server, and Redis.
*   **Risk**: If the Repo Server pod crashes, manifest generation stops. If Redis dies, the cache is lost, and the system slows down while rebuilding.
*   **When to use**: Development, Staging, or small-scale clusters (under 10 applications).

### Multi-Tenant: HA (Production Grade)
*   **Architecture**:
    *   **API Server**: Multiple replicas behind a load balancer.
    *   **Repo Server**: 2+ replicas to handle high Git-to-YAML generation load.
    *   **Redis Sentinel**: A 3-node Redis cluster for resilient caching.
    *   **Application Controller**: Can be sharded across multiple replicas for high cluster counts.
*   **When to use**: **Production**. Always use HA for any cluster that runs business-critical workloads or manages more than 50-100 applications.

---

## 🎯 Decision Matrix: Which one to use?

| Scenario | Recommended Installation | Why? |
| :--- | :--- | :--- |
| **Learning / Sandbox** | Multi-tenant Non-HA | Easiest to explore the UI and features with minimal resources. |
| **Edge / IoT Device** | Core (Headless) | Lowest memory footprint; no need for a UI on every device. |
| **Enterprise Dev Team** | Multi-tenant Non-HA | Provides the dashboard and SSO needed for teams without the cost of HA. |
| **Enterprise Production** | **Multi-tenant HA** | Zero-downtime deployments, resilient manifest caching, and SSO mapping. |
| **Automated CD Bot** | Core (Headless) | If no human ever looks at it, why pay for the UI components? |

---

## 🔒 Post-Installation Best Practices

Regardless of the model, always implement:
1.  **Network Policies**: Restrict the API server to only known CIDR blocks.
2.  **Resource Limits**: Set CPU/Memory limits, especially for the **Repo Server**, which can spike during large manifest generations.
3.  **Horizontal Pod Autoscaling (HPA)**: For the Repo Server in HA mode to handle massive PR merges.

---

## 🚀 Step-by-Step Installation Guide (HA Multi-Tenant)

If you have decided on the **High Availability (HA)** model for your production environment, follow these steps to get started:

### 1. Create the Namespace
Always isolate your Argo CD installation into its own namespace.
```bash
kubectl create namespace argocd
```

### 2. Apply the HA Manifest
Deploy the production-grade manifests directly from the official repository.
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.2.6/manifests/ha/install.yaml
```

### 3. Expose the API Server
To access the UI from outside the cluster, change the service type to `LoadBalancer`.
```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'
```
> [!NOTE]
> Once the LoadBalancer is provisioned, you can get the External IP by running `kubectl get svc argocd-server -n argocd`.

### 4. Retrieve the Initial Admin Password
Argo CD generates a random password for the `admin` user during installation. Use the following command to decrypt it:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```
*   **Username**: `admin`
*   **Password**: (Result of the command above)

---

## 🏁 Summary
**Core** is for machines; **Multi-tenant** is for teams. **HA** is for production; **Non-HA** is for testing. By following the steps above, you now have a production-ready, highly available GitOps engine at your fingertips.
