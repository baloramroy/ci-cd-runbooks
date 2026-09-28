# Kubernetes Objects and Desired State

## 1. Overview

Kubernetes manages applications through **Objects**.

    > A Kubernetes Object is a written record of something what you want Kubernetes to manage and maintain.

For example, instead of manually creating three containers and checking whether they are still running, you tell Kubernetes:

> "I want 3 instances of my application running."

Kubernetes then continuously works to make the actual cluster state match that desired state.

This is the foundation of the **declarative model** used throughout Kubernetes.

---

## 2. What Is a Kubernetes Object?

A Kubernetes Object represents a resource that Kubernetes manages.

Common Kubernetes Objects include:

* Pod
* Deployment
* Service
* ConfigMap
* Secret
* Namespace
* PersistentVolume
* PersistentVolumeClaim
* Job
* CronJob

For example:

```yaml
apiVersion: v1          # 1. Which API version to use
kind: Pod               # 2. What type of object this is
metadata:               # 3. Info about the object (name, labels)
  name: my-app
spec:                   # 4. What you want it to look like
  containers:
    - name: nginx
      image: nginx:1.27
```

This manifest tells Kubernetes:

> Create a Pod named `my-app` containing an `nginx:1.27` container.

The YAML file is not the running application itself. It is a **declaration of the desired state**.

---

## 3. Desired State vs Actual State

Kubernetes continuously compares two states.

### Desired State

The state you declare through Kubernetes configuration.

Example:

```text
3 application Pods should be running.
```

### Actual State

The state that currently exists in the cluster.

Example:

```text
Pod 1 → Running
Pod 2 → Running
Pod 3 → Failed
```

Kubernetes detects that the actual state does not match the desired state.

A controller then takes action to correct it.

```text

   +----------------+   compare   +----------------+
   |  Desired State | <---------> |  Actual State  |
   |    "3 Pods"    |             | "2 Running,    |
   +----------------+             |  1 Failed"     |
           ^                      +----------------+
           |                              |
           +------ Controller acts -------+
                (create replacement Pod)

```

The process of continuously bringing actual state toward desired state is called **reconciliation**.

---

## 4. Why This Model Matters

Without Kubernetes, an administrator might have to manually:

```text
Check application
    ↓
Find failed container
    ↓
Start replacement
    ↓
Check again
```

With Kubernetes, you declare:

```text
I want 3 replicas.
```

Kubernetes handles the ongoing reconciliation.

If one Pod disappears:

```text
Desired: 3 Pods
Actual:  2 Pods
```

The appropriate controller can create another Pod:

```text
Desired: 3 Pods
Actual:  3 Pods
```

This is one of the key reasons Kubernetes can manage large numbers of workloads consistently.

---

## 5. Imperative vs Declarative Configuration

Kubernetes primarily uses a **declarative approach**.

### Imperative approach

You tell the system **what action to perform**.

For example:

```bash
kubectl run nginx --image=nginx
```

The command directly requests an action:

```text
Create an nginx Pod
```

### Declarative approach

You describe **the state you want**.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

Then:

```bash
kubectl apply -f nginx.yaml
```

You are telling Kubernetes:

```text
This is the state I want.
Make the cluster match it.
```

This declarative model is especially important in CI/CD and GitOps because Kubernetes manifests can be stored in Git and applied consistently.

---

## 6. Basic Kubernetes Object Structure

Most Kubernetes manifests follow a common structure.

```yaml
apiVersion: ...
kind: ...
metadata:
  ...
spec:
  ...
```

The four fields you should understand first are:

| Field        | Purpose                                                 |
| ------------ | ------------------------------------------------------- |
| `apiVersion` | Defines which Kubernetes API version handles the object |
| `kind`       | Defines the type of object                              |
| `metadata`   | Identifies and describes the object                     |
| `spec`       | Defines the desired configuration                       |

---

## 7. `apiVersion`

Example:

```yaml
apiVersion: v1
```

or:

```yaml
apiVersion: apps/v1
```

`apiVersion` tells Kubernetes which API group and version should be used to interpret the object.

For example:

```yaml
apiVersion: v1
kind: Pod
```

A Deployment uses:

```yaml
apiVersion: apps/v1
kind: Deployment
```

The API version depends on the resource type.

---

## 8. `kind`

`kind` specifies what type of Kubernetes Object you are defining.

Example:

```yaml
kind: Pod
```

means:

```text
This object is a Pod.
```

Another example:

```yaml
kind: Deployment
```

means:

```text
This object is a Deployment.
```

The combination of:

```yaml
apiVersion:
kind:
```

tells Kubernetes how to interpret the manifest.

---

## 9. `metadata`

`metadata` contains information that identifies the object.

Example:

```yaml
metadata:
  name: my-app
```

Here:

```text
name = my-app
```

is the object's name.

Later, `metadata` will also contain important fields such as:

```yaml
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
```

We will study **namespaces, labels, and selectors** separately.

---

## 10. `spec`

`spec` describes the **desired configuration** of the object.

Example:

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

Here we are declaring:

```text
The Pod should contain an nginx container
using the nginx:1.27 image.
```

For different Kubernetes resources, `spec` contains different configuration.

For example:

```text
Pod
 └── spec
      └── containers

Deployment
 └── spec
      ├── replicas
      └── template

Service
 └── spec
      ├── selector
      └── ports
```

You will learn these fields when we study each resource.

---

## 11. What About `status`?

You will often see Kubernetes objects containing a `status` section.

For example:

```yaml
status:
  phase: Running
```

The important distinction is:

```text
spec   → What you want
status → What Kubernetes currently observes
```

Conceptually:

```text
          Object
             |
       +-----+-----+
       |           |
      spec       status
       |           |
       v           v
Desired State   Actual State
```

You normally define `spec` in your manifest.

Kubernetes and its controllers update `status` based on the current condition of the resource.

---

## 12. Complete Simple Example

Let's create a small Pod object.

Create:

```text
nginx-pod.yaml
```

with:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-pod

spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

The important structure is:

```text
apiVersion
    ↓
Which API?

kind
    ↓
What object?

metadata
    ↓
Which object?

spec
    ↓
What state do we want?
```

---

## 13. Hands-On: Create the Object

Verify the cluster is available:

```bash
kubectl get nodes
```

Create `nginx-pod.yaml` using the manifest from Section 4, then apply it:

```bash
kubectl apply -f nginx-pod.yaml
```

Expected output:

```text
pod/nginx-pod created
```

Check the Pod:

```bash
kubectl get pods
```

```text
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          10s
```

Inspect it in detail, and view the full YAML:

```bash
kubectl describe pod nginx-pod
kubectl get pod nginx-pod -o yaml
```

The YAML will include fields you did not write, such as `uid`, `resourceVersion`, and `creationTimestamp`. These are maintained by Kubernetes.

Now delete the Pod and check again:

```bash
kubectl delete pod nginx-pod
kubectl get pods
```

The Pod is gone and Kubernetes does not recreate it. **A standalone Pod has no controller maintaining a desired replica count.** When we learn Deployments, a controller will keep the desired number of Pods running and you will see reconciliation in action.
---

## 14. Kubernetes Object Workflow

The overall process can now be understood as:

```text
YAML Manifest
     |
     | kubectl apply
     v
Kubernetes API Server
     |
     v
Object stored in cluster
     |
     v
Controllers observe desired state
     |
     v
Controllers compare desired vs actual state
     |
     v
Take corrective action when necessary
     |
     v
Actual State approaches Desired State
```

This is the fundamental operating model behind many Kubernetes resources.

---


## 15. Key Points

Remember these concepts:

```text
Kubernetes Object
    ↓
A resource managed by Kubernetes

spec
    ↓
Desired State

status
    ↓
Observed/Current State

Declarative Configuration
    ↓
Describe the state you want

Controller
    ↓
Continuously reconciles actual state toward desired state
```

And the basic manifest structure:

```yaml
apiVersion: ...
kind: ...
metadata:
  ...
spec:
  ...
```

You do **not** need to memorize every field now.

The important thing at this stage is to understand:

> **Kubernetes manages objects by maintaining the desired state declared by the user and continuously reconciling the cluster toward that state.**

---

## 16. Practice Exercise

Before moving to the next topic, repeat the exercise without copying the example exactly.

Create a Pod named:

```text
web-server
```

using:

```text
nginx:1.27
```

Requirements:

1. Create the YAML manifest.
2. Apply it using `kubectl apply`.
3. Verify the Pod is `Running`.
4. Inspect it using `kubectl describe`.
5. View the complete YAML using `kubectl get -o yaml`.
6. Delete the Pod.
7. Verify that Kubernetes does not recreate it.

Once you understand this exercise, the next topic should be **Labels and Selectors**, because they are fundamental to how Deployments, ReplicaSets, and Services identify and connect workloads.
