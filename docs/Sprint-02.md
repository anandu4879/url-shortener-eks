# Sprint 2 — EKS Cluster via Terraform

## Overview

Sprint 2 marks the transition from Kubernetes theory to a real Amazon EKS cluster provisioned with Terraform.

### Objectives

- Provision an Amazon EKS control plane
- Create an EKS Managed Node Group
- Configure IAM roles for the cluster and worker nodes
- Enable IAM Roles for Service Accounts (IRSA)
- Encrypt Kubernetes Secrets using AWS KMS
- Connect `kubectl` to the cluster
- Understand the responsibility split between AWS and Kubernetes administrators

---

# Architecture

```text
                    Internet
                        │
                        ▼
                kubectl / AWS CLI
                        │
                        ▼
             Amazon EKS Control Plane
        (Managed by AWS: API Server,
      Scheduler, Controller Manager, etcd)
                        │
            ┌───────────┴────────────┐
            ▼                        ▼
      Managed Node Group       IAM OIDC Provider
            │                        │
            ▼                        ▼
        EC2 Worker Nodes          IRSA
            │
            ▼
     Kubernetes Pods
```

---

# Why Amazon EKS?

Running Kubernetes yourself means managing:

- API Server
- etcd
- Scheduler
- Controller Manager
- High Availability
- Upgrades
- Backup and recovery

Amazon EKS manages the control plane so you can focus on workloads.

## Responsibility Split

| AWS Manages | You Manage |
|-------------|------------|
| API Server | Worker Nodes |
| etcd | Applications |
| Scheduler | Kubernetes Resources |
| Control Plane HA | Networking Add-ons |
| Control Plane Upgrades | Monitoring |

---

# Why Managed Node Groups?

Managed Node Groups provide:

- Automated lifecycle management
- Easier upgrades
- Native AWS integration
- Reduced operational overhead

This project intentionally uses **Managed Node Groups** instead of Fargate because understanding nodes is important for DevOps engineering.

---

# IRSA (IAM Roles for Service Accounts)

Traditional EC2 instance profiles grant permissions to every Pod on a node.

IRSA assigns IAM permissions to individual Kubernetes Service Accounts.

Benefits:

- Least privilege
- Better security
- Fine-grained AWS access
- No credential sharing between workloads

Conceptually:

```text
GitHub OIDC
      │
      ▼
GitHub Actions Role

EKS OIDC
      │
      ▼
Service Account
      │
      ▼
Application Pod
```

---

# Terraform Module Layout

```text
terraform/
├── modules/
│   ├── vpc/
│   └── eks/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    └── dev/
        └── main.tf
```

The existing VPC module is reused while a new EKS module provisions:

- EKS Cluster
- IAM Roles
- Managed Node Group
- OIDC Provider
- KMS Key

---

# Cluster IAM Role vs Node IAM Role

| Cluster Role | Node Role |
|--------------|-----------|
| Used by EKS Control Plane | Used by EC2 Worker Nodes |
| Creates AWS networking resources | Pulls container images |
| Manages cluster resources | Runs Pods |

---

# Secrets Encryption

Kubernetes Secrets are Base64 encoded, not encrypted.

This sprint enables:

- AWS KMS
- EKS Secret Encryption

so Secret objects are encrypted at rest.

---

# Public vs Private API Endpoint

Current learning configuration:

- Private Endpoint: Enabled
- Public Endpoint: Enabled

Reason:

Allows `kubectl` from a local machine while still supporting private VPC access.

Production environments often disable the public endpoint and use VPN or bastion hosts.

---

# Apply the Infrastructure

```bash
cd terraform/environments/dev

terraform init
terraform validate
terraform plan
terraform apply
```

Configure kubectl:

```bash
aws eks update-kubeconfig \
  --name url-shortener-dev-cluster \
  --region ap-south-1

kubectl get nodes
```

Successful output confirms the worker nodes have joined the cluster.

---

# EC2 Project vs EKS Project

| Previous Project | Sprint 2 |
|------------------|----------|
| EC2 Instances | EKS Worker Nodes |
| ASG | Managed Node Group |
| Instance Profile | IRSA |
| Docker | Kubernetes |
| User Data | Kubernetes Workloads |

---

# Key Learnings

- Understand what AWS manages in EKS
- Understand worker node responsibilities
- Learn IRSA and OIDC
- Encrypt Secrets using KMS
- Provision EKS using Terraform
- Connect kubectl to the cluster

---

# Interview Questions

1. What does AWS manage in EKS?
2. What is IRSA?
3. Why are separate IAM roles used for the cluster and nodes?
4. Why is KMS used with Kubernetes Secrets?
5. Why use Managed Node Groups?
6. Why might you disable the public API endpoint?
7. How does IRSA improve over EC2 instance profiles?

---

# Screenshots Checklist

- EKS Cluster Overview
- Managed Node Group
- IAM Cluster Role
- IAM Node Role
- KMS Key
- OIDC Provider
- `kubectl get nodes`

---

# Cost

Approximate hourly cost:

| Resource | Cost |
|----------|------|
| EKS Control Plane | ~$0.10/hr |
| 2 × t3.medium | ~$0.08/hr |
| **Total** | **~$0.18/hr** |

Destroy the infrastructure after each learning session.

---

# Cleanup

```bash
cd terraform/environments/dev
terraform destroy
```

---



# Sprint Summary

Sprint 2 transformed the project from Kubernetes concepts into a working Amazon EKS platform.

The key achievements were:

- Provisioned a managed Kubernetes control plane
- Added worker nodes using Managed Node Groups
- Configured IRSA
- Enabled KMS encryption for Secrets
- Connected kubectl to the cluster
- Established the foundation for deploying applications in Sprint 3.
