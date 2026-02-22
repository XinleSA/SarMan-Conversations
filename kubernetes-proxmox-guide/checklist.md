# Kubernetes on Proxmox: Project Tracker Checklist

This checklist provides a detailed, step-by-step project tracker for the Kubernetes deployment.

## Phase 1: Environment Setup and Prerequisites

- [ ] **Proxmox VE Preparation**
  - [ ] Verify Proxmox VE hosts are up-to-date.
  - [ ] Confirm Proxmox cluster is healthy and operational.
- [ ] **VM Creation**
  - [ ] Create VM for `k8s-control-plane` on Proxmox host 1.
  - [ ] Create VM for `k8s-worker-1` on Proxmox host 2.
  - [ ] Create VM for `k8s-worker-2` on Proxmox host 3.
  - [ ] All VMs created with Ubuntu 24.04 LTS.
- [ ] **Ansible-based VM Configuration**
  - [ ] Set up Ansible control node.
  - [ ] Create `inventory.ini` with correct host IPs.
  - [ ] Configure SSH key-based access from Ansible control node to all K8s VMs.
  - [ ] Run `setup_nodes.yml` playbook successfully.

## Phase 2: Kubernetes Cluster Installation

- [ ] **Control Plane Initialization**
  - [ ] SSH into `k8s-control-plane`.
  - [ ] Run `kubeadm init` with the correct pod network CIDR.
  - [ ] Save the `kubeadm join` command securely.
  - [ ] Configure `kubectl` for the `ubuntu` user.
- [ ] **CNI Plugin Installation**
  - [ ] Apply Calico manifests to the cluster.
  - [ ] Verify Calico pods are running in the `kube-system` namespace.
- [ ] **Worker Node Joining**
  - [ ] SSH into `k8s-worker-1` and run the `kubeadm join` command.
  - [ ] SSH into `k8s-worker-2` and run the `kubeadm join` command.
- [ ] **Cluster Verification**
  - [ ] Run `kubectl get nodes` on the control plane.
  - [ ] Confirm all three nodes are listed and have the `Ready` status.

## Phase 3: Storage, Networking, and Ingress

- [ ] **Persistent Storage (Longhorn)**
  - [ ] Apply Longhorn manifests.
  - [ ] Verify Longhorn pods are running in the `longhorn-system` namespace.
  - [ ] Access the Longhorn UI and confirm all nodes are available for storage.
- [ ] **Load Balancer (MetalLB)**
  - [ ] Apply MetalLB manifests.
  - [ ] Create and apply `metallb-config.yml` with the correct IP address range for your network.
- [ ] **Ingress (NGINX)**
  - [ ] Apply NGINX Ingress Controller manifests for bare-metal.
  - [ ] Verify the `ingress-nginx-controller` service gets an external IP from MetalLB.
- [ ] **Certificate Management (cert-manager)**
  - [ ] Apply cert-manager manifests.
  - [ ] Create and apply `letsencrypt-issuer.yml` with your email address.
  - [ ] Verify the `ClusterIssuer` is created successfully.

## Phase 4: GitOps with ArgoCD

- [ ] **ArgoCD Installation**
  - [ ] Create the `argocd` namespace.
  - [ ] Apply ArgoCD installation manifests.
- [ ] **ArgoCD Access**
  - [ ] Patch the `argocd-server` service to type `LoadBalancer`.
  - [ ] Retrieve the initial admin password.
  - [ ] Log in to the ArgoCD UI successfully.
- [ ] **GitOps Repository Setup**
  - [ ] Create a new Git repository for Kubernetes manifests.
  - [ ] Structure the repository with `base` and `overlays` directories.
- [ ] **ArgoCD Application Creation**
  - [ ] Create and apply the `argocd-app.yml` manifest, pointing to your GitOps repository.
  - [ ] Verify the application syncs successfully in the ArgoCD UI.

## Phase 5: n8n Integration for Automation

- [ ] **Secure n8n Integration**
  - [ ] Create the `n8n-automation` namespace.
  - [ ] Create the `n8n-sa` ServiceAccount.
  - [ ] Create and apply the `n8n-rbac.yml` manifest for the Role and RoleBinding.
  - [ ] Configure an n8n workflow to generate short-lived tokens using `kubectl create token`.
- [ ] **Promotion Gating Workflow**
  - [ ] Set up the GitHub Action to trigger the n8n webhook.
  - [ ] Import and configure the `promotion_gating_workflow.json` in n8n.
  - [ ] Test the promotion gate with a sample pull request.
- [ ] **Telegram Bot for Cluster Management**
  - [ ] Create a new Telegram bot and get the API token.
  - [ ] Import and configure the `telegram_bot_workflow.json` in n8n.
  - [ ] Test the bot by sending a `/status` command.

## Phase 6: Monitoring, Backup, and Disaster Recovery

- [ ] **Monitoring (Prometheus & Grafana)**
  - [ ] Add the `prometheus-community` Helm repo.
  - [ ] Install the `kube-prometheus-stack`.
  - [ ] Access the Grafana and Prometheus UIs.
- [ ] **Backup and Recovery (Velero)**
  - [ ] Set up an S3-compatible object storage bucket for backups.
  - [ ] Install the Velero CLI.
  - [ ] Install Velero on the cluster with the correct provider configuration.
  - [ ] Perform an initial test backup and restore.
