# CI/CD Architecture Diagram and Communication Flow Overview

## 1. Objective

Design and implement a **CI/CD pipeline using Jenkins + SonarQube + Trivy (CI)** and **Argo CD + Kubernetes (CD)** following GitOps principles.

---

## 2. Infrastructure Plan (VM + IP Allocation)

Use a consistent private network (example: `192.168.0.0/24`)

| Component        | Hostname       | IP Address   | Purpose                |
| ---------------- | -------------- | ------------ | ---------------------- |
| Jenkins Master   | jenkins-master | 192.168.0.10 | Pipeline orchestration |
| Jenkins Agent    | jenkins-agent  | 192.168.0.11 | Build + Trivy scan     |
| SonarQube Server | sonar-server   | 192.168.0.12 | Code quality analysis  |
| Argo CD Server   | argo-server    | 192.168.0.20 | GitOps deployment      |
| K8s Master       | k8s-master     | 192.168.0.30 | Control plane          |
| K8s Worker-1     | k8s-worker1    | 192.168.0.31 | Application workload   |
| K8s Worker-2     | k8s-worker2    | 192.168.0.32 | Application workload   |

---

## 3. Architecture Overview

```
Developer
   │
   ▼
Git Repository (GitHub/GitLab)
   │ (Webhook)
   ▼
Jenkins Master (192.168.0.10)
   │
   ▼
Jenkins Agent (192.168.0.11)
   ├── Build Application
   ├── Docker Image Build
   ├── Trivy Scan
   │
   ├── SonarQube Scan → Sonar Server (192.168.0.12)
   │
   ▼
Container Registry (DockerHub/Harbor)
   │
   ▼
Update Kubernetes Manifest (Git)
   │
   ▼
Argo CD (192.168.0.20)
   │
   ▼
Kubernetes Cluster
   ├── Master (192.168.0.30)
   ├── Worker1 (192.168.0.31)
   └── Worker2 (192.168.0.32)
```

---

## 4. Required Ports & Communication Matrix

### 4.1 Jenkins Communication

| Source         | Destination    | Port                    | Purpose             |
| -------------- | -------------- | ----------------------- | ------------------- |
| Git Repo       | Jenkins Master | 8080                    | Webhook trigger     |
| Jenkins Master | Jenkins Agent  | 22 (SSH) / 50000 (JNLP) | Agent communication |
| Jenkins Agent  | Git Repo       | 443                     | Clone source code   |
| Jenkins Agent  | Registry       | 443                     | Push Docker image   |
| Jenkins Agent  | SonarQube      | 9000                    | Code analysis       |

#

### 4.2 SonarQube

| Source        | Destination | Port | Purpose          |
| ------------- | ----------- | ---- | ---------------- |
| Jenkins Agent | SonarQube   | 9000 | Analysis request |
| User Browser  | SonarQube   | 9000 | Web UI           |

#

### 4.3 Argo CD

| Source  | Destination    | Port       | Purpose        |
| ------- | -------------- | ---------- | -------------- |
| Argo CD | Git Repo       | 443        | Pull manifests |
| User    | Argo CD        | 8080 / 443 | UI access      |
| Argo CD | Kubernetes API | 6443       | Deploy apps    |

#

### 4.4 Kubernetes

| Source       | Destination | Port | Purpose           |
| ------------ | ----------- | ---- | ----------------- |
| Argo CD      | K8s Master  | 6443 | API communication |
| Worker Nodes | Registry    | 443  | Pull images       |
| Internal     | Pod Network | CNI  | Pod communication |

---


## 5. Security & Best Practices

1. Use SSH keys instead of passwords

2. Store secrets in:

   * Jenkins Credentials Manager
   * Kubernetes Secrets

3. Do NOT:

   * Build on Jenkins Master
   * Deploy directly from Jenkins

4. Always:

   * Use GitOps (Argo CD)
   * Separate repos:
     • App repo
     • Manifest repo

---

# 6. Installation Order

Follow this exact order:

1. Git Repository setup
2. Container Registry setup
3. Kubernetes Cluster setup
4. Argo CD installation
5. SonarQube installation
6. Jenkins Master setup
7. Jenkins Agent setup
8. Configure pipeline

---

# 7. Final Concept

- CI = Jenkins + SonarQube + Trivy
- CD = Argo CD + Kubernetes
- Git = Single Source of Truth
- Registry = Image Storage

---


