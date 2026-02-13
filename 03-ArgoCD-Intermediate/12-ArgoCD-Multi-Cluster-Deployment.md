# Argo CD: Multi-Cluster Deployment

One of Argo CD's most powerful features is its ability to manage **multiple remote clusters** from a single central instance. This is known as the **Hub-and-Spoke** model.

---

## 🏗️ 1. The Hub-and-Spoke Architecture

*   **The Hub**: The central Kubernetes cluster where Argo CD and its components (API Server, Controller, Repo Server) are installed.
*   **The Spokes**: Remote Kubernetes clusters (EKS, GKE, On-prem) that only run your application workloads. Argo CD "reaches out" to these clusters to deploy and manage resources.

### **Why use Multi-Cluster?**
1.  **Environment Separation**: Manage `Dev`, `QA`, and `Prod` from one UI.
2.  **Regional Distribution**: Deploy apps to `us-east-1` and `eu-west-1` for low latency and high availability.
3.  **Blast Radius Reduction**: If the Hub cluster goes down, the applications on the Spoke clusters continue to run (though they won't receive updates until the Hub is fixed).

---

## 🛠️ 2. Method 1: Adding a Cluster via CLI

The fastest way to register a remote cluster is using the Argo CD CLI.

### **Step 0: Preparing Kubeconfig (Manual Approach)**
Before Argo CD can find a cluster to add, your local environment must know about it. If you aren't using a cloud helper (like `aws eks update-kubeconfig`), you can add a cluster manually:

1.  **Set the Cluster**: Define the server URL and CA data.
    ```bash
    kubectl config set-cluster spoke-cluster --server=https://<SPOKE_API_SERVER>:6443 --certificate-authority=ca.crt
    ```
2.  **Set Credentials**: Define the user/token for authentication.
    ```bash
    kubectl config set-credentials spoke-admin --token=<BEARER_TOKEN>
    ```
3.  **Set Context**: Link the cluster and user together.
    ```bash
    kubectl config set-context spoke-context --cluster=spoke-cluster --user=spoke-admin
    ```

### **Step 1: Get Contexts**
Verify that your local `~/.kube/config` has the new context.
```bash
kubectl config get-contexts
```

### **Step 2: Add the Cluster to Argo CD**
Register the cluster in Argo CD using the context name.
```bash
argocd cluster add spoke-context
```

### **Step 3: Manage & Verify Clusters**
Once added, you can manage the clusters from the CLI:

*   **List Clusters**: See all clusters registered in Argo CD.
    ```bash
    argocd cluster list
    ```
*   **Get/Describe Cluster**: View detailed information about a specific cluster (server version, status, namespaces).
    ```bash
    argocd cluster get spoke-context
    ```

---

## 📄 3. Method 2: Declarative Cluster Setup

In a production GitOps environment, you should define your clusters as **Secrets**.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: spoke-cluster-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster # CRITICAL LABEL
type: Opaque
stringData:
  name: production-cluster-eu
  server: https://1.23.45.67:6443 # The API server URL of the remote cluster
  config: |
    {
      "bearerToken": "...",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "..."
      }
    }
```

---

## 🚀 4. Targeting the Spoke Cluster

Once a cluster is registered, you can target it in your `Application` manifest.

```yaml
spec:
  destination:
    # Option A: Use the Server URL
    server: https://1.23.45.67:6443 
    
    # Option B: Use the Cluster Name (Easier to read)
    # name: production-cluster-eu
    
    namespace: my-app-ns
```

---

## 🏢 5. Corporate Scenarios

### **Scenario A: The "Promotion" Pipeline**
*   **Workflow**: A developer pushes code to Git. Argo CD automatically syncs to the **Dev Cluster**. After validation, the team updates the manifest to point the destination to the **QA Cluster**, and finally the **Prod Cluster**.
*   **Benefit**: Identical manifest structure across all environments.

### **Scenario B: Disaster Recovery (DR)**
*   **Workflow**: You have an app running in AWS `us-east-1` (Primary). You also have a registered cluster in `us-west-2` (Standby).
*   **Action**: In the event of a regional outage, you simply update the `destination.name` in your Application manifest to the standby cluster. Argo CD instantly recreates the entire stack in the new region.

---

## 🏁 Summary
Multi-cluster management transforms Argo CD into a **Control Plane** for your entire global infrastructure. By centralizing deployment logic, you reduce operational complexity and ensure consistency across every Kubernetes cluster in your organization.
