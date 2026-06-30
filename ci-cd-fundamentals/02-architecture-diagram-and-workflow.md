# CI/CD Architecture and Workflow Guide

## Introduction:

- This document describes the **Continuous Integration (CI)** and **Continuous Deployment (CD)** architecture used in this project. 
- The **CI pipeline** is implemented using **Jenkins for building**, **testing**, **scanning**, and **publishing** container **images**. 
- The **CD pipeline** follows the **GitOps** model using **Argo CD** to synchronize **Kubernetes deployments** from a **Git repository**.

---

## Architecture Overview

◼️ The **CI/CD architecture** is divided into **two** independent stages:

1. **Continuous Integration (CI):** Responsible for building, testing, scanning, and publishing container images.
2. **Continuous Deployment (CD):** Responsible for updating Kubernetes deployments using the GitOps model.

---

## GitOps Principles

This architecture follows the **GitOps model**, where the **Git repository** serves as the **single source of truth** for the Kubernetes cluster.

- Every deployment change is **committed** to Git.
- Argo CD continuously **monitors** the **repository**.
- The Kubernetes cluster is automatically **synchronized** with the desired state stored in Git.
- **Manual** changes made directly to the cluster are considered drift and are **reconciled** by **Argo CD**

---

## CI Diagram:

◼️ The **CI pipeline** concludes when the container image has been successfully **published** to the **container registry.**

```
                           ┌──────────────────────┐
                           │      Developer       │
                           └─────────┬────────────┘
                                     │ git push
                                     ▼
                           ┌──────────────────────┐
                           │ Application          │
                           │ Repository           │
                           │----------------------│
                           │ Source Code          │
                           │ Dockerfile           │
                           │ Jenkinsfile          │
                           └─────────┬────────────┘
                                     │ webhook
                                     ▼
                           ┌──────────────────────┐
                           │   Jenkins Master     │
                           └─────────┬────────────┘
                                     │ Dispatch Build
                                     ▼
                           ┌──────────────────────┐
                           │   Jenkins Agent      │
                           │   (Build Server)     │
                           └─────────┬────────────┘
                                     │
                                     ▼
                           ┌──────────────────────┐
                           │ Checkout Source Code │
                           └─────────┬────────────┘
                                     │
                                     ▼
                           ┌──────────────────────┐
                           │ Run SonarQube Scan   │
                           └─────────┬────────────┘
                                     │
                             Quality Gate Passed?
                                     │
                                     ▼
                           ┌──────────────────────┐
                           │ Build Docker Image   │
                           └─────────┬────────────┘
                                     │
                                     ▼
                           ┌──────────────────────┐
                           │ Run Trivy Scan       │
                           │ Image Vulnerability  │
                           └─────────┬────────────┘
                                     │
                             Scan Passed?
                                     │
                                     ▼
                           ┌──────────────────────┐
                           │ Build & Tag Image    │
                           └─────────┬────────────┘
                                     │
                                     │ docker push
                                     ▼
                           ┌──────────────────────┐
                           │     Docker Hub       │
                           │----------------------│
                           │      myapp:v1        │
                           └──────────────────────┘

```

#

### CI Workflow

- Developer pushes source code to the Git repository.
- Git webhook triggers Jenkins.
- Jenkins Master assigns the build to a Jenkins Agent.
- Jenkins Agent checks out the source code.
- SonarQube performs static code analysis.
- Jenkins builds the Docker image.
- Trivy scans the Docker image for vulnerabilities.
- If all quality gates pass, Jenkins tags and pushes the image to Docker Hub.

---

## CD Diagram

There are several common approaches for updating the **Kubernetes deployment manifest** (`deployment.yaml`) with the newly built image tag.

1. **Jenkins updates the manifest** (most common and simplest)
2. **Argo CD Image Updater** updates the manifest (or parameter) automatically (**recommended if you want Argo to manage image updates**)
3. Another GitOps automation tool (e.g., Flux Image Automation)

#

### ◼️ CD Approach 1 (Jenkins updates manifests)

In this approach, **Jenkins** is responsible for updating the Kubernetes **deployment manifest** after a successful build.

```
                            ┌──────────────────────┐
                            │    Docker Hub        │
                            │----------------------│
                            │ myapp:v1             │
                            └─────────┬────────────┘
                                      │
                                      │ Image tag reference
                                      ▼
                            ┌──────────────────────┐
                            │ Manifest Repository  │
                            │----------------------│
                            │ deployment.yaml      │
                            │ image: myapp:v1      │
                            └─────────┬────────────┘
                                      ▲
                                      │
                                      │ git push from Jenkins
                                      │ (updated image tag)
                                      │
                            ┌─────────┴────────────┐
                            │ Jenkins Agent        │
                            │----------------------│
                            │ Update Manifest      │
                            │ Commit & Push        │
                            └──────────────────────┘
                                      │
                                      ▼
                            ┌──────────────────────┐
                            │      Argo CD         │
                            │----------------------│
                            │ Watches Git          │
                            └─────────┬────────────┘
                                      │ Sync
                                      ▼
                            ┌──────────────────────┐
                            │ Kubernetes API Server│
                            └─────────┬────────────┘
                                      │ Apply manifests
                                      ▼
                                Worker Nodes
                                      │
                          ┌───────────┴───────────┐
                          │                       │
                          ▼                       ▼
                    ┌────────────────┐      ┌────────────────┐
                    │  Worker Node 1 │      │  Worker Node 2 │
                    │----------------│      │----------------│
                    │  myapp:v1 Pod  │      │  myapp:v1 Pod  │
                    └────────────────┘      └────────────────┘

```


### CD Workflow (Jenkins Updates Manifest)

- Jenkins updates the Kubernetes deployment manifest with the new image tag.
- Jenkins commits and pushes the updated manifest to the Git repository.
- Argo CD detects the Git change.
- Argo CD synchronizes the Kubernetes cluster.
- Kubernetes performs the rolling update.


---


### ◼️ CD Approach 2 (Argo CD Image Updater)

In this approach, **Argo CD Image Updater** is responsible for automatically updating the **Kubernetes deployment manifest** whenever a new **container image** is available in the **container registry**.

**Then Your CD diagram becomes:**

```
                           ┌──────────────────────┐
                           │    Docker Hub        │
                           │----------------------│
                           │ myapp:v1             │
                           └─────────┬────────────┘
                                     │
                                     │ New image available
                                     ▼
                           ┌──────────────────────┐
                           │ Argo CD Image Updater│
                           │----------------------│
                           │ Detect New Image     │
                           │ Update Manifest      │
                           │ Commit & Push        │
                           └─────────┬────────────┘
                                     │ git push
                                     ▼
                           ┌──────────────────────┐
                           │ Manifest Repository  │
                           │----------------------│
                           │ deployment.yaml      │
                           │ image: myapp:v1      │
                           └─────────┬────────────┘
                                     │
                                     │ Watches Git
                                     ▼
                           ┌──────────────────────┐
                           │      Argo CD         │
                           │----------------------│
                           │ Detect Changes       │
                           │ Sync Application     │
                           └─────────┬────────────┘
                                     │ Sync
                                     ▼
                           ┌──────────────────────┐
                           │ Kubernetes API Server│
                           └─────────┬────────────┘
                                     │ Apply manifests
                                     ▼
                                Worker Nodes
                                     │
                         ┌───────────┴───────────┐
                         │                       │
                         ▼                       ▼
                 ┌────────────────┐      ┌────────────────┐
                 │  Worker Node 1 │      │  Worker Node 2 │
                 │----------------│      │----------------│
                 │  myapp:v1 Pod  │      │  myapp:v1 Pod  │
                 └────────────────┘      └────────────────┘
```


### CD Workflow (Argo CD Image Updater)

- Jenkins publishes the container image to Docker Hub.
- Argo CD Image Updater detects the new image tag.
- It updates the deployment manifest (or image parameter) in the Git repository.
- Argo CD detects the Git commit.
- Argo CD synchronizes the Kubernetes cluster.
- Kubernetes deploys the updated application.

---

## Comparison of the two approaches

**Two deployment approaches are presented in this document:**

- `Approach 1:` \
  **Jenkins** updates the **deployment manifest** after a successful build. This approach is simple and widely used in **traditional CI/CD environments.**

- `Approach 2:` \
  **Argo CD** Image Updater automatically updates the **deployment manifest** when a new container image is available. This approach follows **GitOps principles** more closely by separating image publishing from deployment automation.

>[!Note]
**Image Tags:**\
**Note:** Kubernetes **manifests** do not contain the **container image itself**. They only **reference** the **container image** stored in **Docker Hub** (for example, myapp:v1). During **deployment**, Kubernetes **pulls** the image from the container **registry**.

---

## Complete Architecture

>[!Note]
The following diagrams provide a simplified, **high-level** view of the complete **CI/CD workflow**. Refer to the individual **CI** and **CD** diagrams for detailed process flows.

### Complete Architecture (Jenkins approach)

If we combine the CI and CD **(using jenkins)** diagrams, the handoff point is:

```
CI:
Developer
      │
      ▼
Git Repository
      │
      ▼
Jenkins Server
      │
      ▼
Checkout Source Code
      │
      ├─────────────► SonarQube
      │
      ▼
Build Docker Image
      │
      ▼
Trivy Scan
      │
      ▼
Push Image

CD:
Git Repository (deployment.yaml updated)
   │
   ▼
Argo CD
   │
   ▼
Kubernetes API Server
   │
   ▼
Worker Nodes
```

◼️ This approach follows a **Jenkins-managed GitOps** workflow, where Jenkins is responsible for updating the **deployment manifests** before Argo CD synchronizes the cluster.

- Jenkins builds and pushes the Docker image.
- Jenkins updates `deployment.yaml` with the new image tag.
- Jenkins commits and pushes the manifest change to Git.
- Argo CD detects the Git change.
- Argo CD syncs the Kubernetes cluster.
- Kubernetes pulls the image from Docker Hub and creates the new Pods.

This separation cleanly **distinguishes** the **CI responsibility (Jenkins)** from the **CD responsibility (Argo CD)**.


#

### Complete Architecture (Image Updater approach)

Our overall pipeline would then be:

```text
Developer
    │
    ▼
Git Repository (Application Source)
    │
    ▼
Jenkins
    │
    ├── Checkout Source Code
    ├── SonarQube Scan
    ├── Build Image
    ├── Trivy Scan
    └── Push Image to Docker Hub
               │
               ▼
         Docker Hub
               │
               ▼
     Argo CD Image Updater
               │
               ▼
Git Repository (Kubernetes Manifests)
               │
               ▼
            Argo CD
               │
               ▼
        Kubernetes Cluster
```

◼️ This is a **modern GitOps architecture**, where:

* **Jenkins** is responsible only for CI.
* **Argo CD Image Updater** is responsible for updating the image tag in Git.
* **Argo CD** is responsible only for deployment.
* **Kubernetes** runs the application.

>[!Note]
**Recommended:**
This approach follows GitOps best practices by clearly separating CI from CD. Jenkins is responsible for building and publishing container images, while Argo CD Image Updater manages image tag updates in Git and Argo CD synchronizes the Kubernetes cluster with the desired state stored in the Git repository.

---

## Assumptions

- Jenkins is configured with Git webhook integration.
- Docker Hub is used as the container registry.
- Kubernetes manifests are stored in a Git repository.
- Argo CD has access to the Kubernetes cluster.
- Required credentials are configured in Jenkins and Argo CD.
- SonarQube and Trivy are integrated into the Jenkins pipeline.

---

## Conclusion

- This document presents two CI/CD implementation approaches using Jenkins, Docker Hub, Kubernetes, and Argo CD.
- While both approaches successfully automate application delivery, the Argo CD Image Updater approach better aligns with GitOps best practices by separating image publishing from deployment automation.
- For new deployments, the GitOps-based approach is recommended because it provides improved maintainability, scalability, and separation of responsibilities.

---