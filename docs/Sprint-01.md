# Sprint 1 — Kubernetes Fundamentals

## Overview

Sprint 1 focuses entirely on **Kubernetes fundamentals**.

No EKS cluster is created yet.

No AWS infrastructure is provisioned.

No Kubernetes YAML manifests are deployed.

This is intentional.

Before provisioning Amazon EKS or writing Kubernetes manifests, I wanted to understand the fundamental objects that Kubernetes uses to run and manage applications.

The main concepts covered in this sprint are:

* Pods
* Deployments
* ReplicaSets
* Services
* ConfigMaps
* Secrets
* Namespaces
* StatefulSets
* DaemonSets
* Kubernetes reconciliation

Throughout this sprint, I mapped each Kubernetes concept back to the EC2/Auto Scaling architecture from my previous project.

---

# 1. Why Learn the Concepts Before Creating EKS?

It would be easy to immediately create an EKS cluster, copy some Kubernetes YAML, run:

```bash
kubectl apply -f deployment.yaml
```

and see Pods running.

But that would teach me syntax without teaching me **why Kubernetes works the way it does**.

Every Kubernetes object I create later should answer a specific operational problem.

For example:

```text
Problem                           Kubernetes Solution
──────────────────────────────────────────────────────

Run application containers       Pod

Keep desired replicas running    Deployment

Stable networking to Pods        Service

Application configuration        ConfigMap

Sensitive configuration          Secret

Logical workload isolation       Namespace

External HTTP/HTTPS routing      Ingress
```

Understanding these abstractions first makes the YAML configuration much easier to reason about later.

---

# 2. Pod — The Fundamental Kubernetes Unit

## What Problem Does a Pod Solve?

In Docker, the smallest runnable unit is normally a container.

Kubernetes introduces another abstraction above containers:

> The Pod.

A Pod represents **one or more containers that must be scheduled together and share certain resources**.

Containers inside the same Pod can share:

* Network namespace
* IP address
* `localhost`
* Storage volumes
* Lifecycle

The Pod becomes the smallest unit Kubernetes schedules onto a worker node.

---

## Pod Architecture

```text
Kubernetes Cluster
        │
        ▼
     Worker Node
        │
        ▼
       Pod
   ┌───────────────┐
   │               │
   │  App Container│
   │               │
   │      +        │
   │               │
   │ Sidecar       │
   │ Container     │
   │               │
   └───────────────┘

Containers inside the Pod:

✓ Share networking
✓ Can communicate through localhost
✓ Can share volumes
✓ Are scheduled together
```

Most Pods contain only **one primary application container**.

However, Kubernetes allows multiple tightly coupled containers to run together.

Examples include:

* Application + proxy sidecar
* Application + log shipper
* Application + service-mesh proxy

---

# 3. Container vs Pod

A common Kubernetes interview question is:

> What is the difference between a container and a Pod?

The simplest answer:

**A container is the runtime unit that executes an application.**

**A Pod is Kubernetes's scheduling abstraction containing one or more containers that share networking and potentially storage.**

```text
Container
    │
    ▼
Application Process

Pod
    │
    ├── Container
    │
    └── Optional Sidecar Container
```

Kubernetes schedules **Pods**, not individual containers.

---

# 4. Pods Are Disposable

One of the most important Kubernetes concepts is:

> Pods are ephemeral and disposable.

A Pod should not be treated like a specific server that must remain alive forever.

If a Pod disappears, Kubernetes may create another one.

The replacement Pod can have:

* A different Pod name
* A different IP address
* A different node
* A completely new container

This is normal Kubernetes behavior.

---

## EC2 Mental Model

In my previous architecture:

```text
EC2 Instance
     │
     ▼
Docker Container
```

The container's lifecycle was strongly connected to the EC2 instance.

If the EC2 instance failed:

```text
EC2 Failure
     │
     ▼
ASG Detects Failure
     │
     ▼
New EC2 Instance
     │
     ▼
User Data Executes
     │
     ▼
Docker Container Starts
```

Kubernetes moves application lifecycle management to a more granular level.

```text
Pod Failure
     │
     ▼
Deployment Detects Missing Replica
     │
     ▼
New Pod Created
     │
     ▼
Scheduler Places Pod
     │
     ▼
Container Starts
```

The application no longer depends on the identity of a particular machine.

---

# 5. Why We Usually Don't Create Pods Directly

Creating a standalone Pod is possible.

But normally we don't do this for production applications.

Why?

Because if the Pod disappears, nothing necessarily recreates that standalone Pod.

What we really want to declare is:

```text
"I always want 3 copies
of this application running."
```

That requires a controller.

For stateless applications, the most common controller is a:

# Deployment

---

# 6. Deployment — Managing Pods Declaratively

A Deployment manages application Pods for us.

Instead of manually creating individual Pods, we describe the desired state.

For example:

```yaml
replicas: 3
```

This means:

> Kubernetes should maintain three replicas of this application.

---

## Reconciliation

Suppose Kubernetes currently has:

```text
Desired Replicas: 3

Actual Replicas: 3

Status: Healthy
```

Then one Pod crashes.

```text
Desired Replicas: 3

Actual Replicas: 2
```

The Deployment controller notices the difference.

```text
Deployment Controller
        │
        ▼
Desired = 3
Actual  = 2
        │
        ▼
Difference = 1
        │
        ▼
Create New Pod
```

Eventually:

```text
Desired Replicas: 3

Actual Replicas: 3
```

The system has reconciled itself.

This is the reconciliation model introduced during Sprint 0.

---

# 7. Deployment → ReplicaSet → Pods

A Deployment does not directly manage Pods.

Internally:

```text
Deployment
     │
     ▼
ReplicaSet
     │
     ├── Pod
     ├── Pod
     └── Pod
```

The Deployment manages ReplicaSets.

The ReplicaSet maintains the required number of Pods.

Normally, I interact with the **Deployment**, not the ReplicaSet directly.

---

# 8. Why Use Deployment for My FastAPI Application?

My URL Shortener API is designed to be stateless.

Any application instance can process a request because persistent data lives outside the application container:

```text
FastAPI Pods
     │
     ├──────────► PostgreSQL / RDS
     │
     └──────────► Redis / ElastiCache
```

The Pods themselves do not need permanent identity.

Therefore:

```text
Stateless FastAPI API
        │
        ▼
    Deployment
```

is the correct Kubernetes controller.

---

# 9. Deployment vs My Previous Auto Scaling Group

The Deployment is conceptually similar to the combination of:

```text
Launch Template
      +
Auto Scaling Group
```

from my previous AWS architecture.

| Previous Architecture        | Kubernetes                |
| ---------------------------- | ------------------------- |
| Launch Template              | Pod Template              |
| Auto Scaling Group           | Deployment / ReplicaSet   |
| Desired EC2 Capacity         | Desired Pod Replicas      |
| Replace Failed EC2           | Replace Failed Pod        |
| Instance Rolling Replacement | Deployment Rolling Update |

One major improvement is application deployment behavior.

In my previous project, changing EC2 Launch Template user data did not automatically guarantee that existing instances were replaced.

I encountered this directly and had to deal with settings such as:

```text
user_data_replace_on_change
```

With Kubernetes Deployments, updating the container image automatically triggers a **rolling update**.

For example:

```text
v1 Pods

● ● ●

   ↓ Deployment Update

● ● ○

   ↓

● ○ ○

   ↓

○ ○ ○

v2 Pods
```

The application version can be gradually replaced without manually rebuilding every server.

---

# 10. Other Kubernetes Controllers

Deployment is not the only workload controller.

Different workloads require different controllers.

---

## ReplicaSet

Maintains a specific number of identical Pods.

```text
ReplicaSet
   │
   ├── Pod
   ├── Pod
   └── Pod
```

Normally managed indirectly through a Deployment.

---

## StatefulSet

Designed for workloads requiring stable identity or persistent storage.

Typical examples:

* Database clusters
* Kafka
* Stateful distributed systems

```text
StatefulSet

postgres-0
postgres-1
postgres-2
```

Unlike Deployment Pods, StatefulSet Pods maintain predictable identities.

---

## DaemonSet

Ensures that a Pod runs on every applicable node.

```text
Node 1 → Monitoring Pod
Node 2 → Monitoring Pod
Node 3 → Monitoring Pod
```

Common examples:

* Node Exporter
* Log collection agents
* Security agents
* Monitoring agents

This will become relevant later when implementing Kubernetes observability.

---

# 11. Deployment vs StatefulSet

A useful rule:

```text
Stateless Application
        │
        ▼
    Deployment
```

Examples:

* FastAPI
* Node.js API
* Java backend
* Web frontend

---

```text
Stateful Application
        │
        ▼
    StatefulSet
```

Examples:

* Databases
* Distributed storage systems
* Applications requiring stable Pod identities

For this project:

```text
FastAPI
   │
   ▼
Deployment

PostgreSQL
   │
   ▼
Amazon RDS
```

I will deliberately **not run PostgreSQL inside Kubernetes**.

Amazon RDS remains responsible for database durability, backups, availability, and database lifecycle management.

---

# 12. The Pod IP Problem

Pods are disposable.

This creates a networking problem.

Suppose the Deployment contains:

```text
Pod A → 10.0.1.10

Pod B → 10.0.1.11

Pod C → 10.0.1.12
```

Pod B crashes.

Its replacement might receive:

```text
Pod D → 10.0.1.25
```

Any system using the old IP:

```text
10.0.1.11
```

would now be pointing to something that no longer exists.

Therefore:

> Applications should not depend directly on individual Pod IP addresses.

Kubernetes solves this using **Services**.

---

# 13. Service — Stable Networking for Unstable Pods

A Kubernetes Service provides a stable network endpoint for a group of Pods.

```text
               Service
                  │
          ┌───────┼───────┐
          ▼       ▼       ▼
        Pod A   Pod B   Pod C
```

The Service discovers Pods using **labels and selectors**.

Even when Pods disappear and are recreated, the Service automatically routes traffic to the currently available matching Pods.

---

# 14. Labels and Selectors

Suppose application Pods contain the label:

```yaml
app: url-shortener
```

The Service can use:

```yaml
selector:
  app: url-shortener
```

Conceptually:

```text
Service

selector:
app = url-shortener

        │
        ▼

Find matching Pods

        │
        ├── Pod A
        │   app=url-shortener
        │
        ├── Pod B
        │   app=url-shortener
        │
        └── Pod C
            app=url-shortener
```

This means the Service does not need to know individual Pod IP addresses.

It simply tracks Pods matching its selector.

---

# 15. Service vs Target Group

This maps closely to my previous AWS architecture.

Previously:

```text
ALB
 │
 ▼
Target Group
 │
 ├── EC2 Instance
 ├── EC2 Instance
 └── EC2 Instance
```

With Kubernetes:

```text
Ingress / ALB
      │
      ▼
   Service
      │
      ├── Pod
      ├── Pod
      └── Pod
```

The Kubernetes Service performs the stable workload discovery role that my Target Group helped provide in the EC2 architecture.

The major difference is that Kubernetes handles Pod membership dynamically through labels and selectors.

---

# 16. Kubernetes Service Types

There are several Service types.

## ClusterIP

```text
Cluster Internal Traffic
        │
        ▼
     ClusterIP
        │
        ▼
       Pods
```

Provides internal networking inside the cluster.

This will be the main Service type for the application.

---

## NodePort

```text
External Traffic
      │
      ▼
WorkerNode:Port
      │
      ▼
   Service
      │
      ▼
     Pods
```

Exposes the Service through a port on every worker node.

Useful in some scenarios but usually not the preferred production exposure method for this project.

---

## LoadBalancer

```text
Internet
    │
    ▼
AWS Load Balancer
    │
    ▼
LoadBalancer Service
    │
    ▼
Pods
```

On cloud platforms, this can automatically provision an external load balancer.

Simple, but potentially inefficient when many applications each need external access.

---

# 17. Why ClusterIP + Ingress?

For this project, the planned architecture is:

```text
Internet
    │
    ▼
AWS Application Load Balancer
    │
    ▼
Ingress
    │
    ▼
ClusterIP Service
    │
    ▼
FastAPI Pods
```

Instead of creating a separate cloud load balancer for every Service, Ingress allows one external routing layer to route traffic to multiple internal Services.

For example:

```text
ALB
 │
 ├── /api       → API Service
 │
 ├── /analytics → Analytics Service
 │
 └── /admin     → Admin Service
```

This is conceptually similar to the ALB listener-rule architecture from my previous project.

---

# 18. Service vs Ingress

Another important interview distinction:

## Service

Provides stable networking to Pods.

```text
Service
   │
   ▼
Pods
```

Usually responsible for service discovery and internal load balancing.

---

## Ingress

Provides Layer 7 HTTP/HTTPS routing.

```text
Internet
   │
   ▼
Ingress
   │
   ▼
Service
   │
   ▼
Pods
```

On EKS, an Ingress can work with the **AWS Load Balancer Controller** to provision and configure an Application Load Balancer.

---

# 19. ConfigMaps — Non-Sensitive Configuration

Applications require configuration.

Examples:

```text
APP_ENV
LOG_LEVEL
REDIS_HOST
DATABASE_HOST
```

In my EC2 project, configuration was passed through mechanisms such as:

```text
Terraform
   │
   ▼
templatefile()
   │
   ▼
user-data
   │
   ▼
Environment Variables
   │
   ▼
Docker Container
```

Kubernetes provides a native object for non-sensitive configuration:

> ConfigMap

---

## ConfigMap Flow

```text
ConfigMap
    │
    ▼
Pod Environment Variables
    │
    ▼
Application
```

A ConfigMap can also be mounted as files inside containers.

This separates application configuration from the container image.

The same Docker image can therefore run in:

```text
dev
staging
prod
```

with different configuration.

---

# 20. Kubernetes Secrets

Sensitive configuration should not be placed in ConfigMaps.

Examples:

```text
DATABASE_PASSWORD
API_KEYS
TOKENS
CREDENTIALS
```

Kubernetes provides:

> Secret

Conceptually:

```text
Kubernetes Secret
       │
       ▼
Pod Environment Variable
       │
       ▼
Application
```

However, there is an important security detail.

---

# 21. Are Kubernetes Secrets Secure by Default?

Not completely.

Kubernetes Secret values are commonly represented using Base64 encoding.

Base64 is:

```text
Encoding
```

not:

```text
Encryption
```

Anyone with sufficient permission to retrieve the Secret can decode the value.

Therefore, production Kubernetes security requires additional controls.

---

# 22. EKS Secret Encryption

For this project, EKS Secret encryption will be considered as part of the cluster security design.

Conceptually:

```text
Kubernetes Secret
        │
        ▼
      etcd
        │
        ▼
Encryption at Rest
        │
        ▼
     AWS KMS
```

This protects Kubernetes Secret data stored at rest.

Security also depends on:

* Kubernetes RBAC
* IAM permissions
* Least privilege
* KMS permissions
* Proper Secret access controls

---

# 23. External Secrets Operator

A more advanced production architecture could use an external secret store.

For example:

```text
AWS Secrets Manager
        │
        ▼
External Secrets Operator
        │
        ▼
Kubernetes Secret
        │
        ▼
Application Pod
```

This avoids manually managing secret values directly inside Kubernetes manifests.

I will initially learn native Kubernetes Secrets first.

The External Secrets Operator can then be documented as a future security improvement.

---

# 24. ConfigMap vs Secret

| ConfigMap                   | Secret                  |
| --------------------------- | ----------------------- |
| Non-sensitive configuration | Sensitive configuration |
| Environment settings        | Passwords               |
| Feature flags               | API keys                |
| Hostnames                   | Tokens                  |
| Logging configuration       | Credentials             |

Simple rule:

```text
Is the value sensitive?

NO
 │
 ▼
ConfigMap

YES
 │
 ▼
Secret
```

---

# 25. Namespaces

As Kubernetes clusters grow, workloads need logical organization and isolation.

Kubernetes provides:

> Namespaces

A cluster can contain:

```text
Kubernetes Cluster
       │
       ├── dev namespace
       │
       ├── staging namespace
       │
       └── prod namespace
```

Namespaces can help scope:

* Workloads
* Services
* RBAC
* Network Policies
* Resource quotas
* Configuration

---

# 26. Namespaces and Multi-Environment Architecture

Later in this project, I need to make an important architectural decision.

There are two major approaches.

---

## Option 1 — Separate EKS Clusters

```text
Dev EKS Cluster

Staging EKS Cluster

Production EKS Cluster
```

### Advantages

* Strong isolation
* Smaller blast radius
* Independent cluster lifecycle
* Production failures less likely to affect development

### Disadvantages

* Higher cost
* More clusters to maintain
* More upgrades
* More operational overhead
* Separate EKS control plane costs

---

## Option 2 — Shared Cluster with Namespaces

```text
             EKS Cluster
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
       dev     staging     prod
```

### Advantages

* Lower cost
* Fewer clusters to operate
* Easier for smaller teams
* Shared infrastructure

### Disadvantages

* Larger potential blast radius
* RBAC mistakes can become more serious
* Network isolation must be carefully configured
* Cluster-level problems can affect all environments

The final decision will be made explicitly during the multi-environment architecture sprint rather than choosing one automatically.

---

# 27. Kubernetes Concepts Mapped to My EC2 Architecture

This is the most useful mental mapping from Sprint 1.

| Kubernetes  | Previous EC2 Architecture           | Purpose                                |
| ----------- | ----------------------------------- | -------------------------------------- |
| Pod         | Docker container running on EC2     | Runs application workload              |
| Deployment  | Launch Template + ASG               | Maintains desired application replicas |
| ReplicaSet  | ASG capacity management             | Ensures replica count                  |
| Service     | Target Group / service endpoint     | Stable routing to workloads            |
| Ingress     | ALB Listener Rules                  | HTTP/HTTPS routing                     |
| ConfigMap   | User-data environment variables     | Non-sensitive configuration            |
| Secret      | Parameter Store / secret injection  | Sensitive configuration                |
| Namespace   | Logical environment boundary        | Workload organization/isolation        |
| StatefulSet | Stateful instance/workload model    | Stable workload identity               |
| DaemonSet   | Agent running on every EC2 instance | One Pod per node                       |

This mapping helps connect Kubernetes concepts to infrastructure I already understand.

---

# 28. What Happens When a Kubernetes Node Dies?

This is an important reconciliation example.

Imagine:

```text
Deployment
replicas = 3

      │
      ├── Pod A → Node 1
      ├── Pod B → Node 2
      └── Pod C → Node 3
```

Now:

```text
Node 2 Dies
```

Pod B becomes unavailable.

The system eventually detects the failure.

```text
Desired Pods = 3

Available Pods = 2
```

Kubernetes controllers work to restore the desired state.

```text
Missing Replica Detected
        │
        ▼
New Pod Created
        │
        ▼
Scheduler Finds Available Node
        │
        ▼
Container Starts
        │
        ▼
Service Routes Traffic
```

Eventually:

```text
Deployment
replicas = 3

      │
      ├── Pod A
      ├── Pod C
      └── Pod D
```

The identity of the original Pod is irrelevant.

What matters is that the **desired state remains satisfied**.

---

# 29. One Major Improvement Over My EC2 Architecture

One of the biggest lessons from this sprint is how Kubernetes standardizes functionality that previously required several AWS resources and configuration layers.

Previously:

```text
Terraform
    │
    ▼
Launch Template
    │
    ▼
Auto Scaling Group
    │
    ▼
EC2
    │
    ▼
User Data
    │
    ▼
Docker
    │
    ▼
Target Group
    │
    ▼
ALB
```

With Kubernetes:

```text
Deployment
    │
    ▼
Pods
    │
    ▼
Service
    │
    ▼
Ingress
```

The underlying infrastructure still exists.

Kubernetes does not magically eliminate infrastructure.

Instead, it provides a consistent orchestration layer on top of it.

---

# 30. Key Learning Summary

Sprint 1 established the Kubernetes concepts required before provisioning EKS.

### Pod

The disposable atomic scheduling unit containing one or more containers.

### Deployment

Continuously maintains the desired state of stateless application Pods.

### ReplicaSet

Maintains the required number of Pod replicas and is normally managed through a Deployment.

### Service

Provides stable networking over constantly changing Pod IP addresses.

### Ingress

Provides Layer 7 HTTP/HTTPS routing to Services.

### ConfigMap

Stores non-sensitive application configuration.

### Secret

Stores sensitive configuration, while requiring additional security controls for proper production usage.

### Namespace

Provides logical organization and isolation within a Kubernetes cluster.

### StatefulSet

Used when workloads require stable identities or persistent storage relationships.

### DaemonSet

Runs a Pod on each applicable cluster node.

---

# 31. Interview Preparation

After Sprint 1, I should be able to answer the following questions.

### 1. What is the difference between a container and a Pod?

A container executes an application process.

A Pod is Kubernetes's smallest scheduling unit and contains one or more containers sharing networking and potentially storage.

---

### 2. Why are Pods designed to be disposable?

Kubernetes manages applications according to desired state rather than depending on individual workload identities.

If a Pod disappears, controllers can create a replacement.

---

### 3. Why shouldn't applications depend directly on Pod IP addresses?

Pod IP addresses can change whenever Pods are recreated.

Services provide stable networking and automatically route to currently available matching Pods.

---

### 4. How does a Kubernetes Service find Pods?

Services use label selectors.

Pods whose labels match the Service selector become eligible endpoints for the Service.

---

### 5. What does a Deployment manage underneath?

A Deployment manages ReplicaSets.

ReplicaSets maintain the required number of Pods.

---

### 6. Deployment vs StatefulSet?

Use a Deployment for interchangeable stateless workloads.

Use a StatefulSet when Pods require stable identities or persistent storage relationships.

---

### 7. What happens when a Kubernetes node dies?

Pods running on that node become unavailable.

Controllers detect that the desired replica count is no longer satisfied and replacement Pods can be scheduled onto healthy nodes.

---

### 8. Why does changing a Deployment container image trigger a rolling update?

The Deployment controller detects that the Pod template has changed and creates a new ReplicaSet, gradually replacing old Pods according to the Deployment's rolling-update strategy.

---

### 9. Are Kubernetes Secrets encrypted by Base64?

No.

Base64 is encoding, not encryption.

Encryption at rest, RBAC, IAM, and proper access controls are needed for stronger Secret protection.

---

### 10. How can Kubernetes Secrets be protected on EKS?

Security can include:

* Encryption at rest using AWS KMS
* Kubernetes RBAC
* IAM-based access controls
* Least-privilege permissions
* External secret management where appropriate

---

### 11. What's the difference between a Service and Ingress?

A Service provides stable networking and load balancing to Pods.

Ingress provides Layer 7 HTTP/HTTPS routing in front of one or more Services.

---

### 12. Why might LoadBalancer Services become inefficient?

Creating LoadBalancer-type Services can result in separate cloud load-balancer resources for exposed services.

Ingress allows multiple application routes to be consolidated behind a shared routing layer where appropriate.

---

### 13. What's the tradeoff between separate clusters and namespaces?

Separate clusters provide stronger isolation but increase cost and operational overhead.

Namespaces reduce cost and simplify cluster management but share the same cluster-level failure and security boundary.

---

### 14. Name something Kubernetes Deployment solves that was harder in the EC2 project.

Deployments provide built-in:

* Replica management
* Self-healing
* Rolling updates
* Rollbacks
* Declarative application lifecycle management

without requiring application-level EC2 replacement logic.

---




# Sprint 1 Status

```text
Pods                         ✅
Deployments                  ✅
ReplicaSets                  ✅
Services                     ✅
Service Types                ✅
ConfigMaps                   ✅
Secrets                      ✅
Namespaces                   ✅
StatefulSets                 ✅
DaemonSets                   ✅
EC2 → Kubernetes Mapping     ✅
Interview Preparation        ✅

Sprint 1: COMPLETE
```

# Next — Sprint 2: EKS Cluster via Terraform

Sprint 2 moves from concepts to real infrastructure.

The next phase will provision the Kubernetes platform on AWS using Terraform.

The focus will include:

```text
Amazon EKS Control Plane
          │
          ▼
EKS Managed Node Group
          │
          ▼
Worker Nodes
          │
          ▼
Kubernetes Workloads
```

Key areas to understand will include:

* EKS control plane architecture
* Worker nodes
* Managed Node Groups
* Public vs private EKS API endpoints
* IAM roles
* Kubernetes authentication
* OIDC provider
* IRSA
* EKS networking
* VPC CNI
* Security Groups
* AWS KMS encryption
* Terraform EKS provisioning

The goal will not simply be to get:

```bash
kubectl get nodes
```

to work.

The goal will be to understand **every AWS and Kubernetes component involved in making those nodes appear in the cluster.**
