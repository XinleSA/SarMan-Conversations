# Production-Ready Kubernetes on Proxmox: A Corrected and Comprehensive Guide

**Author:** Manus AI
**Date:** February 22, 2026

## 1. Executive Overview

This document provides a comprehensive, corrected, and production-ready guide for deploying a Kubernetes cluster on a three-node Proxmox VE stack. It addresses the specific requirements of automated container updates, a secure lab-to-production application promotion pipeline, and deep integration with n8n for workflow automation and cluster management. This guide is a direct response to a review of a previously generated AI plan, and it systematically corrects all identified deficiencies to align with current industry best practices for security, stability, and operational efficiency.

The original plan, while ambitious, contained several critical flaws, including the use of a non-LTS Ubuntu release, outdated information regarding Kubernetes swap support, redundant and insecure architectural choices, and an unclear GitOps workflow. This guide rectifies these issues by:

- **Using Ubuntu 24.04 LTS:** Ensuring a stable, long-term supported foundation for the cluster.
- **Leveraging Modern Kubernetes Features:** Correctly implementing swap support via the `NodeSwap` feature gate, available since Kubernetes 1.28.
- **Adopting a Clean GitOps Model:** Utilizing ArgoCD as the single source of truth for deployments and implementing a proper Git-based workflow for image updates, deprecating the conflicting registry-polling approach.
- **Enforcing Security Best Practices:** Replacing long-lived ServiceAccount tokens with short-lived, auto-rotating tokens via the `TokenRequest` API for all integrations, including n8n.
- **Simplifying the Ingress Architecture:** Recommending a single, robust NGINX Ingress Controller, supplemented by MetalLB for bare-metal load balancing, removing the unnecessary complexity of a dual-proxy setup.

By following this guide, you will build a resilient, secure, and highly automated Kubernetes environment, complete with detailed checklists, ready-to-use automation scripts, and a clear operational framework for managing the entire application lifecycle.

## 2. Solution Architecture

This architecture is designed for a three-node Proxmox cluster, providing a balance of high availability, resource efficiency, and operational simplicity. It establishes a single Kubernetes cluster that uses namespace-based isolation to separate development, staging, and production environments.

### 2.1. Physical and Virtual Infrastructure

The foundation consists of three physical Proxmox VE hosts. On each host, we will provision a virtual machine that will serve as a Kubernetes node. This setup provides a layer of abstraction and simplifies management and recovery.

**Proxmox Host Configuration:**

| Host | Role | CPU | Memory | Storage |
|---|---|---|---|---|
| pve1 | Kubernetes Node 1 (Control Plane + Worker) | 8+ Cores | 32+ GB | 512+ GB NVMe |
| pve2 | Kubernetes Node 2 (Worker) | 8+ Cores | 32+ GB | 512+ GB NVMe |
| pve3 | Kubernetes Node 3 (Worker) | 8+ Cores | 32+ GB | 512+ GB NVMe |

**Kubernetes VM Configuration (per node):**

| VM Spec | Value | Justification |
|---|---|---|
| **OS** | Ubuntu 24.04 LTS | Provides a stable, secure, and long-term supported operating system. [1] |
| **vCPUs** | 8 | Sufficient for running the Kubernetes control plane and application workloads. |
| **Memory** | 16 GB | Provides ample memory for Kubernetes components and applications. |
| **Disk** | 100 GB | Provides space for the OS, container images, and logs. |

### 2.2. Kubernetes Cluster Architecture

The cluster will consist of one control-plane node and two worker nodes. For a small-scale production environment, co-locating a worker on the control-plane node is acceptable to maximize resource utilization.

```ascii
+--------------------------------------------------------------------+
| Proxmox VE Cluster                                                 |
|                                                                    |
|  +-----------------+    +-----------------+    +-----------------+  |
|  |    Proxmox 1    |    |    Proxmox 2    |    |    Proxmox 3    |  |
|  | +-------------+ |    | +-------------+ |    | +-------------+ |  |
|  | |   K8s VM 1  | |    | |   K8s VM 2  | |    | |   K8s VM 3  | |  |
|  | |-------------| |    | |-------------| |    | |-------------| |  |
|  | | Control Pln | |    | |   Worker    | |    | |   Worker    | |  |
|  | |   Worker    | |    | |   Node      | |    | |   Node      | |  |
|  | +-------------+ |    | +-------------+ |    | +-------------+ |  |
|  +-----------------+    +-----------------+    +-----------------+  |
|                                                                    |
+--------------------------------------------------------------------+
```

### 2.3. Core Kubernetes Components

| Component | Tool/Implementation | Justification |
|---|---|---|
| **Container Runtime** | `containerd` | The industry-standard, lightweight, and efficient container runtime for Kubernetes. |
| **CNI Plugin** | Calico | Provides robust networking and network policy enforcement. |
| **Cluster Provisioning** | `kubeadm` | The standard tool for creating production-ready Kubernetes clusters. |
| **Persistent Storage** | Longhorn | A cloud-native, distributed block storage system that is easy to deploy and manage. |
| **Ingress Controller** | NGINX Ingress Controller | The de-facto standard for managing external access to HTTP/S services in Kubernetes. |
| **Load Balancer** | MetalLB | Provides a network load-balancer implementation for bare-metal Kubernetes clusters. |
| **Certificate Management**| cert-manager | Automates the management and issuance of TLS certificates from Let's Encrypt. |

### 2.4. GitOps and Automation Architecture

This architecture places Git at the center of all deployment and configuration management activities. ArgoCD acts as the GitOps controller, ensuring the cluster state always matches the desired state defined in a Git repository.

**Key Workflow:**

1.  **Code Change:** A developer pushes a code change to an application repository.
2.  **CI Pipeline:** A CI pipeline (e.g., GitHub Actions, GitLab CI) builds a new container image and pushes it to a container registry.
3.  **Image Tag Update:** The CI pipeline automatically updates the image tag in the Kubernetes manifests within a dedicated GitOps repository.
4.  **ArgoCD Sync:** ArgoCD detects the change in the GitOps repository and automatically syncs the new manifest to the cluster, deploying the updated application.

**n8n Integration:**

n8n provides a powerful orchestration layer for automating manual approval gates, routing alerts, and enabling interactive cluster management.

- **Promotion Gating:** For promotions to production, the CI pipeline will trigger an n8n workflow. n8n will send a notification (e.g., via Telegram) with "Approve" and "Deny" buttons. On approval, n8n will merge the pull request in the GitOps repository, allowing ArgoCD to proceed with the deployment.
- **Alerting:** Prometheus Alertmanager will send all alerts to an n8n webhook. n8n can then enrich, deduplicate, and route these alerts to different channels based on severity.
- **ChatOps:** An n8n workflow connected to a Telegram bot will allow authorized users to perform basic cluster operations (e.g., checking status, rolling back a deployment) by sending commands.

This corrected architecture provides a secure, automated, and observable platform for running containerized applications. The following sections will provide the detailed steps and scripts to implement it.

## 3. Correcting the Deficiencies

This guide explicitly addresses and corrects the six major deficiencies identified in the original AI-generated plan. Understanding these corrections is crucial for building a robust and maintainable cluster.

| Deficiency | Original Flaw | Corrected Approach & Justification |
|---|---|---|
| **1. Operating System** | Recommended Ubuntu 25.10, a non-LTS release with a 9-month support cycle. | **Use Ubuntu 24.04 LTS.** Production systems require stability and long-term support. LTS releases provide security updates for 5 years, making them the only suitable choice for critical infrastructure like a Kubernetes cluster. [1] |
| **2. Swap Management** | Stated that swap must be disabled, which is outdated information. | **Enable and configure swap support.** Since Kubernetes v1.28, the `NodeSwap` feature gate allows nodes to use swap memory. This can improve node stability and resource utilization, especially for workloads that may benefit from swap. We will enable this feature and configure it correctly. [2] |
| **3. GitOps Tooling** | Recommended using both Flux CD and ArgoCD, creating unnecessary complexity. | **Standardize on ArgoCD.** Flux and ArgoCD serve the same purpose. Using both introduces redundancy and potential conflicts. ArgoCD is a mature, popular, and feature-rich choice for implementing GitOps. We will use ArgoCD exclusively. |
| **4. n8n Security** | Recommended using a long-lived, non-expiring ServiceAccount token for n8n integration. | **Use short-lived tokens via the `TokenRequest` API.** Long-lived tokens are a major security risk. The corrected approach involves creating a ServiceAccount for n8n and then using the `TokenRequest` API (via `kubectl create token`) within the n8n workflow itself to generate temporary, auto-rotating tokens. This drastically reduces the security exposure. [4] |
| **5. Ingress Architecture** | Proposed a redundant architecture with Nginx Proxy Manager (NPM) in front of the NGINX Ingress Controller. | **Use NGINX Ingress Controller with MetalLB.** An ingress controller is the designated entry point for cluster traffic. Adding another proxy in front is unnecessary. The standard and efficient approach for bare-metal clusters is to use a robust ingress controller (NGINX) and a bare-metal load balancer solution (MetalLB) to expose it. [5] |
| **6. Container Updates** | Suggested a confusing model using Keel for registry polling alongside GitOps. | **Implement a pure GitOps update workflow.** In a true GitOps model, Git is the single source of truth. Image updates should be triggered by a CI pipeline that builds a new image, pushes it to a registry, and then commits the updated image tag to the GitOps repository. ArgoCD then syncs this change. This provides a clear, auditable, and conflict-free update process, making tools like Keel redundant. [3] |

## 4. Phase 1: Node Preparation with Ansible

This phase focuses on preparing the Ubuntu 24.04 LTS virtual machines to become Kubernetes nodes. We will use Ansible to automate this process, ensuring consistency and repeatability across all three nodes.

### 4.1. Prerequisites

- **Ansible Control Node:** You need a machine with Ansible installed. This can be your local laptop or a dedicated bastion host.
- **SSH Access:** Ensure you have passwordless SSH access (using SSH keys) from your Ansible control node to all three Kubernetes VMs. The user must have `sudo` privileges.

### 4.2. Ansible Inventory

Create an `inventory.ini` file on your Ansible control node to define the hosts that Ansible will manage. Replace the IP addresses with the actual IPs of your VMs.

**File: `automation_scripts/ansible/inventory.ini`**
```ini
[k8s_cluster]
k8s-control-plane ansible_host=192.168.1.10
k8s-worker-1 ansible_host=192.168.1.11
k8s-worker-2 ansible_host=192.168.1.12

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### 4.3. Ansible Playbook

This playbook, `setup_nodes.yml`, automates all the necessary steps to prepare a node for `kubeadm`.

**Key Tasks Performed by the Playbook:**

1.  **System Update:** Updates the `apt` package cache.
2.  **Install Dependencies:** Installs `containerd`, `apt-transport-https`, and other necessary tools.
3.  **Configure `containerd`:** Sets up the `containerd` configuration file and enables the `SystemdCgroup` driver, which is required by Kubernetes.
4.  **Kernel Modules:** Ensures the `overlay` and `br_netfilter` kernel modules are loaded on boot.
5.  **Sysctl Settings:** Configures kernel parameters required for Kubernetes networking.
6.  **Install Kubernetes Components:** Adds the official Kubernetes `apt` repository and installs `kubeadm`, `kubelet`, and `kubectl`.
7.  **Hold Versions:** Pins the versions of the Kubernetes components to prevent unintended automatic upgrades.

*(The complete playbook is located in the `automation_scripts/ansible/` directory.)*

### 4.4. Running the Playbook

From your Ansible control node, execute the playbook:

```bash
ansible-playbook -i automation_scripts/ansible/inventory.ini automation_scripts/ansible/setup_nodes.yml
```

After the playbook completes successfully, all three VMs will be ready for Kubernetes installation.

## 5. Phase 2: Kubernetes Cluster Installation with `kubeadm`

With the nodes prepared, we will now use the `kubeadm` tool to bootstrap the Kubernetes cluster.

### 5.1. Initializing the Control Plane

SSH into the designated control plane node (`k8s-control-plane`).

First, create a `kubeadm-config.yaml` file. This approach is preferred over command-line flags as it allows for more configuration options, including enabling the `NodeSwap` feature gate.

**File: `kubeadm-config.yaml`**
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "v1.30.0"
controlPlaneEndpoint: "k8s-api.your-domain.com:6443" # Replace with your load-balanced endpoint or the control plane IP
podNetworkCidr: "10.244.0.0/16"
featureGates:
  NodeSwap: true
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
failSwapOn: false
```

Now, run `kubeadm init` with this configuration file:

```bash
sudo kubeadm init --config kubeadm-config.yaml --upload-certs
```

This command will perform the following actions:
- Run a series of pre-flight checks to ensure the node is ready.
- Generate the necessary certificates and keys for the cluster.
- Initialize the control plane components (etcd, API server, scheduler, controller manager).
- Generate a `kubeadm join` command with a token for adding worker nodes.

**Important:** The output will contain a `kubeadm join` command. Copy this command and save it in a secure location. You will need it to add the worker nodes.

After initialization, configure `kubectl` for your regular user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 5.2. Installing the CNI Plugin (Calico)

A Container Network Interface (CNI) plugin is required for pods to communicate with each other. We will use Calico, a popular and powerful CNI provider.

Apply the Calico manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

### 5.3. Joining the Worker Nodes

SSH into each of the two worker nodes (`k8s-worker-1` and `k8s-worker-2`).

On each worker, run the `kubeadm join` command that you saved from the control plane initialization. You must run this command with `sudo`.

```bash
sudo kubeadm join k8s-api.your-domain.com:6443 --token <your-token> --discovery-token-ca-cert-hash sha256:<your-hash>
```

### 5.4. Verifying the Cluster

Return to the control plane node. Verify that all nodes have successfully joined the cluster and are in the `Ready` state.

```bash
kubectl get nodes -o wide
```

The output should look similar to this, with all nodes showing a `Ready` status:

```
NAME                  STATUS   ROLES           AGE   VERSION    INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-control-plane   Ready    control-plane   10m   v1.30.0    192.168.1.10   <none>        Ubuntu 24.04 LTS   6.8.0-31-generic    containerd://1.7.13
k8s-worker-1        Ready    <none>          2m    v1.30.0    192.168.1.11   <none>        Ubuntu 24.04 LTS   6.8.0-31-generic    containerd://1.7.13
k8s-worker-2        Ready    <none>          2m    v1.30.0    192.168.1.12   <none>        Ubuntu 24.04 LTS   6.8.0-31-generic    containerd://1.7.13
```

At this point, you have a fully functional, three-node Kubernetes cluster.

## 6. Phase 3: Cluster Storage Configuration

Stateful applications require a persistent storage solution that is independent of the node lifecycle. For this, we will deploy Longhorn, a cloud-native, distributed block storage system that is lightweight, easy to use, and provides enterprise-grade features like snapshots, backups, and replication.

### 6.1. Installing Longhorn

Longhorn is installed via a single manifest file. It will automatically discover the storage available on each node and create a storage pool.

**Prerequisites:**

- `open-iscsi` package must be installed on all Kubernetes nodes. The Ansible playbook in Phase 1 should have handled this, but you can verify with `dpkg -l | grep open-iscsi`.

**Installation Command:**

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml
```

This command creates the `longhorn-system` namespace and deploys all the necessary components, including the Longhorn manager, UI, and a daemonset that runs on each node.

### 6.2. Accessing the Longhorn UI

To manage Longhorn, you can expose its UI through an Ingress. First, you need to create a basic authentication secret.

**1. Create a `htpasswd` file:**
```bash
htpasswd -c auth "your-username"
```

**2. Create the secret in the `longhorn-system` namespace:**
```bash
kubectl -n longhorn-system create secret generic basic-auth --from-file=auth
```

**3. Create the Ingress manifest:**

**File: `automation_scripts/kubernetes/longhorn-ingress.yaml`**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
spec:
  ingressClassName: nginx
  rules:
  - host: longhorn.your-domain.com # Replace with your domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
  tls:
  - hosts:
    - longhorn.your-domain.com # Replace with your domain
    secretName: longhorn-tls
```

Apply the manifest to access the UI at `https://longhorn.your-domain.com`.

### 6.3. Setting the Default StorageClass

After installation, Longhorn creates a StorageClass named `longhorn`. It is best practice to set this as the default StorageClass for the cluster, so that any `PersistentVolumeClaim` that does not specify a `storageClassName` will automatically use Longhorn.

```bash
kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## 7. Phase 4: Networking, Ingress, and Certificates

This phase configures how your cluster communicates with the outside world. We will deploy MetalLB to provide `LoadBalancer` services, an NGINX Ingress Controller to manage HTTP/S traffic, and cert-manager to automate TLS certificates.

### 7.1. Bare-Metal Load Balancer (MetalLB)

In cloud environments, a `LoadBalancer` service automatically provisions a cloud provider's load balancer. On bare metal, we need a tool to provide this functionality. MetalLB fills this gap.

**1. Installation:**
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
```

**2. IP Address Pool Configuration:**

MetalLB needs to be told which IP addresses it is allowed to use. Create a manifest to define an `IPAddressPool`. This range should be part of your local network but reserved for MetalLB to avoid IP conflicts.

**File: `automation_scripts/kubernetes/metallb-config.yaml`**
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250 # Adjust this range for your network
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
```

Apply the configuration:
```bash
kubectl apply -f automation_scripts/kubernetes/metallb-config.yaml
```

### 7.2. Ingress Controller (NGINX)

The NGINX Ingress Controller is the standard for managing external access to services in the cluster. It provides routing, SSL termination, and more.

**Installation:**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml
```

After applying this manifest, a `LoadBalancer` service will be created for the ingress controller. MetalLB will automatically assign it an external IP address from the pool you defined. You can find this IP by running:

```bash
kubectl get svc -n ingress-nginx
```

### 7.3. Certificate Management (cert-manager)

cert-manager automates the process of obtaining and renewing TLS certificates from sources like Let's Encrypt.

**1. Installation:**
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```

**2. Configure a `ClusterIssuer`:**

A `ClusterIssuer` is a cluster-wide resource that can issue certificates in any namespace. We will create one for Let's Encrypt's production environment.

**File: `automation_scripts/kubernetes/letsencrypt-issuer.yaml`**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com # IMPORTANT: Replace with your email
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

Apply the issuer:
```bash
kubectl apply -f automation_scripts/kubernetes/letsencrypt-issuer.yaml
```

Now, any Ingress resource annotated with `cert-manager.io/cluster-issuer: "letsencrypt-prod"` will automatically have a TLS certificate issued for it.

## 8. Phase 5: Environment Strategy with Namespaces

For a single-cluster setup, Kubernetes namespaces are the primary mechanism for isolating environments. We will create namespaces for `lab`, `staging`, and `production` workloads, as well as for system components.

### 8.1. Namespace Creation

Create the namespaces using `kubectl`:

```bash
kubectl create namespace lab
kubectl create namespace staging
kubectl create namespace production
kubectl create namespace monitoring # For Prometheus/Grafana
kubectl create namespace n8n-automation # For n8n integration components
```

### 8.2. RBAC and Resource Quotas

To enforce isolation, you should apply Role-Based Access Control (RBAC) and `ResourceQuota` objects to each namespace.

- **RBAC:** Create `Role` and `RoleBinding` objects to control which users or service accounts can access resources in each namespace.
- **ResourceQuotas:** Limit the total amount of CPU, memory, and storage that can be consumed by all pods in a namespace.

**Example `ResourceQuota` for the `lab` namespace:**

**File: `automation_scripts/kubernetes/lab-quota.yaml`**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab-quota
  namespace: lab
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "50"
    count/persistentvolumeclaims: "20"
```

## 9. Phase 6: GitOps with ArgoCD

This phase implements the core of our automation strategy: GitOps using ArgoCD. The Git repository becomes the single source of truth for our cluster's desired state.

### 9.1. Installing ArgoCD

Create a dedicated namespace for ArgoCD and apply the installation manifest.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 9.2. Accessing the ArgoCD Server

By default, the ArgoCD API server is not exposed externally. We'll change its service type to `LoadBalancer` so MetalLB can assign it an IP.

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Get the initial admin password, which is stored in a secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

You can now access the ArgoCD UI at the external IP assigned to the `argocd-server` service.

### 9.3. Structuring the GitOps Repository

A well-structured GitOps repository is key to managing multiple environments. We will use a Kustomize-based approach.

**Recommended Repository Structure:**

```
├── apps
│   ├── my-app
│   │   ├── base
│   │   │   ├── deployment.yaml
│   │   │   ├── kustomization.yaml
│   │   │   └── service.yaml
│   │   └── overlays
│   │       ├── lab
│   │       │   ├── kustomization.yaml
│   │       │   └── patch-replicas.yaml
│   │       └── production
│   │           ├── kustomization.yaml
│   │           └── patch-resources.yaml
└── cluster
    ├── core
    │   ├── cert-manager.yaml
    │   └── metallb.yaml
    └── argocd
        └── root-app.yaml
```

- **`apps/`**: Contains the manifests for your applications.
- **`apps/my-app/base`**: The common, environment-agnostic manifests for `my-app`.
- **`apps/my-app/overlays/`**: Environment-specific patches (e.g., different replica counts, resource limits).
- **`cluster/`**: Contains the manifests for cluster-wide components.

### 9.4. The App of Apps Pattern

Instead of manually creating applications in ArgoCD, we use the "App of Apps" pattern. A single "root" ArgoCD application tells ArgoCD to look in a specific directory in your Git repo, and it automatically creates all the applications defined there.

**Example Root Application:**

**File: `automation_scripts/kubernetes/argocd-root-app.yaml`**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/your-org/k8s-gitops-repo.git' # Your GitOps repo
    targetRevision: HEAD
    path: apps
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Once you apply this root app to your cluster, ArgoCD will take over and manage all your other applications automatically.

## 10. Phase 7: n8n Integration for Secure Automation

Integrating n8n allows for powerful orchestration of cluster and application lifecycle events. The key to this integration is ensuring it is done securely, without resorting to risky practices like long-lived tokens.

### 10.1. The Secure Token Generation Model

Instead of storing a static, powerful token in n8n, we will empower n8n to request temporary, short-lived tokens from the Kubernetes API on-demand. This is the modern, secure approach.

**The Workflow:**
1.  **Initial Token:** A single, long-lived token with *only* the permission to create other tokens (`serviceaccounts/token`) is created and stored as a credential in n8n. This is the *only* long-lived token in the system.
2.  **Workflow Trigger:** An n8n workflow starts.
3.  **Token Request:** The first step in the workflow is an HTTP request to the Kubernetes API, using the initial token, to call the `TokenRequest` API for a specific ServiceAccount.
4.  **Receive Short-Lived Token:** The API returns a new, short-lived token (e.g., valid for 15 minutes) with a specific set of permissions.
5.  **Execute Actions:** The rest of the n8n workflow uses this new, temporary token to perform its tasks (e.g., patch a deployment, get pod status).
6.  **Token Expiration:** After its short lifetime, the temporary token automatically becomes invalid.

This model ensures that even if a token is compromised during a workflow execution, its usefulness to an attacker is extremely limited in time and scope.

### 10.2. Setting up the n8n ServiceAccount and RBAC

We need to create two ServiceAccounts:
- `n8n-token-creator-sa`: Has the single permission to create tokens.
- `n8n-executor-sa`: Has the permissions needed to manage applications (e.g., edit deployments, get logs).

**File: `automation_scripts/kubernetes/n8n-rbac.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: n8n-automation
---
# Service Account for creating tokens
apiVersion: v1
kind: ServiceAccount
metadata:
  name: n8n-token-creator-sa
  namespace: n8n-automation
---
# Role that ONLY allows token creation
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: token-creator-role
  namespace: n8n-automation
rules:
- apiGroups: [""]
  resources: ["serviceaccounts/token"]
  verbs: ["create"]
---
# Bind the role to the Service Account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: token-creator-binding
  namespace: n8n-automation
subjects:
- kind: ServiceAccount
  name: n8n-token-creator-sa
roleRef:
  kind: Role
  name: token-creator-role
  apiGroup: rbac.authorization.k8s.io
---
# Service Account for executing tasks
apiVersion: v1
kind: ServiceAccount
metadata:
  name: n8n-executor-sa
  namespace: n8n-automation
---
# Role with permissions for application management
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-manager-role
  namespace: production # Or other target namespaces
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "patch"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
# Bind the app manager role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-manager-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: n8n-executor-sa
  namespace: n8n-automation
roleRef:
  kind: Role
  name: app-manager-role
  apiGroup: rbac.authorization.k8s.io
```

### 10.3. Example n8n Token Generation Workflow

An example n8n workflow (`automation_scripts/n8n/generate_k8s_token.json`) demonstrates how to use the `n8n-token-creator-sa`'s token to generate a short-lived token for the `n8n-executor-sa`.

## 11. Phase 8: The GitOps Container Update Pipeline

This section details the correct, Git-native way to handle container image updates, replacing the flawed Keel-based model. The core principle is that any change to the running system, including a new image tag, must be reflected in the Git repository.

### 11.1. The CI/CD and GitOps Flow

1.  **Developer Commits Code:** A developer pushes code changes to an application's source code repository.
2.  **CI Pipeline Triggers:** A CI pipeline (e.g., GitHub Actions) is triggered by the push.
3.  **Build and Push Image:** The CI pipeline builds a new container image, tags it (e.g., with the Git commit SHA), and pushes it to your container registry.
4.  **Update Manifests in Git:** This is the crucial step. The CI pipeline then checks out the **GitOps repository** and updates the Kustomize or Helm manifests to point to the new image tag.
5.  **Commit and Push to GitOps Repo:** The pipeline commits and pushes this change to the GitOps repository.
6.  **ArgoCD Syncs:** ArgoCD, which is monitoring the GitOps repository, detects the change and syncs the cluster state, pulling the new image and updating the deployment.

This creates a fully auditable trail. The Git history shows exactly which code commit corresponds to which deployed version.

### 11.2. Automating the Manifest Update with Argo CD Image Updater

While the CI pipeline can script the manifest update, a more integrated tool is the **Argo CD Image Updater**. This is a companion controller for ArgoCD that automates the process.

**How it Works:**
1.  You annotate your ArgoCD `Application` resources with instructions on which images to watch.
2.  The Image Updater periodically scans your container registry for new tags that match a specified versioning strategy (e.g., semantic versioning).
3.  When it finds a new, eligible tag, it automatically commits the change to the image tag in your GitOps repository on your behalf.
4.  ArgoCD detects this commit and syncs the application as usual.

This approach keeps the update logic within the GitOps ecosystem and simplifies the CI pipeline, which now only needs to build and push the image.

**Installation:**

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

**Example Annotation for an ArgoCD Application:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: my-app=your-registry/my-app
    argocd-image-updater.argoproj.io/my-app.update-strategy: semver
    argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/git-creds
```

## 12. Phase 9: Monitoring, Logging, and Alerting

A production cluster is incomplete without a robust monitoring and alerting stack. We will use the `kube-prometheus-stack`, which is the industry standard.

### 12.1. Deploying the `kube-prometheus-stack`

This stack is best deployed using its official Helm chart.

**1. Add the Helm repository:**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

**2. Install the stack:**

We will install it into the `monitoring` namespace we created earlier.

```bash
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

This single command deploys:
- **Prometheus:** For time-series metrics collection and storage.
- **Grafana:** For building and viewing dashboards.
- **Alertmanager:** For handling and routing alerts.
- **Node Exporter:** For collecting node-level metrics.
- **kube-state-metrics:** For collecting metrics on Kubernetes API objects.

### 12.2. Configuring Alertmanager with n8n

Instead of having Alertmanager send notifications directly, we can route them through n8n to create more intelligent alerting workflows.

**1. Create an n8n Webhook:** In your n8n instance, create a new workflow with a Webhook trigger. This will give you a webhook URL.

**2. Configure Alertmanager:**

We need to provide custom values to the Helm chart to configure Alertmanager. Create a `prometheus-values.yaml` file.

**File: `prometheus-values.yaml`**
```yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ["job"]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: "n8n-webhook"
    receivers:
    - name: "n8n-webhook"
      webhook_configs:
      - url: "http://your-n8n-instance/webhook/your-webhook-id"
```

Upgrade the Helm release with these values:

```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack --namespace monitoring -f prometheus-values.yaml
```

Now, all alerts will be sent to your n8n workflow, where you can parse them, enrich them with data from other sources, and route them to Telegram, Slack, email, or any other service based on their severity or labels.

## 13. Phase 10: Backup, Disaster Recovery, and Restore

No production system is complete without a solid backup and disaster recovery (DR) plan. We will use Velero, the open-source standard for Kubernetes backup and restore.

### 13.1. Velero Architecture

Velero works by:
1.  Querying the Kubernetes API to get the state of resources.
2.  Storing the resulting JSON objects in a cloud object store (like S3, MinIO, etc.).
3.  For persistent volumes, it integrates with the storage provider (Longhorn in our case) to create snapshots and back them up to the same object store.

### 13.2. Installing and Configuring Velero

**1. Prerequisites:**
- **Object Storage:** You need an S3-compatible object storage bucket. For a self-hosted setup, you can deploy MinIO in your cluster or use an external service.
- **S3 Credentials:** A credentials file for accessing your bucket.

**2. Download the Velero CLI:** Get the latest release from the [Velero GitHub page](https://github.com/vmware-tanzu/velero/releases).

**3. Install Velero:**

The `velero install` command is the easiest way to get started. It will configure the Velero deployment, ServiceAccount, and necessary RBAC.

```bash
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.9.0 \
    --bucket your-velero-bucket-name \
    --secret-file ./your-s3-credentials-file \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://your-minio-url \
    --use-volume-snapshots=true
```

This command assumes you are using an S3-compatible provider like MinIO. Adjust the flags for your specific provider.

### 13.3. Performing Backups

**On-Demand Backup:**

To back up all resources in the `production` namespace:

```bash
velero backup create prod-backup-20260222 --include-namespaces production
```

**Scheduled Backup:**

It is crucial to have automated, scheduled backups. To back up the entire cluster every 24 hours:

```bash
velero schedule create daily-full-cluster-backup --schedule="@every 24h"
```

### 13.4. Disaster Recovery and Restore

In a disaster scenario where your cluster is lost, the recovery process is:
1.  Build a new, clean Kubernetes cluster following the steps in this guide.
2.  Install Velero on the new cluster, pointing it to the *same* object storage bucket.
3.  Verify Velero can see the existing backups:
    ```bash
    velero backup get
    ```
4.  Restore from your latest backup:
    ```bash
    velero restore create --from-backup daily-full-cluster-backup-202602221200
    ```

Velero will recreate all the resources and, through its integration with Longhorn, restore the persistent volume data, bringing your applications back online.

## 14. Phase 11: Interactive Cluster Management with a Telegram Bot

Building on the n8n integration, we can create a powerful ChatOps bot that allows authorized users to interact with the Kubernetes cluster from a Telegram chat. This enables quick status checks, simple operations, and receiving critical alerts on the go.

### 14.1. Workflow Overview

The core of the Telegram bot is an n8n workflow that:
1.  **Listens for Commands:** Uses the Telegram Trigger node to listen for messages that start with `/` (e.g., `/status`, `/pods`).
2.  **Authorizes User:** Checks if the Telegram user ID of the sender is on an approved list.
3.  **Generates a Token:** Executes the secure token generation flow described in Phase 7 to get a short-lived `kubectl` token.
4.  **Parses Command and Executes:** Parses the incoming command, constructs a safe `kubectl` command, and executes it using the Execute Command node.
5.  **Returns Output:** Formats the `stdout` from the command and sends it back to the user in the Telegram chat.

### 14.2. Setting up the Telegram Bot

1.  **Create a Bot:** Talk to the `@BotFather` on Telegram to create a new bot. It will give you a unique HTTP API token.
2.  **Create n8n Credentials:** In your n8n instance, create a new credential for the Telegram API and paste the token you received.
3.  **Build the Workflow:** Create a new workflow in n8n based on the `automation_scripts/n8n/telegram_bot_workflow.json` template. Configure the Telegram Trigger node with your new credentials and add the logic for authorization and command execution.

### 14.3. Recommended Command Set

Start with a limited, read-only set of commands to ensure safety. You can expand this later as you gain confidence in the workflow.

| Command | Description | Example `kubectl` command |
|---|---|---|
| `/status` | Shows the status of all nodes in the cluster. | `kubectl get nodes -o wide` |
| `/pods <namespace>` | Lists all pods in a specified namespace. | `kubectl get pods -n <namespace>` |
| `/logs <pod> -n <ns>` | Retrieves the logs for a specific pod. | `kubectl logs <pod> -n <ns>` |
| `/describe <type> <name>` | Runs `kubectl describe` on a resource. | `kubectl describe <type> <name>` |

**Security Note:** Be extremely cautious when adding commands that modify the cluster state (e.g., `delete`, `scale`, `apply`). Implement strict authorization and consider adding a second confirmation step within the chat before executing such commands.

## 15. Phase 12: Cluster Upgrade Strategy

Keeping your Kubernetes cluster up-to-date is critical for security and access to new features. `kubeadm` provides a structured process for upgrading a cluster.

### 15.1. The `kubeadm` Upgrade Process

The general process for a minor version upgrade (e.g., v1.30.0 to v1.31.0) is as follows:

1.  **Upgrade the Control Plane:**
    a. On the control plane node, update the `apt` cache and install the target version of `kubeadm`.
    b. Run `sudo kubeadm upgrade plan` to see the proposed upgrade and check for any issues.
    c. Run `sudo kubeadm upgrade apply v1.31.0` (or the target version).
    d. Drain the control plane node: `kubectl drain <control-plane-node-name> --ignore-daemonsets`.
    e. Upgrade `kubelet` and `kubectl`: `sudo apt-get install -y kubelet=1.31.0-00 kubectl=1.31.0-00`.
    f. Restart the `kubelet`: `sudo systemctl restart kubelet`.
    g. Uncordon the node: `kubectl uncordon <control-plane-node-name>`.

2.  **Upgrade the Worker Nodes:**
    a. On each worker node, one at a time, drain the node from the control plane: `kubectl drain <worker-node-name> --ignore-daemonsets`.
    b. On the worker node itself, upgrade `kubeadm`: `sudo apt-get install -y kubeadm=1.31.0-00`.
    c. Run `sudo kubeadm upgrade node`.
    d. Upgrade the `kubelet`: `sudo apt-get install -y kubelet=1.31.0-00`.
    e. Restart the `kubelet`: `sudo systemctl restart kubelet`.
    f. From the control plane, uncordon the worker node: `kubectl uncordon <worker-node-name>`.

### 15.2. Best Practices for Upgrades

- **Read the Changelog:** Always read the Kubernetes changelog for the target version to be aware of any breaking changes or deprecations.
- **Backup First:** Perform a full cluster backup with Velero before starting any upgrade.
- **Upgrade One Minor Version at a Time:** Do not skip minor versions (e.g., do not upgrade from 1.29 to 1.31 directly).
- **Upgrade Staging First:** If you have a staging cluster, always perform the upgrade there first to identify any potential issues with your workloads.

## 16. Quick Reference: Essential `kubectl` Commands

This section provides a quick reference for common `kubectl` commands that are useful for daily operations.

| Command | Description |
|---|---|
| `kubectl get nodes` | List all nodes in the cluster and their status. |
| `kubectl get pods -n <ns>` | List all pods in a specific namespace. |
| `kubectl get all -A` | List all resources (pods, services, deployments, etc.) in all namespaces. |
| `kubectl describe pod <pod> -n <ns>` | Show detailed information about a specific pod, including events. |
| `kubectl logs <pod> -n <ns>` | Print the logs for a specific pod. Use `-f` to follow the log stream. |
| `kubectl exec -it <pod> -n <ns> -- /bin/bash` | Get an interactive shell inside a running pod. |
| `kubectl apply -f <file.yaml>` | Create or update a resource from a YAML file. |
| `kubectl delete -f <file.yaml>` | Delete a resource defined in a YAML file. |
| `kubectl top nodes` | Show CPU and memory usage for all nodes. |
| `kubectl top pods -n <ns>` | Show CPU and memory usage for pods in a namespace. |
| `kubectl get events -n <ns> --sort-by=.metadata.creationTimestamp` | View the most recent events in a namespace, useful for debugging. |
| `kubectl config get-contexts` | List all available Kubernetes contexts. |
| `kubectl config use-context <context-name>` | Switch to a different Kubernetes context. |

## 17. References

[1] Ubuntu. (n.d.). *Ubuntu release cycle*. Retrieved from https://ubuntu.com/about/release-cycle
[2] Kubernetes. (2023, August 24). *Kubernetes 1.28: Beta support for using swap on Linux*. Retrieved from https://kubernetes.io/blog/2023/08/24/swap-linux-beta/
[3] Argo CD Image Updater. (n.d.). *Argo CD Image Updater*. Retrieved from https://argocd-image-updater.readthedocs.io/
[4] Kubernetes. (2024, November 19). *Service Accounts*. Retrieved from https://kubernetes.io/docs/concepts/security/service-accounts/
[5] GitHub. (n.d.). *ingress-nginx/docs/deploy/baremetal.md*. Retrieved from https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/baremetal.md
[6] Velero. (n.d.). *Velero*. Retrieved from https://velero.io/
