# Hands-on: Argo CD Multi-Cluster Deployment Demo

In this lab, we will walk through the process of connecting a remote EKS cluster to your central Argo CD Hub and deploying an application across regional boundaries.

---

## 🏗️ 1. The Scenario
We have two Kubernetes clusters:
1.  **`eksdemo1` (Hub)**: Located in `us-east-1`. Argo CD is already installed here.
2.  **`eksdemo2` (Spoke)**: Located in `ap-south-1`. This is our target for remote deployments.

---

## 🔍 2. Initial Verification

First, let's see which clusters Argo CD currently knows about. On your **Hub Cluster (`eksdemo1`)**, run:

```bash
argocd cluster list
```

**Expected Output**:
```text
SERVER                          NAME        VERSION  STATUS      MESSAGE  PROJECT
https://kubernetes.default.svc  in-cluster  1.34     Successful           
```
> [!NOTE]
> `in-cluster` refers to the `eksdemo1` cluster where Argo CD itself is running.

---

## 🔌 3. Connecting the Remote Cluster

### **A. Check your Kubeconfig**
Argo CD uses your local `kubectl` contexts to find the remote cluster.
```bash
kubectl config get-contexts
```

**Output**:
```text
CURRENT   NAME                                     CLUSTER                         AUTHINFO
*         saikiran@eksdemo1.us-east-1.eksctl.io    eksdemo1.us-east-1.eksctl.io    saikiran@eksdemo1...
          saikiran@eksdemo2.ap-south-1.eksctl.io   eksdemo2.ap-south-1.eksctl.io   saikiran@eksdemo2...
```

### **B. Add the Cluster**
Run the following command to register `eksdemo2` into Argo CD:
```bash
argocd cluster add saikiran@eksdemo2.ap-south-1.eksctl.io
```

### **C. Verify the Addition**
```bash
argocd cluster list
```
You should now see **both** clusters in the list!

---

## 🔐 4. Under the Hood: Where are the credentials?

Argo CD doesn't store a "kubeconfig" file. Instead, it stores the remote cluster's API URL and Bearer Token as a **Kubernetes Secret**.

### **Find the Secret**:
```bash
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster
```

### **Inspect the Data**:
To see the encrypted/encoded connection details:
```bash
# Replace <SECRET_NAME> with the name from the previous command
kubectl get secret <SECRET_NAME> -n argocd -o json
```

---

## 🚀 5. Deploying an App to the Remote Cluster

1.  Open the **Argo CD UI**.
2.  Click **+ NEW APP**.
3.  Fill in the usual details (Name, Project, Repository).
4.  **The Critical Step**: In the **DESTINATION** section:
    *   Click the **Cluster URL** dropdown.
    *   Select your new cluster: `https://<eksdemo2-api-endpoint>`.
5.  Click **CREATE** and **SYNC**.

---

## ✅ 6. Final Verification

Now, switch your CLI context to the remote cluster to see your app in action:

```bash
# Switch to Spoke Cluster
kubectl config use-context saikiran@eksdemo2.ap-south-1.eksctl.io

# Verify Pods
kubectl get pods -n <your-app-namespace>
```

---

## 🏁 Summary
You have successfully turned your single-cluster setup into a **Multi-Region Control Plane**. You can now deploy applications to any cluster in the world just by changing a single dropdown in the Argo CD UI.
