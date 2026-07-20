# Sprint 0 — Planning: From EC2/ASG to Kubernetes on Amazon EKS

## Overview

This project is the next phase of my DevOps learning journey.

In my previous project, I built and deployed a URL Shortener application on AWS using:

* Terraform
* EC2
* Auto Scaling Groups
* Application Load Balancer
* Amazon RDS PostgreSQL
* Amazon ElastiCache for Redis
* Amazon ECR
* GitHub Actions
* GitHub OIDC
* Prometheus
* Grafana

That project taught me how to build infrastructure from scratch and understand how individual AWS components work together.

This project takes the same application and evolves the infrastructure from an **EC2 + Auto Scaling Group architecture** into a **Kubernetes-native architecture running on Amazon EKS**.

The goal is not simply to replace EC2 with Kubernetes.

The goal is to understand the operational model Kubernetes introduces and why companies adopt container orchestration platforms as their systems grow.

---

# 1. Why This Project?

My previous EC2/ASG architecture works.

However, every part of the orchestration layer had to be configured explicitly:

* Which EC2 instances should run
* How many instances should exist
* How unhealthy instances should be replaced
* How instances register with target groups
* How health checks work
* How application configuration reaches instances
* How new application versions are deployed

For a single application, this is manageable.

But imagine managing the same infrastructure pattern for 10, 50, or 100 services.

Each service could require its own:

* Auto Scaling Group
* Launch Template
* Target Group
* Load Balancer rules
* Health checks
* Scaling policies
* IAM permissions
* Deployment logic

The operational complexity grows quickly.

Kubernetes provides a standardized orchestration model where applications use the same primitives for:

* Deployment
* Scaling
* Networking
* Configuration
* Health checks
* Service discovery

Instead of manually wiring these systems together for every application, Kubernetes provides a common platform for managing containerized workloads.

---

# 2. The Core Conceptual Shift

The biggest change in this project is moving from a **one-shot infrastructure model** to a **continuous reconciliation model**.

## Terraform Model

Terraform follows an apply-based model.

```text
Desired Infrastructure
        │
        ▼
terraform plan
        │
        ▼
terraform apply
        │
        ▼
Infrastructure Created
        │
        ▼
Terraform Stops
```

Terraform calculates the difference between the desired infrastructure and the current infrastructure, applies the required changes, and then stops.

If someone manually changes or deletes infrastructure afterward, Terraform does not automatically repair it.

The drift is detected the next time Terraform runs.

---

## Kubernetes Model

Kubernetes works differently.

```text
Desired State
replicas: 2
     │
     ▼
Kubernetes Controller
     │
     ▼
Compare Desired State
with Actual State
     │
     ▼
Reconcile Differences
     │
     └──────────────┐
                    │
                    ▼
              Repeat Forever
```

For example, if I declare:

```yaml
replicas: 2
```

Kubernetes continuously checks whether two replicas actually exist.

If one Pod crashes:

```text
Desired: 2 Pods
Actual:  1 Pod
```

The Kubernetes controller detects the difference and automatically creates another Pod.

The system continuously works toward the desired state.

This is known as the **reconciliation loop**.

---

## Terraform vs Kubernetes

| Terraform                                   | Kubernetes                                              |
| ------------------------------------------- | ------------------------------------------------------- |
| Runs when `terraform apply` is executed     | Controllers run continuously                            |
| Calculates infrastructure differences       | Continuously compares desired and actual state          |
| Stops after applying changes                | Reconciliation runs indefinitely                        |
| Drift is detected on the next Terraform run | Drift is automatically corrected                        |
| Primarily manages infrastructure resources  | Primarily orchestrates workloads and platform resources |

A useful mental model is:

> Terraform creates the infrastructure that Kubernetes runs on. Kubernetes continuously manages the workloads running inside that infrastructure.

---

# 3. Mapping My Existing Knowledge to Kubernetes

One goal of this project is to connect Kubernetes concepts to infrastructure concepts I already learned while building the EC2 version of the project.

| EC2 / AWS Concept                    | Kubernetes Equivalent     | Key Difference                                                          |
| ------------------------------------ | ------------------------- | ----------------------------------------------------------------------- |
| Launch Template + Auto Scaling Group | Deployment                | Kubernetes continuously reconciles the desired number of Pods           |
| Target Group                         | Service                   | Provides stable networking and service discovery                        |
| ALB Listener Rules                   | Ingress                   | Provides Layer 7 routing into Kubernetes services                       |
| EC2 Instance Profile                 | IRSA                      | IAM permissions can be assigned to specific Kubernetes Service Accounts |
| User Data Environment Variables      | ConfigMaps / Secrets      | Application configuration becomes Kubernetes-native                     |
| Security Groups                      | Network Policies          | Controls workload-to-workload communication inside the cluster          |
| ASG Health Checks                    | Liveness Probes           | Determines whether a container should be restarted                      |
| ALB Health Checks                    | Readiness Probes          | Determines whether a Pod should receive traffic                         |
| ASG Scaling Policies                 | Horizontal Pod Autoscaler | Automatically adjusts Pod replicas                                      |
| EC2 Auto Scaling                     | Node Autoscaling          | Automatically adjusts cluster compute capacity                          |

The important part is not memorizing these mappings.

The goal is understanding **why Kubernetes introduces these abstractions and what operational problems they solve**.

---

# 4. Why Amazon EKS?

There are multiple ways to run containerized applications on AWS.

Some alternatives include:

### Amazon ECS / AWS Fargate

AWS-native container orchestration.

Advantages:

* Simpler operational model
* Deep AWS integration
* Less Kubernetes complexity

Tradeoff:

* Less portable outside the AWS ecosystem

### EC2 + Auto Scaling Groups

This is the architecture used in my previous project.

Advantages:

* Full infrastructure control
* Easier to understand individual AWS components
* Good for simpler workloads

Tradeoff:

* More orchestration logic must be built manually

### HashiCorp Nomad

A simpler workload orchestrator.

Tradeoff:

* Less commonly requested in DevOps job postings compared with Kubernetes

### Amazon EKS

Amazon's managed Kubernetes service.

I chose EKS because:

* Kubernetes is widely used across the industry
* It provides a deeper orchestration model to learn
* It is frequently mentioned in DevOps and Platform Engineering roles
* It integrates Kubernetes with AWS services
* It allows me to build on my existing AWS knowledge

This project deliberately goes deeper into Kubernetes than what is required for the AWS Solutions Architect Associate exam because the primary goal is practical DevOps and cloud engineering experience.

---

# 5. Tradeoffs of Kubernetes

Kubernetes is not automatically the correct solution for every application.

It introduces real complexity.

Some tradeoffs include:

* More infrastructure components
* A larger operational surface area
* More networking concepts
* More security considerations
* A new deployment model
* A new debugging model
* Additional monitoring requirements
* Control plane costs when using managed Kubernetes

Amazon EKS also introduces a fixed control plane cost regardless of how small the workload is.

For small applications, ECS, Fargate, or even EC2 may be simpler and more cost-effective.

Understanding **when not to use Kubernetes** is just as important as knowing how to use it.

---

# 6. Multi-Environment Architecture

Unlike my previous project, which primarily used a development environment, this project will implement three environments:

```text
Development
     │
     ▼
Staging
     │
     ▼
Production
```

Each environment has a specific purpose.

---

## Development

Optimized for fast iteration and lower cost.

Characteristics:

* Single application replica
* Smaller infrastructure
* Smaller database instance
* Single-AZ RDS
* No manual deployment approval

---

## Staging

Designed to validate production-like behavior before release.

Characteristics:

* Multiple application replicas
* Architecture matching production
* Smaller overall scale
* Production-like deployment configuration

The goal is to catch issues that may not appear in development.

---

## Production

Optimized for reliability and availability.

Characteristics:

* Higher replica count
* Multi-AZ RDS
* Stricter monitoring
* Stricter alerting thresholds
* Manual deployment approval
* Production promotion workflow

---

## Environment Design Principle

Staging should mirror the **architecture of production**, not necessarily its scale.

```text
Development
Small + Fast

      │

      ▼

Staging
Production Architecture
Smaller Scale

      │

      ▼

Production
Production Architecture
Full Scale
```

If staging and production have fundamentally different architectures, staging becomes a poor indicator of whether a production deployment will succeed.

---

# 7. What Will Be Reused?

This project builds on my previous URL Shortener project rather than starting everything from zero.

## Reused

The following components can largely remain unchanged:

* VPC Terraform module
* Public subnets
* Private subnets
* Internet Gateway
* NAT architecture
* Application code
* FastAPI backend
* PostgreSQL integration
* Redis integration
* Docker image
* ECR repository pattern

The application itself does not need to understand Kubernetes.

Kubernetes changes **how the container runs**, not the application's business logic.

---

# 8. What Will Be Rebuilt?

The compute and orchestration layer will change completely.

The previous architecture used:

```text
Launch Template
      │
      ▼
Auto Scaling Group
      │
      ▼
EC2 Instances
      │
      ▼
Target Group
      │
      ▼
Application Load Balancer
```

The new architecture will move toward:

```text
Amazon EKS
      │
      ▼
Managed Node Groups
      │
      ▼
Kubernetes Deployment
      │
      ▼
Pods
      │
      ▼
Kubernetes Service
      │
      ▼
Ingress
      │
      ▼
AWS Application Load Balancer
```

The following EC2-specific components will be removed from the application compute layer:

* EC2 Launch Templates
* Application Auto Scaling Groups
* User-data deployment scripts
* Manual target group registration logic

These responsibilities will move to Kubernetes and EKS.

---

# 9. What Will Be Adapted?

Some existing Terraform modules will remain but require environment-specific configuration.

### Amazon RDS

Parameters will vary by environment:

```text
dev
├── Smaller instance
├── Single-AZ
└── Lower cost

staging
├── Production-like configuration
└── Smaller scale

prod
├── Larger instance
├── Multi-AZ
└── Higher availability
```

### Amazon ElastiCache

Redis configuration will also be parameterized per environment.

The goal is to reuse infrastructure modules while allowing each environment to define its own capacity and availability requirements.

---

# 10. Planned Architecture

The target architecture for this project will evolve toward:

```text
                        Internet
                            │
                            ▼
                    Application Load
                        Balancer
                            │
                            ▼
                      EKS Ingress
                            │
                            ▼
                    Kubernetes Service
                            │
                            ▼
                    Kubernetes Deployment
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
               Pod 1                 Pod 2
                 │                     │
                 └──────────┬──────────┘
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
          Amazon RDS                ElastiCache
          PostgreSQL                   Redis
```

The infrastructure will be provisioned using Terraform.

Application workloads will be deployed using Kubernetes manifests and later managed across environments using Kustomize or Helm.

---



# 11. Key Learning from Sprint 0

The biggest lesson from this sprint is that Kubernetes is not simply another way to run Docker containers.

The fundamental shift is the move toward a **continuous reconciliation model**.

Instead of manually defining and connecting every infrastructure component required to run an application, Kubernetes provides standardized abstractions for:

* Deployments
* Scaling
* Networking
* Configuration
* Secrets
* Health checks
* Service discovery
* Workload identity

My previous EC2/ASG project helped me understand the individual infrastructure components underneath these abstractions.

This project will help me understand why orchestration platforms exist and how they manage those concerns at scale.

---

# 13. Interview Preparation

Questions I should be able to answer after Sprint 0:

### 1. What is the fundamental difference between Terraform's apply model and Kubernetes's reconciliation model?

Terraform calculates and applies infrastructure changes when `terraform apply` is executed.

Kubernetes controllers continuously monitor the actual state of the system and reconcile it with the declared desired state.

---

### 2. When would you recommend against Kubernetes?

Kubernetes may be unnecessary when:

* The application is small
* There are only a few services
* The team lacks Kubernetes operational experience
* Simpler services like ECS or Fargate can meet the requirements
* The operational complexity outweighs the benefits

---

### 3. What cost consideration does EKS introduce?

Amazon EKS charges for the Kubernetes control plane independently of the compute resources used by workloads.

This means even a small cluster has a baseline cost before considering worker nodes, load balancers, NAT Gateways, databases, and other services.

---

### 4. Why should staging mirror production?

Staging should reproduce the architecture and deployment behavior of production so that problems can be discovered before production deployment.

The capacity can be smaller, but the architecture should remain similar.

---

### 5. How does IRSA improve on EC2 instance profiles?

With an EC2 instance profile, workloads running on the same EC2 instance may share the permissions assigned to that node.

IRSA allows AWS IAM roles to be associated with Kubernetes Service Accounts, enabling AWS permissions to be scoped more precisely to specific workloads.

---




## Next: Sprint 1 — Kubernetes Fundamentals

Before provisioning an EKS cluster, the next sprint will focus on understanding Kubernetes fundamentals locally and conceptually.

The goal is to understand the Kubernetes objects and control-plane behavior before introducing the additional complexity and cost of AWS EKS.
