# Kubernetes Object Names and IDs

## 1. Overview

Every Kubernetes object needs an identity so that Kubernetes users and the API server can distinguish it from other objects.

Kubernetes primarily uses two fields to identify an object:

* **Name (`metadata.name`):** A human-readable name used to refer to an object.
* **UID (`metadata.uid`):** A unique identifier assigned by Kubernetes to distinguish an object from other objects, including objects that existed previously.

For example, you might create a Pod named `nginx-pod`. You can use that name with `kubectl` to inspect or delete it.

However, if you delete the Pod and create another Pod with the same name, Kubernetes assigns the new Pod a different UID.

Understanding this distinction is important when managing Kubernetes resources and troubleshooting applications.

---

## 2. Object Names

The `metadata.name` field specifies the name of a Kubernetes object.

Example:

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

Here:

```yaml
metadata:
  name: nginx-pod
```

The Pod's name is `nginx-pod`.

You can use this name to interact with the Pod.

For example, retrieve the Pod:

```bash
kubectl get pod nginx-pod
```

Inspect it:

```bash
kubectl describe pod nginx-pod
```

Delete it:

```bash
kubectl delete pod nginx-pod
```

Names make Kubernetes objects easy to identify and manage.

### 2.1 Naming Rules

Kubernetes object names must follow the naming rules applicable to their resource type.

Many resources use DNS-compatible naming conventions. For example, a Pod name generally must:

* Contain lowercase letters, numbers, hyphens, or periods, where permitted.
* Start and end with an alphanumeric character.
* Follow the length limit for its resource type.

Examples of valid Pod names:

```text
nginx
nginx-pod
web-server
app-01
frontend.production
```

Examples of invalid Pod names:

```text
Nginx
web_server
-my-pod
my-pod-
```

The exact naming rules can vary by resource type, so always check the relevant Kubernetes API documentation when necessary.

---

## 3. Uniqueness of Object Names

An object's name must be unique within its namespace for a given resource type.

For example, you can create two Pods with different names in the same namespace:

```text
Namespace: default

nginx-pod
web-pod
```

However, you cannot have two Pods named `nginx-pod` in the same namespace at the same time.

You can, however, have objects with the same name in different namespaces.

For example:

```text
Namespace: development
    └── nginx-pod

Namespace: production
    └── nginx-pod
```

Both Pods can exist because they belong to different namespaces.

The full reference to a namespaced object is commonly expressed as:

```text
<namespace>/<name>
```

For example:

```text
development/nginx-pod
production/nginx-pod
```

This distinction becomes particularly useful when managing applications across multiple environments.

---

## 4. Object UID

The `metadata.uid` field contains a unique identifier assigned to an object by Kubernetes.

Unlike a name, a UID is not chosen by the user.

Example:

```yaml
metadata:
  name: nginx-pod
  uid: 7c2b1e9a-1234-4567-89ab-0123456789ab
```

The UID distinguishes that particular object from other objects.

For example, consider this sequence:

1. Create a Pod named `nginx-pod`.
2. Kubernetes assigns it a UID.
3. Delete the Pod.
4. Create another Pod with the same name.

The new Pod receives a different UID.

Although both Pods have the same name, they are different Kubernetes objects.

### 4.1 Why Does Kubernetes Need UIDs?

Names can be reused after an object is deleted.

This creates a potential ambiguity.

Imagine a controller or another system is tracking a Pod named `nginx-pod`.

The original Pod is deleted, and a new Pod is created with the same name.

If the system relies only on the name, it may not be able to distinguish the original Pod from the replacement.

The UID solves this problem by providing an identity for each object instance.

---

## 5. Name vs. UID

| Feature                                                | Name (`metadata.name`)                                   | UID (`metadata.uid`)                 |
| ------------------------------------------------------ | -------------------------------------------------------- | ------------------------------------ |
| Purpose                                                | Human-readable object name                               | Unique identity of an object         |
| Assigned by                                            | User or configuration                                    | Kubernetes                           |
| Readability                                            | Easy to understand                                       | Usually a generated identifier       |
| Uniqueness                                             | Unique within the applicable namespace and resource type | Unique across objects in the cluster |
| Reusable                                               | Yes, after deletion                                      | No                                   |
| Changes when an object is recreated with the same name | Remains the same if specified again                      | A new UID is assigned                |
| Common use                                             | `kubectl` commands and references                        | Tracking a specific object instance  |

**Remember:**

* A name identifies an object within its namespace and resource type.
* A UID identifies a particular object instance.

---

## 6. Hands-On: Observe Names and UIDs

Let's create a Pod and examine its name and UID.

### Step 1: Create a Pod

Create a manifest named `nginx-pod.yaml`:

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

Apply the manifest:

```bash
kubectl apply -f nginx-pod.yaml
```

### Step 2: Check the Pod Name

Run:

```bash
kubectl get pods
```

Example output:

```text
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          10s
```

The Pod's name is `nginx-pod`.

### Step 3: Retrieve the UID

Run:

```bash
kubectl get pod nginx-pod -o jsonpath='{.metadata.uid}'
```

Example output:

```text
7c2b1e9a-1234-4567-89ab-0123456789ab
```

Your actual UID will be different.

You can also retrieve both the name and UID:

```bash
kubectl get pod nginx-pod -o custom-columns=NAME:.metadata.name,UID:.metadata.uid
```

Example output:

```text
NAME        UID
nginx-pod   7c2b1e9a-1234-4567-89ab-0123456789ab
```

### Step 4: Delete the Pod

```bash
kubectl delete pod nginx-pod
```

Verify that it has been deleted:

```bash
kubectl get pods
```

### Step 5: Recreate the Pod

Apply the same manifest again:

```bash
kubectl apply -f nginx-pod.yaml
```

Retrieve the new UID:

```bash
kubectl get pod nginx-pod -o jsonpath='{.metadata.uid}'
```

The new UID should be different from the original one, even though the Pod has the same name.

This demonstrates that Kubernetes treats the recreated Pod as a new object.

---

## 7. Object Identity in Practice

Understanding names and UIDs is useful in several Kubernetes operations.

### 7.1 Using Names with kubectl

Most everyday `kubectl` operations use object names.

For example:

```bash
kubectl get pod nginx-pod
kubectl describe pod nginx-pod
kubectl delete pod nginx-pod
```

When multiple namespaces are involved, specify the namespace:

```bash
kubectl get pod nginx-pod -n production
```

### 7.2 Tracking Object Recreation

Consider a Deployment that maintains three Pods.

If one Pod is deleted, the Deployment's controller can create a replacement.

The replacement may have the same generated name pattern, but it will have a different UID.

For example:

```text
Original Pod:
nginx-7c9d8f6b5c-abcde
UID: 1111-aaaa

Replacement Pod:
nginx-7c9d8f6b5c-fghij
UID: 2222-bbbb
```

The example illustrates that a replacement Pod is a new object, not the original Pod restored.

### 7.3 Relationship with Owners and Dependents

Kubernetes uses owner references to represent relationships between objects.

For example, a ReplicaSet can own Pods.

Owner references include the owner's UID, allowing Kubernetes to identify the specific owner object rather than relying only on its name.

You will study this relationship in more detail when learning Deployments, ReplicaSets, and garbage collection.

---

## 8. Key Takeaways

| Concept         | Description                                                    |
| --------------- | -------------------------------------------------------------- |
| `metadata.name` | Human-readable name of an object                               |
| `metadata.uid`  | Unique identifier assigned to an object                        |
| Namespace       | Provides a scope in which names are unique for a resource type |
| Name reuse      | A name can be reused after an object is deleted                |
| UID uniqueness  | A recreated object receives a new UID                          |
| Owner reference | Uses the owner's UID to identify the specific owner            |

The key principle is:

**A Kubernetes object's name is a convenient reference, while its UID identifies the specific object instance.**

---

## 9. Practice Exercise

Complete the following tasks in your Kubernetes lab:

1. Create a Pod named `web-server`.
2. Retrieve its name and UID.
3. Delete the Pod.
4. Recreate it using the same manifest.
5. Retrieve its name and UID again.
6. Compare the original and new UIDs.

Answer these questions:

* Did the Pod's name change?
* Did its UID change?
* Why does Kubernetes assign a new UID when the Pod is recreated?

Once you can explain these differences, you're ready to move on to **Labels and Selectors**, which Kubernetes uses to organize and identify groups of related objects.
