# Kubernetes Object Management

## 1. Overview

Kubernetes Object Management is the process of creating, updating, inspecting, and deleting resources in a Kubernetes cluster.

The primary command-line tool for managing Kubernetes objects is **`kubectl`**.

Using `kubectl`, you can interact with the Kubernetes API server to manage resources such as Pods, Deployments, Services, ConfigMaps, and Secrets.

There are two main approaches to object management:

* **Imperative management:** Specify the action you want Kubernetes to perform.
* **Declarative management:** Define the desired state in a configuration file and let Kubernetes reconcile the object with that configuration.

For most application deployments and CI/CD workflows, declarative management is the preferred approach.

---

## 2. Prerequisites

Before starting, ensure that:

* A Kubernetes cluster is running.
* `kubectl` is installed.
* `kubectl` is configured to communicate with your cluster.
* You have permission to create and manage the resources used in this lesson.

Verify cluster connectivity:

```bash
kubectl cluster-info
```

Check the nodes:

```bash
kubectl get nodes
```

If these commands return cluster information, `kubectl` can communicate with your cluster.

---

## 3. Imperative vs. Declarative Management

Kubernetes supports both imperative and declarative object management.

### 3.1 Imperative Management

In imperative management, you tell Kubernetes **what action to perform**.

For example:

```bash
kubectl run nginx --image=nginx:1.27
```

This command instructs Kubernetes to create a Pod named `nginx` using the specified image.

Another example:

```bash
kubectl delete pod nginx
```

This directly requests the deletion of the Pod.

Imperative commands are useful for quick operations, troubleshooting, and experimentation.

### 3.2 Declarative Management

In declarative management, you define **the desired state** of an object in a YAML or JSON manifest.

For example, create a file named `nginx-pod.yaml`:

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

Apply the configuration:

```bash
kubectl apply -f nginx-pod.yaml
```

Kubernetes creates or updates the object to match the configuration.

The manifest becomes the source of truth for the configuration you manage declaratively.

### 3.3 Comparison

| Feature           | Imperative                       | Declarative                           |
| ----------------- | -------------------------------- | ------------------------------------- |
| Approach          | Specify an action                | Define the desired state              |
| Configuration     | Usually command arguments        | YAML or JSON manifests                |
| Typical command   | `kubectl run`                    | `kubectl apply`                       |
| Updates           | Issue another command            | Update the manifest and apply it      |
| Reusability       | Commands can be saved in scripts | Manifests can be version-controlled   |
| CI/CD suitability | Useful for specific operations   | Well suited to repeatable deployments |

**Important:** Imperative commands can create resources, while declarative management is particularly useful for maintaining consistent configurations over time.

---

## 4. Creating Kubernetes Objects

There are several ways to create Kubernetes objects.

### 4.1 Create an Object Imperatively

Create a Pod directly:

```bash
kubectl run nginx --image=nginx:1.27
```

Verify:

```bash
kubectl get pods
```

Example output:

```text
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          10s
```

This creates a Pod without requiring a YAML manifest.

However, the command alone does not provide a reusable configuration file describing the entire object.

### 4.2 Create an Object Declaratively

Create a manifest:

```bash
vi nginx-pod.yaml
```

Add:

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

Create the object:

```bash
kubectl apply -f nginx-pod.yaml
```

Example output:

```text
pod/nginx-pod created
```

Verify:

```bash
kubectl get pod nginx-pod
```

Example output:

```text
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          10s
```

The manifest can now be reused to create the same configuration in another suitable environment.

---

## 5. Inspecting Kubernetes Objects

After creating an object, you need to inspect it to understand its current state.

### 5.1 `kubectl get`

The `kubectl get` command retrieves information about Kubernetes objects.

List all Pods in the current namespace:

```bash
kubectl get pods
```

List all Deployments:

```bash
kubectl get deployments
```

List Services:

```bash
kubectl get services
```

To see more details about Pods, use:

```bash
kubectl get pods -o wide
```

This includes additional information such as the Pod IP and the node where the Pod is running.

To retrieve the complete object configuration in YAML:

```bash
kubectl get pod nginx-pod -o yaml
```

This output includes fields maintained by Kubernetes, such as `status`, `uid`, and `resourceVersion`.

### 5.2 `kubectl describe`

The `kubectl describe` command provides detailed information about an object, including its configuration, current status, and related events.

Example:

```bash
kubectl describe pod nginx-pod
```

The output includes information such as:

* Pod name and namespace
* Labels
* Node assignment
* Container details
* Container state
* Events

This command is particularly useful when troubleshooting a Pod that is not starting or behaving as expected.

**Difference between `get` and `describe`:**

| Command               | Purpose                                          |
| --------------------- | ------------------------------------------------ |
| `kubectl get`         | Displays object information in a concise format  |
| `kubectl get -o yaml` | Displays the complete object in YAML             |
| `kubectl describe`    | Displays detailed information and related events |

---

## 6. Updating Kubernetes Objects

Objects often need to be updated when application configurations change.

For example, you may need to change a container image from `nginx:1.27` to `nginx:1.28`.

### 6.1 Update Using a Manifest

Open the existing manifest:

```bash
vi nginx-pod.yaml
```

Change the image:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.28
```

Apply the updated configuration:

```bash
kubectl apply -f nginx-pod.yaml
```

Kubernetes will attempt to apply the change.

**Important:** A running Pod has several immutable fields. Changing its container image through `kubectl apply` may fail because the Pod cannot be updated in place. For this standalone Pod, you can delete and recreate it:

```bash
kubectl delete -f nginx-pod.yaml
kubectl apply -f nginx-pod.yaml
```

In real application deployments, you will generally use a Deployment to manage image updates and rolling replacements.

### 6.2 Update Using `kubectl edit`

You can also edit an existing object directly:

```bash
kubectl edit pod nginx-pod
```

This opens the object's configuration in your configured text editor.

You can make changes and save the file. Kubernetes then attempts to apply the updated configuration.

However, not every field can be changed. For example, changing a standalone Pod's container image in place is generally not allowed.

`kubectl edit` is useful for troubleshooting and quick changes, but direct edits are not automatically reflected in your original YAML file.

For consistent, repeatable management, keep your desired configuration in version-controlled manifests.

---

## 7. Deleting Kubernetes Objects

You can delete objects using either their names or their manifest files.

### 7.1 Delete by Object Name

Delete the Pod:

```bash
kubectl delete pod nginx-pod
```

Example output:

```text
pod "nginx-pod" deleted
```

Verify:

```bash
kubectl get pods
```

The deleted Pod should no longer appear.

### 7.2 Delete Using a Manifest

You can also delete an object using the YAML file that defines it:

```bash
kubectl delete -f nginx-pod.yaml
```

This is useful when managing resources through manifests.

**Note:** Deleting a standalone Pod removes that Pod. It does not create a replacement. A controller such as a Deployment can create replacement Pods when necessary to maintain its desired replica count.

---

## 8. Understanding `kubectl apply`

`kubectl apply` is one of the most important commands for declarative Kubernetes management.

```bash
kubectl apply -f nginx-pod.yaml
```

It can be used to create an object if it does not exist or update an existing object based on the supplied configuration.

For example:

```text
First apply
    |
    v
Object does not exist
    |
    v
Create object
    |
    v
Update manifest
    |
    v
Apply again
    |
    v
Update object where possible
```

You can also apply all supported manifests in a directory:

```bash
kubectl apply -f ./manifests/
```

This is useful when an application consists of multiple Kubernetes resources.

For example:

```text
manifests/
├── deployment.yaml
├── service.yaml
└── configmap.yaml
```

Applying the directory submits these manifests to Kubernetes.

The order in which resources are processed should not be treated as a guarantee of application readiness. For example, creating a Deployment does not mean its Pods are already running.

---

## 9. Previewing Changes Before Applying

Before applying a manifest, you can preview the changes Kubernetes would make.

```bash
kubectl diff -f nginx-pod.yaml
```

This compares the configuration in the manifest with the live object.

If there are differences, the command displays them.

For example, if the image changes from `nginx:1.27` to `nginx:1.28`, the output can show that change.

This is useful in CI/CD pipelines because it allows you to inspect configuration differences before applying them.

Note that `kubectl diff` requires the appropriate permissions and may return a nonzero exit code when differences are found.

---

## 10. Object Management Workflow

A typical declarative object management workflow looks like this:

```text
        YAML Manifest
              |
              v
       kubectl diff
              |
              v
       Review Changes
              |
              v
       kubectl apply
              |
              v
       Kubernetes API
              |
              v
       Object Created
       or Updated
              |
              v
       kubectl get
              |
              v
       Verify Status
              |
              v
       kubectl describe
       (when more detail is needed)
```

In production environments, the manifest should generally be stored in version control. Changes can then be reviewed and applied through an approved deployment process.

---

## 11. Practical Exercise

Use your Kubernetes lab to practice the object management workflow.

### Objective

Create a Pod, inspect it, update its configuration, and delete it.

### Step 1: Create a Manifest

Create a file named `web-server.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

### Step 2: Preview the Configuration

```bash
kubectl diff -f web-server.yaml
```

If the object does not exist, there may be no difference to display.

### Step 3: Create the Pod

```bash
kubectl apply -f web-server.yaml
```

### Step 4: Verify the Pod

```bash
kubectl get pod web-server -o wide
```

Then inspect it:

```bash
kubectl describe pod web-server
```

### Step 5: Retrieve the Object YAML

```bash
kubectl get pod web-server -o yaml
```

Compare the returned configuration with your original manifest.

Identify the fields that Kubernetes added or populated.

### Step 6: Update the Manifest

Change the image in `web-server.yaml` to:

```yaml
image: nginx:1.28
```

Preview the change:

```bash
kubectl diff -f web-server.yaml
```

Because this is a standalone Pod, delete and recreate it to apply the image change:

```bash
kubectl delete -f web-server.yaml
kubectl apply -f web-server.yaml
```

Verify the new image:

```bash
kubectl get pod web-server -o jsonpath='{.spec.containers[0].image}'
```

Expected output:

```text
nginx:1.28
```

### Step 7: Delete the Pod

```bash
kubectl delete -f web-server.yaml
```

Verify that it has been removed:

```bash
kubectl get pods
```

---

## 12. Key Takeaways

| Concept                | Description                                               |
| ---------------------- | --------------------------------------------------------- |
| Imperative management  | Tells Kubernetes which action to perform                  |
| Declarative management | Defines the desired state in a manifest                   |
| `kubectl apply`        | Creates or updates objects from configuration             |
| `kubectl get`          | Retrieves object information                              |
| `kubectl describe`     | Displays detailed object information and events           |
| `kubectl edit`         | Edits a live object                                       |
| `kubectl diff`         | Previews differences between a manifest and a live object |
| `kubectl delete`       | Deletes Kubernetes objects                                |

The most important principle is to **manage application configuration declaratively whenever practical**. This makes changes easier to review, reproduce, and integrate into CI/CD workflows.

---

## 13. References

* [Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/)
* [Object Management](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/)
* [kubectl Reference](https://kubernetes.io/docs/reference/kubectl/)
* [Declarative Object Management](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)

**Learning checkpoint:** Complete the practical exercise and make sure you understand the difference between `kubectl apply`, `kubectl get`, and `kubectl describe`. The next article will be **Kubernetes Object Names and IDs**.
