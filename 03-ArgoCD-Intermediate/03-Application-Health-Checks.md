# Argo CD: Application Health Checks Deep-Dive

In Kubernetes, a pod might be "Running" but the application inside could be crashed or misconfigured. Argo CD goes beyond basic pod status by implementing **Health Checks** to determine if an application is truly functional and ready to serve traffic.

---

## 🚦 1. The 6 Health Statuses

Argo CD assigns one of these six statuses to every resource and application:

| Status | Icon | Description |
| :--- | :---: | :--- |
| **Healthy** | ✅ | The resource is 100% functional and matches its desired state. |
| **Progressing** | ⏳ | The resource is moving towards a healthy state (e.g., pods are starting, replicas are scaling). |
| **Degraded** | ❌ | Something is wrong. The resource failed to reach health (e.g., `CrashLoopBackOff`, failed hook). |
| **Suspended** | ⏸️ | The resource is paused intentionally (e.g., a CronJob not scheduled to run, or a paused Rollout). |
| **Missing** | ❓ | The resource is defined in Git but does not exist in the cluster. |
| **Unknown** | ❕ | Argo CD cannot determine the health (e.g., connectivity issues with the cluster). |

---

## 🧩 2. Health Aggregation Logic

Argo CD uses a "bottom-up" approach to determine the overall Health of an Application:

1.  **Individual Resources**: Each Pod, Service, and ConfigMap has its own health.
2.  **Parent Resources**: A **Deployment** is only "Healthy" if all its underlying **ReplicaSets** are healthy, which in turn depends on the **Pods**.
3.  **Application Level**: The overall **Application Health** is the "worst-case" status of its child resources. 
    *   *If 99 resources are Healthy but 1 is Degraded, the entire Application is **Degraded**.*

---

## 🛠️ 3. Built-in vs. Custom Health Checks

### Built-in Checks
Argo CD comes with native support for standard Kubernetes resources:
*   **Deployments/StatefulSets**: Checks if `availableReplicas` == `updatedReplicas`.
*   **Services**: Checks if the LoadBalancer IP is assigned (for type: LoadBalancer).
*   **Ingress**: Checks if the status has at least one IP/hostname.

### Custom Health Checks (Lua)
For **Custom Resource Definitions (CRDs)** like Argo Rollouts or Cert-Manager, Argo CD doesn't know what "healthy" means. You can define custom logic using **Lua scripts** in the `argocd-cm` ConfigMap.

**Example: Custom Health for a "Database" CRD**
```yaml
data:
  resource.customizations.health.mycompany.com_Database: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.phase == "Running" then
        hs.status = "Healthy"
        hs.message = "Database is ready"
        return hs
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for DB startup"
    return hs
```

---

## 🔍 4. Deep Dive: Kubernetes Service Health

While Pod health is about "Readiness," **Service Health** is about "Reachability." Argo CD evaluates Services differently based on their type:

### A. ClusterIP and NodePort
*   **Status**: `Healthy` ✅
*   **Logic**: These are marked healthy immediately upon creation because they are internal and don't require external provisioning.
*   **Note**: A Service can be "Healthy" even if its selector matches 0 pods. This is why **Aggregation** is important—the Application will still be `Degraded` if the pods are dead.

### B. LoadBalancer (Cloud Services)
*   **Progressing** ⏳: After sync, while waiting for the Cloud Provider (AWS/Azure/GCP) to assign an External IP or Hostname.
*   **Healthy** ✅: Once the IP/Hostname appears in `.status.loadBalancer.ingress`.
*   **Degraded** ❌: If the IP isn't assigned within the timeout period or if the cloud provider returns an error.

### 🏢 Corporate Example: The "Quota Exhaustion" Outage
A company deploys a microservice to a new AWS region during a scale-up. The pods are healthy, but they have hit their **Elastic IP (EIP) quota** in AWS. 
*   Argo CD detects that the Service has no IP.
*   The Service status stays **`Progressing`**.
*   The CI/CD pipeline stops because the overall Application is not **`Healthy`**.
*   **Result**: This prevents the system from assuming the deployment succeeded when the entry point was actually broken.

---

## 🌐 5. Deep Dive: Kubernetes Ingress Health

An **Ingress** is the "External Door" to your application. If it's not healthy, your Service and Pods (even if they are green) are completely hidden from the outside world.

### How Argo CD Evaluates Ingress:
Argo CD monitors the `.status.loadBalancer.ingress` field of the Ingress resource, which is typically populated by your Ingress Controller (Nginx, ALB, Traefik).

*   **Progressing** ⏳: The Ingress resource has been created, but the Ingress Controller hasn't yet assigned it an IP or Hostname.
*   **Healthy** ✅: As soon as at least one IP or Hostname is visible in the status.
*   **Degraded** ❌: If the controller fails to provision the load balancer or if there are configuration errors (e.g., non-existent secret for TLS).

### 🏢 Corporate Example: The "SSL Provisioning" Delay
**The Situation**:
A company uses **Cert-Manager** to automatically provision Let's Encrypt SSL certificates for every new Ingress.

**The Workflow**:
1.  Argo CD syncs a new Ingress.
2.  Cert-Manager sees the Ingress and starts the challenge process to get a TLS certificate.
3.  The **Ingress Controller** waits for the certificate to be ready before it fully activates the Load Balancer.

**With Argo CD Health Checks**:
*   While Cert-Manager is working, the Ingress status is **`Progressing`**.
*   The overall Argo CD application shows **⏳ Progressing**.
*   The DevOps engineer realizes they don't need to manually check `kubectl get ingress` repeatedly. They just watch the Argo CD UI.
*   Once the certificate is issued and the Load Balancer IP is assigned, the status flips to **`Healthy`**.
*   **Result**: This ensures that traffic is only expected to be functional once the entire chain (Ingress -> Cert -> LB) is complete.

---

## 💾 6. Deep Dive: Kubernetes PVC Health

For stateful applications (Databases, Message Queues), the **PersistentVolumeClaim (PVC)** is the "Oxygen." If the storage isn't bound, the application simply won't start.

### How Argo CD Evaluates PVCs:
Argo CD monitors the `.status.phase` of the PVC resource.

*   **Progressing** ⏳: The PVC status is `Pending`. This means it has been requested but no volume has been bound to it yet.
*   **Healthy** ✅: The PVC status is `Bound`. This confirmed that storage has been provisioned and is ready for use.
*   **Degraded** ❌: If the binding fails or if the resource cannot be provisioned (e.g., non-existent StorageClass).

### 🏢 Corporate Example: The "Storage Exhaustion" Disaster
**The Situation**:
A DevOps team is deploying a new **Elasticsearch cluster** using a StatefulSet. Each node requires a 200GB SSD volume.

**The Workflow**:
1.  Argo CD syncs the StatefulSet and the PVC templates.
2.  The PVCs are created in Kubernetes.
3.  The Cloud Provider (e.g., AWS EBS) attempts to provision the volumes.
4.  However, the AWS region has hit the **EBS Volume Quota** for that account.

**With Argo CD Health Checks**:
*   The PVCs remain in `Pending` state.
*   Argo CD marks the PVCs as **`Progressing`** (and eventually **`Degraded`** if the sync timeout is hit).
*   The **StatefulSet** health will also stay as **`Progressing`** because its pods cannot start without the volumes.
*   The overall Application color in Argo CD stays blue (Progressing) or red (Degraded).
*   **Result**: The team can immediately identify that the issue is **Storage Provisioning** rather than an application bug, preventing them from wasting hours debugging the code.

---

## 🏢 7. Corporate Example: The "Ghost Deployment" Protection

**The Scenario**:
A financial institution triggers a deployment for a new payment gateway. The pods start (Kubernetes sees them as "Running"), but they fail to connect to the internal HSM (Hardware Security Module) and are throwing 500 errors.

**Without Argo CD Health Checks**:
The CI/CD pipeline sees "Pods Running" and considers the job finished. Traffic is routed to the broken gateway, causing massive transaction failures.

**With Argo CD Health Checks**:
1.  Argo CD monitors the **Readiness Probes** and custom health scripts.
2.  The application stays in the **`Progressing`** state because the health check hasn't turned green.
3.  Because the health is not **`Healthy`**, the **Sync Strategy** (if configured with `Prune=true`) might prevent the deletion of old, working pods, or the **Argo Rollout** will automatically pause/abort seeing the **`Degraded`** status.
4.  **Result**: Traffic is never fully shifted to the broken pods, protecting the customer experience.

---

## 🚀 5. Best Practices for Production

*   **Always use Readiness Probes**: Argo CD relies heavily on Kubernetes' native probes for its built-in checks.
*   **Monitor 'Progressing' Timeouts**: Configure `progressDeadlineSeconds` in your Deployments. If a resource stays "Progressing" for too long, Argo CD will mark it as **`Degraded`**.
*   **Integrate with Notifications**: Set up alerts for when an app moves from `Healthy` -> `Degraded` to catch issues before users do.

---

## 🏁 Summary
Application Health is the difference between "deployed code" and "functional code." By aggregating thousands of resource statuses into a single, actionable health indicator, Argo CD provides a reliable safety net for complex, multi-service environments.
