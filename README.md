# SarMan-Conversations

A repository storing SarMan conversations.

## Kubernetes Proxmox Guide

This repository now includes a comprehensive guide and a collection of automation scripts for deploying a Kubernetes cluster on Proxmox VE. The guide is intended to provide a step-by-step process for setting up a production-ready environment.

### Contents:

- **`k8s_proxmox_guide.md`**: The main deployment guide.
- **`checklist.md`**: A project tracker checklist to help you follow the deployment progress.
- **`automation_scripts/`**: A folder containing various scripts to automate the setup process:
    - **`ansible/`**: Ansible playbooks for node setup.
    - **`kubernetes/`**: Kubernetes manifests for various components like MetalLB, LetsEncrypt, and ArgoCD.
    - **`n8n/`**: n8n workflows for promotion gating and Telegram bot integration.
