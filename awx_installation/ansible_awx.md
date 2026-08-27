# Ansible AWX Installation Guide (Local Kind + WSL2 Setup)

This document provides a comprehensive end-to-end guide to installing and running **Ansible AWX (v24.6.1)** on a local Kubernetes cluster using **Kind (Kubernetes in Docker)** inside a **WSL2 (Windows Subsystem for Linux)** environment using **AWX Operator v2.19.1**.

---

## 1. High-Level Process Flow

```
+-----------------------------------------------------------------------------------+
|                                 HIGH-LEVEL FLOW                                   |
+-----------------------------------------------------------------------------------+
|  [Step 1: Environment & Repository Setup]                                        |
|   └── Clone AWX Operator repository & switch to target release tag (2.19.1).      |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼
+-----------------------------------------------------------------------------------+
|  [Step 2: Kustomize Configuration & Image Overrides]                             |
|   └── Configure kustomization.yaml to deploy AWX Operator.                        |
|   └── Override `gcr.io/kubebuilder/kube-rbac-proxy` -> `quay.io/brancz/...`        |
|       to resolve image pull failures on deprecated registries.                     |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼
+-----------------------------------------------------------------------------------+
|  [Step 3: AWX Operator Deployment]                                                |
|   └── Apply Kustomize manifests (`kubectl apply -k .`).                           |
|   └── Wait for operator pod to reach `2/2 Running` state.                         |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼
+-----------------------------------------------------------------------------------+
|  [Step 4: AWX Instance Manifest Creation & Deployment]                            |
|   └── Create Custom Resource definition `awx-demo.yml` (Service: NodePort).       |
|   └── Deploy instance (`kubectl apply -f awx-demo.yml`).                          |
|   └── Operator provisions PostgreSQL database, runs migrations, & deploys AWX.   |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼
+-----------------------------------------------------------------------------------+
|  [Step 5: Access & Authentication]                                                |
|   └── Establish port-forwarding (`0.0.0.0:8080` -> Service Port `80`).            |
|   └── Extract base64-decoded `awx-demo-admin-password` secret.                     |
|   └── Access UI via browser at `http://localhost:8080`.                           |
+-----------------------------------------------------------------------------------+
```

---

## 2. Step-by-Step Installation Commands & Explanations

### Step 1: Navigate & Switch to Release Tag

We start by navigating into the cloned `awx-operator` repository directory and switching to the stable release tag `2.19.1`. 

```bash
# Move into the AWX Operator source directory
cd ~/awx/awx-operator

# Switch to the verified stable release tag (2.19.1) instead of using the unstable 'devel' branch
git checkout tags/2.19.1

# Verify that git is pinned precisely to tag 2.19.1
git describe --tags --always
```
* **Why run this?** Running off the `devel` branch can expose your deployment to unreleased code or breaking changes. Pinned release tags guarantee repeatable deployment builds aligned with AWX 24.6.1.

---

### Step 2: Configure Kustomize & Image Overrides

Create or modify `kustomization.yaml` inside `~/awx/awx-operator/`. We must add an image override because `gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0` has been moved/deprecated on GCR, leading to `ErrImagePull` / `ImagePullBackOff`. We map it to its official Quay mirror (`quay.io/brancz/kube-rbac-proxy`).

```bash
# Open or create kustomization.yaml
nano kustomization.yaml
```

Insert the following content:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Upstream CRDs and deployment resources for AWX Operator 2.19.1
resources:
  - github.com/ansible/awx-operator/config/default?ref=2.19.1

images:
  # Ensure AWX Operator image is explicitly pinned
  - name: quay.io/ansible/awx-operator
    newTag: 2.19.1
  # Fix for ImagePullBackOff: redirect container registry from deprecated GCR to Quay
  - name: gcr.io/kubebuilder/kube-rbac-proxy
    newName: quay.io/brancz/kube-rbac-proxy
    newTag: v0.15.0

namespace: awx
```
* **Why run this?** Kustomize allows declarative customization of raw Kubernetes YAMLs without modifying upstream source code directly. The image mapping prevents container creation failure due to broken image paths.

---

### Step 3: Deploy the AWX Operator

Apply the Kustomize manifests to deploy CRDs, roles, service accounts, and the operator controller manager.

```bash
# Apply the Kustomization directory configuration into the cluster
kubectl apply -k .

# Watch the deployment process for the AWX Operator controller manager
kubectl get pods -n awx -w
```
* **Why run this?** The AWX Operator acts as the Kubernetes controller. It listens for AWX Custom Resources (CRs) and automates database provisioning, schema upgrades, configuration management, and pod lifecycle for AWX.
* **Expected Result:** `awx-operator-controller-manager-xxxxxxxxxx-xxxxx` reaching **`2/2 Running`**.

---

### Step 4: Define and Apply the AWX Custom Resource

Now create the actual AWX instance manifest (`awx-demo.yml`).

```bash
# Create the deployment manifest file
nano awx-demo.yml
```

Insert the following specification:

```yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-demo
  namespace: awx
spec:
  # Expose service as NodePort for easy cluster access
  service_type: nodeport
```

Apply the file:

```bash
# Instruct the AWX Operator to deploy an AWX instance based on awx-demo.yml
kubectl apply -f awx-demo.yml

# Monitor pod creation progress across all components
kubectl get pods -n awx -w
```
* **Why run this?** This custom resource signals the AWX Operator to initialize PostgreSQL 15, run database migration jobs (`awx-demo-migration-24.6.1`), spin up task worker pods (`awx-demo-task`), and spin up web interface pods (`awx-demo-web`).
* **Expected Result:**
  - `awx-demo-postgres-15-0` -> `1/1 Running`
  - `awx-demo-migration-xxxxx` -> `0/1 Completed`
  - `awx-demo-web-xxxxx` -> `3/3 Running`
  - `awx-demo-task-xxxxx` -> `4/4 Running`

---

### Step 5: Establish Network Access & Retrieve Admin Credentials

Because WSL2 runs inside a virtual network namespace separate from Windows host loopback, bind `kubectl port-forward` to `0.0.0.0`.

```bash
# Forward traffic from all network interfaces (0.0.0.0) on port 8080 to AWX service port 80
kubectl port-forward --address 0.0.0.0 svc/awx-demo-service -n awx 8080:80
```
*(Leave this command running in your terminal).*

Open a **new terminal tab** to decode the initial superuser password:

```bash
# Extract base64 encoded admin password secret and decode it to stdout
kubectl get secret awx-demo-admin-password -n awx -o jsonpath="{.data.password}" | base64 --decode; echo
```
* **Why run this?** AWX generates a randomized strong admin password stored securely inside a Kubernetes secret (`awx-demo-admin-password`). Decoding base64 allows initial login.

---

## 3. Detailed Summary

* **Architecture:** The AWX Operator deploys a microservices-based architecture on top of Kubernetes. PostgreSQL provides persistent state storage, Celery/Redis handles background task execution queues (within `awx-demo-task`), and uWSGI/Nginx serves the frontend web application (`awx-demo-web`).
* **Troubleshooting Highlights:**
  1. **RBAC Proxy Image Error:** Solved by mapping `gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0` to `quay.io/brancz/kube-rbac-proxy:v0.15.0` in `kustomization.yaml`.
  2. **WSL2 Networking:** Standard NodePort IP bindings aren't automatically forwarded to Windows host ports. Executing `kubectl port-forward --address 0.0.0.0` explicitly bridges WSL2 networking to Windows `http://localhost:8080`.
* **Resource Awareness:** AWX typically requires around 2-4 vCPUs and 4-8 GB RAM. Running this workload inside Docker-in-WSL2 consumes significant memory during migration phases; ensuring 8 GB total system RAM allocation prevents Out-Of-Memory (OOM) pod evictions.

---
