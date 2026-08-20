# Production-Grade CI/CD & DevSecOps Roadmap

## Local Linux VM Lab — Production-Grade Architecture and Practices

**Document Type:** Standard Operating Procedure / Learning Roadmap
**Scope:** CI/CD, DevSecOps, GitOps, Containers, Kubernetes
**Automation Tools Excluded:** Ansible, Terraform
**Kubernetes Distribution:** kubeadm
**Practice Environment:** Local Linux Virtual Machines
**Target Architecture:** Production-grade enterprise CI/CD design reproduced in a local VM laboratory

---

# 1. Purpose

This SOP defines a structured, production-grade roadmap for learning and implementing an end-to-end **CI/CD, DevSecOps, and GitOps platform** using Linux Virtual Machines.

The objective is not to create a production system on a local VM. Instead, the local VM environment will be used to **practice the same architecture, communication flow, security controls, deployment strategies, operational procedures, and troubleshooting techniques that would be expected in a production environment**.

The final implementation will provide the following lifecycle:

```text
Developer
    │
    ▼
Git Repository
    │
    ▼
Jenkins CI
    │
    ├── Build
    ├── Unit Test
    ├── Gitleaks
    ├── SonarQube
    ├── Quality Gate
    ├── Docker Build
    ├── Trivy Scan
    └── Image Push
             │
             ▼
      Container Registry
             │
             ▼
       GitOps Repository
             │
             ▼
          Argo CD
             │
             ▼
       Kubernetes Cluster
             │
             ▼
       Application Pods
             │
             ▼
      Health Verification
```

---

# 2. Objectives

By completing this roadmap, the learner shall be able to:

1. Build and operate Linux-based CI/CD infrastructure.
2. Understand Git-based software delivery.
3. Build secure container images.
4. Implement Jenkins declarative pipelines.
5. Separate Jenkins Controller and Jenkins Agent responsibilities.
6. Integrate SonarQube into CI.
7. Implement Quality Gates.
8. Detect secrets using Gitleaks.
9. Perform filesystem/dependency and container image vulnerability scanning using Trivy.
10. Operate a container registry.
11. Build a Kubernetes cluster using kubeadm.
12. Understand Kubernetes networking and container runtime architecture.
13. Deploy applications using Kubernetes manifests.
14. Package applications using Helm.
15. Implement GitOps using Argo CD.
16. Implement automated Kubernetes deployment through Git changes.
17. Implement rolling updates and rollback procedures.
18. Understand Kubernetes health checks.
19. Troubleshoot failed CI/CD pipelines.
20. Troubleshoot failed Kubernetes deployments.
21. Understand production-grade CI/CD architecture without depending on Ansible or Terraform.

---

# 3. Scope

This SOP covers:

### Infrastructure

* Linux
* SSH
* Networking
* Firewall
* DNS
* TLS concepts
* Systemd
* Docker
* containerd
* kubeadm
* Kubernetes

### CI

* Git
* Jenkins
* Jenkins Controller
* Jenkins Agent
* Jenkinsfile
* Declarative Pipeline
* Credentials
* Webhooks
* Artifacts

### DevSecOps

* Gitleaks
* SonarQube
* SonarQube Quality Gates
* Trivy
* Dependency scanning
* Filesystem scanning
* Container image scanning

### Containerization

* Dockerfile
* Multi-stage builds
* Image tagging
* Image optimization
* Non-root containers
* Health checks
* Container registry

### CD / GitOps

* Argo CD
* GitOps repository
* Kubernetes manifests
* Helm
* Automated synchronization
* Drift detection
* Self-healing
* Rollback

### Kubernetes

* kubeadm
* containerd
* Control Plane
* Worker Nodes
* CNI
* Pods
* Deployments
* Services
* ConfigMaps
* Secrets
* Ingress
* Probes
* Resources
* Rolling updates
* Rollbacks

---

# 4. Tools and Technology Stack

| Category                  | Technology                                     |
| ------------------------- | ---------------------------------------------- |
| Operating System          | Ubuntu Server / RHEL-compatible Linux          |
| Source Control            | Git                                            |
| CI                        | Jenkins LTS                                    |
| CI Agent                  | Dedicated Jenkins Agent VM                     |
| Code Quality              | SonarQube Community                            |
| Secret Scanning           | Gitleaks                                       |
| Vulnerability Scanner     | Trivy                                          |
| Container Engine          | Docker Engine                                  |
| Kubernetes Runtime        | containerd                                     |
| Container Registry        | Docker Registry initially; Harbor later        |
| Kubernetes                | kubeadm                                        |
| Kubernetes CNI            | Calico or Cilium                               |
| Package Management        | Helm 3                                         |
| GitOps CD                 | Argo CD                                        |
| Application               | FastAPI / Node.js / similar simple application |
| Automation                | Jenkins Pipeline                               |
| Infrastructure Automation | Not included                                   |
| Terraform                 | Not included                                   |
| Ansible                   | Not included                                   |

---

# 5. Target Production-Grade Architecture

The final laboratory architecture shall resemble:

```text
                           Developer
                               │
                               ▼
                       Git Application Repo
                               │
                               │ Webhook
                               ▼
                    ┌──────────────────────┐
                    │ Jenkins Controller   │
                    │        VM-1          │
                    └──────────┬───────────┘
                               │
                               │ Pipeline
                               ▼
                    ┌──────────────────────┐
                    │ Jenkins Agent        │
                    │        VM-2          │
                    │ Build / Test / Scan  │
                    └───────┬──────┬───────┘
                            │      │
              ┌─────────────┘      └──────────────┐
              ▼                                    ▼
       ┌─────────────┐                    ┌─────────────┐
       │ SonarQube   │                    │ Trivy /     │
       │    VM-3     │                    │ Gitleaks    │
       └─────────────┘                    └─────────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │ Container       │
                   │ Registry        │
                   │ VM / Service    │
                   └────────┬────────┘
                            │
                            ▼
                    GitOps Repository
                            │
                            │ Pull
                            ▼
                    ┌─────────────────┐
                    │    Argo CD      │
                    │      VM-4       │
                    └────────┬────────┘
                             │
                             │ Kubernetes API
                             ▼
                 ┌──────────────────────────┐
                 │   Kubernetes Cluster     │
                 │        kubeadm           │
                 │                          │
                 │ Control Plane            │
                 │ Worker Node 1            │
                 │ Worker Node 2            │
                 └──────────────────────────┘
```

---

# 6. Important Architectural Principle

The CI and CD responsibilities shall remain logically separated.

## CI

Jenkins is responsible for:

```text
Source
  ↓
Build
  ↓
Test
  ↓
Security Scan
  ↓
Code Quality
  ↓
Container Build
  ↓
Image Scan
  ↓
Registry Push
  ↓
GitOps Repository Update
```

## CD

Argo CD is responsible for:

```text
GitOps Repository
       ↓
Desired Kubernetes State
       ↓
Argo CD
       ↓
Kubernetes
       ↓
Application Deployment
       ↓
Health / Drift Monitoring
```

Jenkins shall **not directly deploy the application to Kubernetes** in the final GitOps architecture.

---

# 7. Repository Architecture

Two repositories shall be used.

## 7.1 Application Repository

Example:

```text
my-app/
├── src/
├── tests/
├── Dockerfile
├── .dockerignore
├── Jenkinsfile
├── requirements.txt
└── README.md
```

This repository contains:

* Application source code
* Tests
* Dockerfile
* Jenkinsfile
* Application documentation

---

## 7.2 GitOps Repository

Example:

```text
my-app-gitops/
├── environments/
│   ├── dev/
│   │   └── my-app/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── ingress.yaml
│   │       └── kustomization.yaml
│   │
│   └── prod/
│       └── my-app/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           └── kustomization.yaml
│
└── helm/
    └── my-app/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

The GitOps repository becomes the **source of truth for Kubernetes desired state**.

---

# 8. Phase 0 — Linux and Networking Foundation

Before installing CI/CD tools, establish a strong Linux foundation.

## Required Knowledge

### Linux

Learn and practice:

```text
systemctl
journalctl
ps
top
ss
ip
df
du
mount
find
grep
awk
sed
tar
curl
wget
ssh
scp
rsync
chmod
chown
useradd
sudo
```

### Services

Understand:

```text
systemd
service startup
service dependencies
logs
restart policies
resource limits
```

### Networking

Understand:

```text
IP Address
Subnet
Gateway
DNS
TCP
UDP
Ports
HTTP
HTTPS
TLS
Firewall
NAT
Proxy
```

### Required Testing Tools

```text
ping
curl
nc
ss
dig
nslookup
traceroute
```

## Milestone

The learner must be able to troubleshoot:

```text
Application cannot reach Jenkins
Jenkins cannot reach Git
Jenkins cannot reach SonarQube
Jenkins cannot reach Registry
Argo CD cannot reach Git
Kubernetes node cannot reach API Server
Pod cannot reach Service
```

without immediately assuming that the application is broken.

---

# 9. Phase 1 — Git

Learn:

```text
Repository
Commit
Branch
Merge
Tag
Remote
Clone
Pull
Push
SSH authentication
PAT/token
Webhook
```

Practice:

```text
Developer
    ↓
Git Commit
    ↓
Remote Repository
    ↓
Webhook
    ↓
Jenkins
```

## Milestone

A Git push must automatically trigger Jenkins.

---

# 10. Phase 2 — Docker and Containerization

Install Docker Engine on the appropriate CI/build VM.

Learn:

```text
Image
Container
Layer
Registry
Dockerfile
Build Context
Volume
Network
Port Mapping
Multi-stage Build
Build Cache
```

## Production Practices

Docker images shall:

* Use minimal base images.
* Use multi-stage builds where appropriate.
* Avoid unnecessary packages.
* Run as a non-root user.
* Use `.dockerignore`.
* Avoid hard-coded secrets.
* Use explicit image tags.
* Include appropriate health checks.
* Be reproducible.

Example lifecycle:

```text
Source
  ↓
Dockerfile
  ↓
Docker Build
  ↓
Image
  ↓
Security Scan
  ↓
Registry
```

## Milestone

Build and run:

```text
registry/my-app:1.0.0
```

from the local registry.

---

# 11. Phase 3 — Jenkins Infrastructure

Deploy Jenkins using a production-style Controller/Agent model.

## Jenkins Controller

Responsibilities:

```text
Pipeline orchestration
Job management
Credentials management
Build scheduling
Pipeline metadata
```

## Jenkins Agent

Responsibilities:

```text
Source checkout
Compilation
Testing
Docker build
Security scanning
Artifact generation
Registry push
```

The Jenkins Controller shall not be treated as the primary build server.

## Milestone

Jenkins Controller successfully launches jobs on the Jenkins Agent.

---

# 12. Phase 4 — Jenkins Declarative Pipeline

Build the first pipeline using a `Jenkinsfile`.

Recommended structure:

```text
Checkout
   ↓
Validate
   ↓
Build
   ↓
Unit Test
   ↓
Package
   ↓
Docker Build
   ↓
Archive/Test Artifacts
```

Learn:

```text
pipeline
agent
environment
options
parameters
stages
steps
post
when
credentials
```

Also learn:

* Build numbers
* Environment variables
* Workspace management
* Pipeline logs
* Pipeline failure handling
* Artifacts
* Credentials

## Milestone

A successful Git commit automatically produces a tested Docker image.

---

# 13. Phase 5 — SonarQube Integration

Integrate SonarQube into Jenkins.

Pipeline:

```text
Checkout
   ↓
Build
   ↓
Unit Test
   ↓
SonarQube Analysis
   ↓
Quality Gate
   ↓
Continue
```

Learn:

```text
Bugs
Vulnerabilities
Code Smells
Coverage
Duplications
Security Hotspots
Quality Profiles
Quality Gates
```

The pipeline shall stop when the Quality Gate fails.

## Failure Test

Intentionally introduce code that violates the configured Quality Gate.

Expected:

```text
SonarQube
    ↓
Quality Gate FAILED
    ↓
Jenkins FAILURE
    ↓
No image push
```

---

# 14. Phase 6 — Gitleaks

Integrate secret scanning.

Pipeline:

```text
Checkout
   ↓
Gitleaks
   ↓
Build
```

Detect:

```text
API keys
Passwords
Tokens
Private keys
Credentials
Secrets
```

## Failure Test

Commit a controlled test secret.

Expected:

```text
Gitleaks
   ↓
Secret detected
   ↓
Pipeline FAILURE
```

The test secret must never be a real credential.

---

# 15. Phase 7 — Trivy Security Scanning

Use Trivy at multiple stages.

## Filesystem / Dependency Scan

```text
Source
   ↓
Trivy FS
   ↓
Dependency vulnerabilities
```

## Container Image Scan

```text
Docker Build
   ↓
Trivy Image
   ↓
Vulnerability Assessment
   ↓
Registry Push
```

Recommended initial policy:

```text
HIGH
CRITICAL
```

The pipeline shall fail according to the defined vulnerability policy.

## Important Principle

Security scanning must happen **before the image is promoted for deployment**.

---

# 16. Phase 8 — Container Registry

Initially use a local registry for laboratory purposes.

Example architecture:

```text
Jenkins Agent
      │
      │ docker push
      ▼
Local Registry
      │
      │ image pull
      ▼
Kubernetes
```

Learn:

```text
Repository
Image Tag
Image Digest
Push
Pull
Authentication
TLS
Registry Storage
```

## Image Tagging Strategy

Do not depend on:

```text
latest
```

Use immutable or traceable tags such as:

```text
my-app:1.0.0
my-app:1.0.1
my-app:git-a82f91c
```

A production-grade implementation should be able to identify exactly which source revision produced an image.

---

# 17. Phase 9 — Kubernetes Architecture Using kubeadm

The Kubernetes practice cluster shall be built using **kubeadm**.

Recommended architecture:

```text
Kubernetes Cluster
│
├── Control Plane
│
├── Worker Node 1
│
└── Worker Node 2
```

The Kubernetes container runtime shall be:

```text
containerd
```

The cluster shall include:

```text
kubeadm
kubelet
kubectl
containerd
CNI
```

The CNI shall be selected and documented during cluster preparation.

---

# 18. Kubernetes Learning Sequence

Do not immediately deploy Argo CD.

First learn Kubernetes independently.

Sequence:

```text
Cluster
   ↓
Nodes
   ↓
Namespaces
   ↓
Pods
   ↓
Deployments
   ↓
ReplicaSets
   ↓
Services
   ↓
ConfigMaps
   ↓
Secrets
   ↓
Ingress
   ↓
Resource Requests/Limits
   ↓
Probes
   ↓
Rolling Updates
   ↓
Rollback
```

---

# 19. Kubernetes Application Deployment

First deploy the application manually.

Example:

```text
Docker Registry
      ↓
Kubernetes Deployment
      ↓
Pods
      ↓
Service
      ↓
Application
```

Verify:

```text
kubectl get nodes
kubectl get pods
kubectl get deployments
kubectl get services
kubectl logs
kubectl describe
```

The objective is to understand Kubernetes **before introducing GitOps**.

---

# 20. Kubernetes Health Management

Production-grade deployments shall use health probes where appropriate.

Learn:

### Startup Probe

Determines whether the application has successfully started.

### Readiness Probe

Determines whether the application can receive traffic.

### Liveness Probe

Determines whether the application needs to be restarted.

Architecture:

```text
Application
    │
    ├── startupProbe
    ├── readinessProbe
    └── livenessProbe
```

These probes shall be tested by intentionally creating unhealthy application states.

---

# 21. Kubernetes Resource Management

Learn:

```text
Requests
Limits
CPU
Memory
QoS
```

Example concept:

```text
Container
 ├── CPU Request
 ├── CPU Limit
 ├── Memory Request
 └── Memory Limit
```

This is important for production-grade scheduling and resource protection.

---

# 22. Kubernetes Deployment Strategy

Use:

```text
Deployment
    ↓
RollingUpdate
```

Understand:

```text
replicas
maxUnavailable
maxSurge
readinessProbe
terminationGracePeriodSeconds
```

Test:

```text
v1
 ↓
v2
 ↓
Rolling Update
 ↓
Health Verification
```

---

# 23. Kubernetes Rollback

Practice both:

### Kubernetes rollback

```text
kubectl rollout history
kubectl rollout undo
```

### GitOps rollback

```text
Git commit
   ↓
Revert
   ↓
Argo CD
   ↓
Previous Kubernetes state
```

The GitOps rollback is the preferred final operational model.

---

# 24. Phase 10 — Helm

After understanding raw Kubernetes manifests, introduce Helm.

Learn:

```text
Chart
Chart.yaml
values.yaml
templates
Helm Release
helm install
helm upgrade
helm rollback
```

Environment-specific configuration:

```text
values-dev.yaml
values-prod.yaml
```

Example:

```text
Helm Chart
     │
     ├── Development Values
     │
     └── Production Values
```

---

# 25. Phase 11 — Argo CD

Install Argo CD inside the Kubernetes cluster.

Architecture:

```text
GitOps Repository
       │
       │ Pull
       ▼
    Argo CD
       │
       │ Kubernetes API
       ▼
 Kubernetes Cluster
```

Learn:

```text
Application
Project
Repository
Target Revision
Path
Sync
Auto Sync
Prune
Self Heal
Health
Drift
```

---

# 26. GitOps Operating Model

The final CD architecture shall follow:

```text
Git
 │
 │ Desired State
 ▼
Argo CD
 │
 │ Reconciliation
 ▼
Kubernetes
```

Jenkins shall not execute:

```text
kubectl apply
```

as the primary production CD mechanism.

Instead:

```text
Jenkins
   ↓
Update image reference
   ↓
Commit GitOps repository
   ↓
Argo CD detects commit
   ↓
Argo CD synchronizes
   ↓
Kubernetes deployment
```

---

# 27. Phase 12 — Complete CI/CD Integration

The final Jenkins pipeline shall resemble:

```text
Stage 1
Checkout
     ↓
Stage 2
Gitleaks
     ↓
Stage 3
Build
     ↓
Stage 4
Unit Test
     ↓
Stage 5
SonarQube
     ↓
Stage 6
Quality Gate
     ↓
Stage 7
Docker Build
     ↓
Stage 8
Trivy Image Scan
     ↓
Stage 9
Registry Push
     ↓
Stage 10
Update GitOps Repository
```

Then:

```text
GitOps Repository
       ↓
     Argo CD
       ↓
 Kubernetes
       ↓
Rolling Update
       ↓
Readiness Check
       ↓
Deployment Healthy
```

---

# 28. Development and Production Environments

The final lab should simulate:

```text
Kubernetes
│
├── dev namespace
│
└── prod namespace
```

The workflow should be:

```text
Developer
   ↓
Application Repository
   ↓
Jenkins
   ↓
Build/Test/Scan
   ↓
Registry
   ↓
GitOps Repository
   ↓
Development
   ↓
Validation
   ↓
Production
```

Promotion should be controlled through Git changes rather than direct Kubernetes commands.

---

# 29. Production-Grade Security Model

The lab shall practice the following principles.

## Secrets

Never store credentials inside:

```text
Git
Dockerfile
Jenkinsfile
Kubernetes YAML
```

Use:

```text
Jenkins Credentials
Kubernetes Secrets
External secret-management concepts
```

where appropriate.

---

## Jenkins

Practice:

* Least-privilege credentials.
* Dedicated agents.
* Restricted permissions.
* Credential IDs instead of hard-coded secrets.
* Pipeline approval where appropriate.
* Secure webhook configuration.

---

## Container Security

Practice:

* Minimal images.
* Non-root containers.
* Vulnerability scanning.
* Immutable image tags.
* No embedded secrets.
* Dependency scanning.

---

## Kubernetes

Practice:

* Namespaces.
* RBAC.
* Service Accounts.
* Secrets.
* Resource limits.
* Network policies as an advanced topic.
* Pod security concepts.
* Least privilege.

---

# 30. CI/CD Failure Testing

The lab shall intentionally test failures.

## Test 1 — Git Failure

```text
Invalid credentials
     ↓
Checkout failure
```

## Test 2 — Unit Test Failure

```text
Unit Test FAILED
     ↓
Pipeline STOP
```

## Test 3 — Gitleaks Failure

```text
Secret detected
     ↓
Pipeline STOP
```

## Test 4 — SonarQube Failure

```text
Quality Gate FAILED
     ↓
Pipeline STOP
```

## Test 5 — Trivy Failure

```text
Critical vulnerability
     ↓
Pipeline STOP
```

## Test 6 — Registry Failure

```text
Registry unavailable
     ↓
Image push failure
```

## Test 7 — Kubernetes Failure

```text
Bad image
     ↓
Pod ImagePullBackOff
```

## Test 8 — Readiness Failure

```text
Application unhealthy
     ↓
Readiness probe FAILED
     ↓
Traffic not sent to pod
```

## Test 9 — Argo CD Drift

```text
Manual Kubernetes change
     ↓
Drift detected
     ↓
Argo CD Self-Heal
```

## Test 10 — Rollback

```text
Bad release
     ↓
Failure
     ↓
Git revert
     ↓
Argo CD
     ↓
Previous version restored
```

---

# 31. Observability and Operational Verification

The final roadmap should also introduce basic observability.

Learn:

```text
Application Logs
Container Logs
Kubernetes Events
Jenkins Logs
Argo CD Logs
System Logs
```

Practice:

```text
journalctl
kubectl logs
kubectl describe
kubectl get events
```

Monitoring tools such as:

```text
Prometheus
Grafana
```

may be added as an **advanced extension**, after the CI/CD pipeline itself is complete.

---

# 32. Final Production-Grade Pipeline

The completed laboratory shall demonstrate:

```text
                         ┌───────────────────┐
                         │     Developer     │
                         └─────────┬─────────┘
                                   │
                                   ▼
                         ┌───────────────────┐
                         │ Application Git   │
                         └─────────┬─────────┘
                                   │
                                Webhook
                                   │
                                   ▼
                         ┌───────────────────┐
                         │ Jenkins Controller│
                         └─────────┬─────────┘
                                   │
                                   ▼
                         ┌───────────────────┐
                         │   Jenkins Agent   │
                         └─────────┬─────────┘
                                   │
                     ┌─────────────┼─────────────┐
                     │             │             │
                     ▼             ▼             ▼
                 Gitleaks      Unit Tests    SonarQube
                     │             │             │
                     └─────────────┼─────────────┘
                                   ▼
                              Quality Gate
                                   │
                                   ▼
                             Docker Build
                                   │
                                   ▼
                              Trivy Scan
                                   │
                              PASS │
                                   ▼
                         Container Registry
                                   │
                                   ▼
                         GitOps Repository
                                   │
                                   ▼
                               Argo CD
                                   │
                                   ▼
                         Kubernetes API
                                   │
                    ┌──────────────┴──────────────┐
                    ▼                             ▼
             Control Plane                  Worker Nodes
                                                  │
                                                  ▼
                                             Application
                                                  │
                                      ┌───────────┴──────────┐
                                      ▼                      ▼
                                Readiness Probe        Liveness Probe
                                      │
                                      ▼
                               Healthy Release
```

---

# 33. Final Project Milestones

## Milestone 1 — Linux Foundation

* [ ] Linux administration
* [ ] SSH
* [ ] Networking
* [ ] Firewall
* [ ] Systemd
* [ ] Git

**Exit Criteria:** Linux VM infrastructure can be administered and network connectivity can be independently diagnosed.

---

## Milestone 2 — Containerization

* [ ] Docker installed
* [ ] Dockerfile created
* [ ] Multi-stage build implemented
* [ ] Non-root container implemented
* [ ] Local registry deployed
* [ ] Image pushed and pulled

**Exit Criteria:** Application can be containerized and retrieved from the registry.

---

## Milestone 3 — Jenkins CI

* [ ] Jenkins Controller deployed
* [ ] Jenkins Agent deployed
* [ ] Controller-Agent communication working
* [ ] Git webhook configured
* [ ] Declarative Jenkinsfile created
* [ ] Unit tests automated
* [ ] Docker build automated

**Exit Criteria:** Git commit automatically triggers a successful CI pipeline.

---

## Milestone 4 — DevSecOps

* [ ] SonarQube integrated
* [ ] Quality Gate enforced
* [ ] Gitleaks integrated
* [ ] Trivy filesystem scan implemented
* [ ] Trivy image scan implemented
* [ ] Pipeline failure policies implemented

**Exit Criteria:** Vulnerable or insecure code cannot proceed through the defined CI gates.

---

## Milestone 5 — Kubernetes

* [ ] Control Plane created using kubeadm
* [ ] Worker 1 joined
* [ ] Worker 2 joined
* [ ] containerd configured
* [ ] CNI installed
* [ ] Nodes healthy
* [ ] Application deployed manually
* [ ] Service configured
* [ ] Probes configured
* [ ] Rolling update tested
* [ ] Rollback tested

**Exit Criteria:** Application can be deployed and operated manually on the kubeadm cluster.

---

## Milestone 6 — Helm

* [ ] Helm installed
* [ ] Chart created
* [ ] Values configured
* [ ] Dev configuration created
* [ ] Production configuration created
* [ ] Helm upgrade tested
* [ ] Helm rollback tested

**Exit Criteria:** Application can be packaged and deployed using Helm.

---

## Milestone 7 — Argo CD

* [ ] Argo CD installed
* [ ] GitOps repository configured
* [ ] Application created
* [ ] Manual synchronization tested
* [ ] Automated synchronization tested
* [ ] Drift detection tested
* [ ] Self-healing tested

**Exit Criteria:** Git becomes the source of truth for Kubernetes desired state.

---

## Milestone 8 — Complete CI/CD

* [ ] Jenkins builds application
* [ ] Unit tests execute
* [ ] Gitleaks executes
* [ ] SonarQube executes
* [ ] Quality Gate executes
* [ ] Docker image builds
* [ ] Trivy scans image
* [ ] Image pushed to registry
* [ ] Jenkins updates GitOps repository
* [ ] Argo CD detects Git change
* [ ] Kubernetes performs rolling update
* [ ] Application health verified

**Exit Criteria:** A single application code change can travel from Git commit to Kubernetes deployment automatically.

---

# 34. Advanced Production Topics

After completing the main pipeline, study:

```text
RBAC
NetworkPolicy
Pod Security
TLS everywhere
Private Container Registry
Harbor
Ingress Controller
cert-manager
External Secrets
Secret Management
Image Signing
Cosign
SBOM
Trivy SBOM
Software Supply Chain Security
Kubernetes Audit Logs
Resource Quotas
LimitRanges
Horizontal Pod Autoscaler
Prometheus
Grafana
Alertmanager
Centralized Logging
Backup and Recovery
Disaster Recovery
High Availability
Jenkins HA concepts
Argo CD HA concepts
Kubernetes HA
```

These are **advanced extensions**, not prerequisites for completing the core CI/CD project.

---

# 35. What Is Deliberately Excluded

The following are intentionally outside this roadmap:

```text
Ansible
Terraform
Cloud Infrastructure Provisioning
AWS
Azure
GCP
```

The reason is that this project is focused specifically on mastering:

```text
Linux
   ↓
Git
   ↓
Docker
   ↓
Jenkins
   ↓
DevSecOps
   ↓
Registry
   ↓
Kubernetes
   ↓
Helm
   ↓
Argo CD
   ↓
GitOps
   ↓
CI/CD
```

Infrastructure-as-Code can be learned separately later.

---

# 36. Final Target State

At the completion of this roadmap, the local VM environment should behave conceptually like a production CI/CD platform:

```text
                 SOFTWARE DELIVERY PLATFORM

Developer
    │
    ▼
Git
    │
    ▼
Jenkins
    │
    ├── Build
    ├── Test
    ├── Gitleaks
    ├── SonarQube
    ├── Quality Gate
    ├── Docker Build
    └── Trivy
             │
             ▼
         Registry
             │
             ▼
        GitOps Repo
             │
             ▼
          Argo CD
             │
             ▼
       kubeadm Cluster
             │
       ┌─────┴─────┐
       ▼           ▼
    Worker 1     Worker 2
       │           │
       └─────┬─────┘
             ▼
        Application
             │
       ┌─────┴─────┐
       ▼           ▼
   Readiness    Liveness
     Probe        Probe
       │
       ▼
    Healthy
```

The **local VM environment is only the practice platform**. The architecture, security controls, GitOps model, deployment process, rollback procedures, and troubleshooting methodology are designed to mirror what would be expected when the same platform is eventually implemented on production infrastructure.

---

# 37. Recommended Execution Order

The roadmap shall be executed sequentially:

```text
01. Linux + Networking
        ↓
02. Git
        ↓
03. Docker
        ↓
04. Jenkins Controller/Agent
        ↓
05. Jenkins Declarative Pipeline
        ↓
06. SonarQube
        ↓
07. Gitleaks
        ↓
08. Trivy
        ↓
09. Container Registry
        ↓
10. kubeadm + containerd
        ↓
11. Kubernetes Fundamentals
        ↓
12. Kubernetes Application Deployment
        ↓
13. Kubernetes Health/Rolling/Rollback
        ↓
14. Helm
        ↓
15. Argo CD
        ↓
16. GitOps
        ↓
17. Jenkins → GitOps Integration
        ↓
18. Complete CI/CD Pipeline
        ↓
19. Failure Testing
        ↓
20. Security Hardening
        ↓
21. Observability
        ↓
22. Enterprise Architecture Simulation
```

**Final objective:**

> Build the platform once manually in the local VM laboratory, understand every communication path and component, break it intentionally, troubleshoot it, secure it, and finally operate the complete Jenkins → DevSecOps → Registry → GitOps → Argo CD → kubeadm Kubernetes delivery pipeline as a production-grade system.
