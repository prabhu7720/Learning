# 🚀 Argo CD on EKS – GitOps Made Easy

This README serves as a **complete hands-on guide** to installing and learning **Argo CD** on **Amazon EKS (Elastic Kubernetes Service)**.
It covers step-by-step installation, GitOps workflow, and essential Argo CD concepts with examples.

---

## 📚 What You’ll Learn

* ✅ How to **install Argo CD** on an existing **EKS Cluster**
* ✅ Understand **Argo CD Core Concepts** – Applications, Sync, Prune, and Self-Heal
* ✅ How to **access the Argo CD UI** securely
* ✅ How to **deploy a sample application** using GitOps
* ✅ Learn about **ApplicationSet** and **Rollouts (Canary / Blue-Green)** basics

---

## 🧠 Argo CD Overview

**Argo CD** (Argo Continuous Delivery) is a **GitOps tool** for Kubernetes that synchronizes Kubernetes manifests from Git repositories into your cluster.

### 🔑 Key Features

* Declarative Git-based configuration
* Real-time synchronization and drift detection
* Automated deployments (auto-sync)
* Integration with Helm, and plain YAML
* Rollback and version history

---

## 🧩 Argo CD Core Concepts

| Concept            | Description                                                                                         |
| ------------------ | --------------------------------------------------------------------------------------------------- |
| **Application**    | Defines what to deploy (source), and where (destination).                                           |
| **Source**         | Points to a Git repo, path, and revision that contains Kubernetes manifests or Helm charts.         |
| **Destination**    | Defines the target cluster and namespace for deployment.                                            |
| **Sync**           | Aligns the live cluster state with the Git repository state.                                        |
| **Prune**          | Deletes resources removed from the Git repo.                                                        |
| **Self-Heal**      | Automatically reverts manual cluster changes that drift from Git.                                   |
| **ApplicationSet** | Automates creation of multiple Applications dynamically (e.g., multi-cluster or multi-environment). |

---

## ⚙️ Prerequisites

Before you begin, ensure you have:

* 🟢 AWS CLI configured (`aws configure`)
* 🟢 `kubectl` installed
* 🟢 `eksctl` installed
* 🟢 `helm` installed
* 🟢 An **EKS cluster** already created and accessible
* 🟢 A **GitHub repository** with sample manifests (optional demo app)

---


## 🧩 Step 1B: Install and Configure kubectl
```bash
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --client
```

## ☸️ Step 1A: Connect to Your EKS Cluster

```bash
aws configure
aws eks update-kubeconfig --name <cluster-name> --region <region>
kubectl get nodes
```

## ⚙️ Step 1C: Install Helm 3

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Ensure nodes are listed and ready.

---

## 🚀 Step 2: Install Argo CD in EKS

Create a namespace:

```bash
kubectl create namespace argocd
```

Use this values.yaml for exposing argocd in NodePort
```yaml
# Basic Argo CD values for NodePort exposure
server:
  service:
    type: NodePort
    nodePortHttp: 30080        # optional — choose a fixed NodePort (HTTP)
    nodePortHttps: 30443       # optional — choose a fixed NodePort (HTTPS)
    ports:
      http: 80
      https: 443
  ingress:
    enabled: false
  insecure: true                # serve UI without enforcing HTTPS (for lab use)
  extraArgs:
    - --insecure                # disable TLS redirection in ArgoCD server

configs:
  cm:
    # Disable TLS redirect so UI works over plain HTTP
    server.insecure: "true"

redis:
  enabled: true

controller:
  replicas: 1
repoServer:
  replicas: 1
applicationSet:
  replicas: 1
```

Install Argo CD using the official Helm:
```bash
# Add and update the Argo Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install using the NodePort values file
helm upgrade --install argocd argo/argo-cd \
  -n argocd \
  -f values.yaml
```

Verify installation:

```bash
kubectl get pods -n argocd
```

---

## 🌐 Step 3: Access the Argo CD UI

Access via browser:
Allow ==30080== port in Worker Node Security Group 
```
https://<ec2-publicip>:30080
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o yaml
echo <base64secret> | base64 --decode
```

Login using:

```
Username: admin
Password: <decoded-password>
```

---

## 🧩 Step 4: Create Your First Application (Using the Argo CD UI)

Now that Argo CD is running, let’s create your first GitOps-managed application directly from the **Argo CD Web UI**.

### 🧭 Step 4.1 – Prepare Your Application Repository

Push the following YAML manifests to your GitHub repository (e.g., `guestbook-ui/` folder):

```yaml
# guestbook-ui/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook-ui
  template:
    metadata:
      labels:
        app: guestbook-ui
    spec:
      containers:
        - image: gcr.io/google-samples/gb-frontend:v5
          name: guestbook-ui
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: guestbook-ui
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: guestbook-ui
```

Make sure this folder is committed and pushed to your GitHub repo.

---

### 🖥️ Step 4.2 – Create an Application in Argo CD UI

1. Open the **Argo CD UI** → click **“NEW APP”**.
2. Fill in the following details:

| Field                | Value                                                    |
| -------------------- | -------------------------------------------------------- |
| **Application Name** | `guestbook-ui`                                           |
| **Project**          | `default`                                                |
| **Sync Policy**      | `Automatic` *(optional)*                                 |
| **Repository URL**   | `https://github.com/<your-username>/<your-repo>.git`     |
| **Revision**         | `HEAD`                                                   |
| **Path**             | `guestbook-ui`                                           |
| **Cluster URL**      | `https://kubernetes.default.svc` *(default EKS cluster)* |
| **Namespace**        | `default`                                                |

3. Click **“Create”** and then **“Sync”** to deploy.

---

### 🧩 Step 4.3 – Verify the Deployment

Check deployment status:

```bash
kubectl get pods -n default
kubectl get svc guestbook-ui -n default
```

Expected output:

```
NAME                            READY   STATUS    RESTARTS   AGE
guestbook-ui-xxxxx              1/1     Running   0          1m
```

🎉 **Congratulations!** Your first Argo CD-managed app is live on EKS — created entirely from the Argo CD UI.


---

## 🔁 Step 4.4: Sync Options in Argo CD

| Option          | Description                                     |
| --------------- | ----------------------------------------------- |
| **Manual Sync** | Deploys only when you click “Sync” or run CLI.  |
| **Auto Sync**   | Automatically applies changes when Git updates. |
| **Prune**       | Removes deleted resources from cluster.         |
| **Self-Heal**   | Fixes drifted or manually changed resources.    |

---

## 🧱 Step 4.5: Prune & Self-Heal Demo

1. Enable **Auto Sync** in UI or YAML.
2. Delete a file from Git repo.
3. Argo CD will **automatically prune** that resource.
4. Modify a resource directly on cluster — it will be **reverted (self-healed)**.

---

## ⚙️ BONUS: SYNC OPTIONS in Argo CD

When Argo CD performs a **sync**, it reconciles your **live cluster state** with what’s declared in **Git**.
You can control *how* this happens using **Sync Options** in your Application manifest (`.spec.syncPolicy.syncOptions`).

---

### 🧩 Commonly Used Sync Options

| **Sync Option**                       | **Purpose / Description**                                                                                                                      | **Default Behavior**            |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- |
| **`CreateNamespace=true`**            | Automatically creates the namespace if it doesn’t exist before applying resources.                                                             | ❌ Disabled                      |
| **`ApplyOutOfSyncOnly=true`**         | Only applies resources that are out-of-sync (doesn’t reapply unchanged ones).                                                                  | ❌ Disabled                      |
| **`PrunePropagationPolicy=<policy>`** | Controls *how pruning (deletion)* of removed resources is propagated.                                                                          | `Foreground`                    |
| **`PruneLast=true`**                  | Ensures prune happens **after** all other resources are applied. Useful for dependency order (e.g., deleting old config after new deployment). | ❌ Disabled                      |
| **`Replace=true`**                    | Replaces resources instead of patching them (delete + create). Useful when `kubectl apply` fails due to immutable field changes.               | ❌ Disabled                      |
| **`Validate=false`**                  | Skips `kubectl` schema validation when applying manifests.                                                                                     | ✅ Validation enabled by default |
| **`ServerSideApply=true`**            | Uses **Server-Side Apply (SSA)** instead of traditional client-side apply. Improves merge handling and ownership tracking.                     | ❌ Disabled                      |
| **`RespectIgnoreDifferences=true`**   | Ensures differences ignored in `.spec.ignoreDifferences` are not reapplied.                                                                    | ✅ Enabled                       |
| **`ApplyParallel=true`**              | Applies manifests in parallel instead of sequentially. Faster, but may break dependencies.                                                     | ❌ Disabled                      |

---

### ✅ Example: Sync Options in `values.yaml` or `Application` manifest

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
      - ServerSideApply=true
```

---

## 🪓 PRUNE in Argo CD

* **Pruning** = deleting resources from the cluster that **no longer exist in Git**.
  (Think of it as “cleaning up old manifests.”)

**Example:**
If you remove a Deployment from Git → Argo CD will delete that Deployment during the next sync (if pruning is enabled).

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
```

---

## 🌳 PRUNE PROPAGATION POLICY

This determines **how Kubernetes handles dependent resources** when pruning (deleting) a parent object.

### Available Options:

| **Policy**                 | **Meaning**                                                                               | **Behavior Example**                                                               |
| -------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Foreground** *(default)* | Delete dependents **before** deleting the parent.                                         | Deleting a Deployment deletes its ReplicaSets and Pods first, then the Deployment. |
| **Background**             | Delete parent **immediately**, dependents deleted in background by the garbage collector. | Deployment deleted first, then ReplicaSets/Pods later.                             |
| **Orphan**                 | Keep dependents (do not delete).                                                          | Delete Deployment but leave ReplicaSets and Pods intact.                           |

---

### 🧠 Example with Prune Propagation Policy

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - PrunePropagationPolicy=background
```

➡️ Here, if Argo CD prunes (deletes) a Deployment, it will tell Kubernetes:
“Delete it **now**, and let GC clean up child resources later.”

---

## 🧭 Summary Diagram

```
         ┌────────────┐
         │   Git Repo │
         └─────┬──────┘
               │
        ┌──────▼──────────┐
        │   Argo CD Sync  │
        └──────┬──────────┘
               │
    ┌──────────┼────────────────┐
    │          │                │
Apply Options  │         Prune (Delete)
   │           │
   ▼           ▼
(CreateNamespace, ServerSideApply, PrunePropagationPolicy)
```
---


## 🧩 Step 5: Deploy Multiple Environments Using ApplicationSet

**ApplicationSet** allows you to **automate the creation of multiple Argo CD Applications** dynamically — perfect for multi-environment or multi-cluster deployments.

### 🧭 Step 5.1 – What is ApplicationSet?

Instead of manually creating multiple Applications for `dev`, `stage`, and `prod`, an **ApplicationSet** automatically generates them based on a list, Git directories, clusters, or other generators.

### 🧱 Step 5.2 – Create a Multi-Environment ApplicationSet

Save the following YAML as `nginx-helm-multienv.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: nginx-helm-multienv
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - name: dev
            namespace: dev
            url: https://kubernetes.default.svc
          - name: prod
            namespace: prod
            url: https://kubernetes.default.svc
  template:
    metadata:
      # each generated Application will be named: <name>-nginx
      name: '{{name}}-nginx'
    spec:
      project: default
      # Helm chart source (Bitnami public Helm repo)
      source:
        repoURL: 'https://charts.bitnami.com/bitnami'
        chart: nginx
        # targetRevision can be pinned if required
        # targetRevision: 1.17.0
        helm:
          values: |
            replicaCount: 2
            service:
              type: ClusterIP
      destination:
        server: '{{url}}'
        namespace: '{{namespace}}'
      # optional: enable automated sync with prune + selfHeal for the demo
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

Apply it:

```bash
kubectl apply -f nginx-helm-multienv.yaml
```

Verify:

```bash
kubectl get applications -n argocd
```

You’ll see two Argo CD Applications automatically created:

```
NAME           SYNC STATUS   HEALTH STATUS
dev-nginx      Synced        Healthy
prod-nginx     Synced        Healthy
```

---

## 🌀 Step 6: Argo Rollouts – Canary Deployment Strategy

**Argo Rollouts** extends Kubernetes with **advanced deployment strategies** such as *Canary*, *Blue-Green*, and *Progressive Delivery*.

---

### ⚙️ Step 6.1 – Install Argo Rollouts

Install CRDs and controller:

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

Verify installation:

```bash
kubectl get pods -n argo-rollouts
```

Install Rollouts kubectl plugin (optional for visualization):

```bash
brew install argoproj/tap/kubectl-argo-rollouts
# or manually:
# curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
# chmod +x ./kubectl-argo-rollouts-linux-amd64 && mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

---

### 🚀 Step 6.2 – Deploy a Canary Rollout

Save the following as `rollout-demo.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: rollout-demo
  namespace: default
spec:
  replicas: 4
  strategy:
    canary:
      steps:
        - setWeight: 25
        - pause: {duration: 30s}
        - setWeight: 50
        - pause: {duration: 30s}
        - setWeight: 100
  selector:
    matchLabels:
      app: rollout-demo
  template:
    metadata:
      labels:
        app: rollout-demo
    spec:
      containers:
      - name: demo
        image: argoproj/rollouts-demo:blue
        ports:
        - containerPort: 8080
```

Apply it:

```bash
kubectl apply -f rollout-demo.yaml
```

Watch rollout progress:

```bash
kubectl argo rollouts get rollout rollout-demo --watch
```

You’ll see traffic gradually shift:

```
Step 1/5: SetWeight 25 -> Paused (30s)
Step 2/5: SetWeight 50 -> Paused (30s)
Step 3/5: SetWeight 100 -> Completed
```

---

### ✅ Step 6.3 – Verify Canary Deployment

Check pods and rollout status:

```bash
kubectl get rollouts
kubectl get pods -l app=rollout-demo
```

Output example:

```
NAME            STATUS        STEP  SET-WEIGHT
rollout-demo    Healthy       5/5   100
```

🎉 **Congratulations!** You have now successfully:

* Created a **multi-environment ApplicationSet**
* Installed **Argo Rollouts**
* Performed a **Canary deployment** on EKS

---

## 🧭 Summary

You’ve learned how to:

* Install Argo CD on EKS
* Access and manage Argo CD UI
* Create and manage Applications
* Understand Sync, Prune, and Self-Heal
* Deploy using Helm via ApplicationSet

---

## 📺 Watch the Complete ArgoCD Video

👉 [Learn ArgoCD & GitOps from Scratch for Continuous Deployment](https://youtu.be/sQTVWEJUV-w)

### 📞 Contact Us
**Phone:** [+91 91500 87745](tel:+919150087745)

### 💬 Ask Your Doubts
Join our **Discord Community**  
👉 [Click here to connect](https://discord.gg/N7GBNHBdqw)

### 📺 Explore More Learning
Subscribe to my **YouTube Channel** – *Learn With Mithran*  
🎯 [Watch Now](https://www.youtube.com/@LearnWithMithran)

---

