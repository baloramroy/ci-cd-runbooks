# **Total Overview of CI/CD pipeline.**

This **SOP** is devided in two parts:

1. CI Process (what happens step-by-step)
2. CD Process (what happens after CI)

---

# 1. CI (Continuous Integration) — SOP

## Objective

Transform source code into a **validated, secure, deployable artifact (Docker image)**.

---

## CI Trigger

1. Developer pushes code to Git repository
2. Webhook triggers **Jenkins pipeline**

---

## CI Pipeline Flow

### 1. Source Code Checkout

* Jenkins pulls latest code from Git
* Branch-based logic may apply (dev, main, etc.)

#

### 2. Build Stage

Executed on **Build Server (Agent)**

* Compile application

  * Example:

    * Java → Maven/Gradle
    * Node.js → npm build
* Resolve dependencies
* Build ensures the code is structurally sound.

✔️ Output:

* Compiled binaries / build artifacts

#

### 3. Unit Testing
Executed on **Build Server (Agent)**
* Run automated tests
* Validate code functionality
* Unit tests ensure it's logically sound.

✔️ Decision:

* ❌ Fail → Pipeline stops
* ✔️ Pass → Continue

#

### 4. Code Quality Analysis

Using **SonarQube**

* Analyze:

  * Code smells
  * Bugs
  * Security vulnerabilities

✔️ Quality Gate:

* ❌ Fail → Pipeline stops
* ✔️ Pass → Continue

#

### 5. Build Docker Image

* Create container image from Dockerfile

✔️ Output:

* Versioned Docker image\
  Example:

  ```
  myapp:v1.0.3
  ```

#

### 6. Security Scan (Image Level)

Using **Trivy**

* Scan Docker image for:

  * OS vulnerabilities
  * Package vulnerabilities

✔️ Decision:

* ❌ Critical issues → Fail pipeline
* ✔️ Clean → Continue

#

### 7. Push to Container Registry

* Push image to:

  * Docker Hub OR Private Registry

✔️ Output:

* Stored, versioned image

#

### 8. Update GitOps Repository

* Jenkins updates deployment manifest:

  * Changes image tag

Example:

```yaml
image: myapp:v1.0.3
```

✔️ This is the **handover point to CD**

---

## CI Final Output

- Verified code
- Tested application
- Secure Docker image
- Updated deployment configuration

---

## CI Summary Flow

```text
Git Push → Jenkins → Build → Test → SonarQube → Docker Build → Trivy Scan → Push Image → Update GitOps Repo
```

---

# 2. CD (Continuous Deployment) — SOP

## Objective

Automatically deploy validated application to Kubernetes using GitOps.

---

## CD Trigger

* Change detected in GitOps repository
* Handled by **Argo CD**

---

## CD Pipeline Flow

### 1. GitOps Sync Detection

* Argo CD continuously monitors Git repo
* Detects new commit (image update)

#

### 2. Compare Desired vs Actual State

* Desired state → Git repo (YAML/Helm)
* Actual state → Kubernetes cluster

#

### 3. Deployment Execution

Argo CD applies changes to Kubernetes:

* Update Deployment
* Pull new image from registry
* Rolling update starts

#

### 4. Kubernetes Handles Deployment

Inside cluster:

* Pull image
* Create new pods
* Gradually replace old pods

✔️ Ensures:

* Zero downtime (if configured)

#

### 5. Health Check & Sync Status

* Argo CD verifies:

  * Pods running
  * Services healthy

✔️ Status:

* Synced & Healthy → Success
* Degraded → Needs attention

---

### 6. Promotion Strategy (Multi-Cluster)

You have 3 clusters:

1. Dev

   * Auto deploy

2. Staging

   * Manual approval OR auto after dev success

3. Production

   * Strict control (manual trigger recommended)

---

## CD Final Output

- Application deployed
- Running in Kubernetes
- Version controlled via Git

---

## CD Summary Flow

```text
GitOps Repo Change → Argo CD Detect → Sync → Kubernetes Deploy → Pods Running
```

---

# 3. Full CI/CD Flow

```text
Developer
   ↓
Git Repo (App Code)
   ↓
CI Pipeline (Jenkins)
   ↓
Build + Test + Scan
   ↓
Docker Image
   ↓
Container Registry
   ↓
GitOps Repo Update
   ↓
CD Pipeline (Argo CD)
   ↓
Kubernetes Deployment
   ↓
Application Running
```

---

# 4. Responsibility Separation

| Stage    | Tool              | Responsibility     |
| -------- | ----------------- | ------------------ |
| CI       | Jenkins           | Build, test, scan  |
| QA       | SonarQube         | Code quality       |
| Security | Trivy             | Vulnerability scan |
| Registry | Docker Hub/Harbor | Store image        |
| CD       | Argo CD           | Deploy             |
| Runtime  | Kubernetes        | Run app            |

---

# 5. Key Concepts You Must Remember

1. CI ends when artifact is ready
2. CD starts when deployment begins
3. Git is the **single source of truth** (GitOps)
4. Jenkins does NOT deploy
5. Argo CD does NOT build

---

