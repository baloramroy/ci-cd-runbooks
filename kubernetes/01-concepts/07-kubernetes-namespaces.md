# Kubernetes Namespaces

## 1. Overview

A **Namespace** is a Kubernetes object that provides a logical scope for organizing and managing resources within a cluster.

Namespaces allow multiple teams, applications, or environments to use the same Kubernetes cluster while keeping their namespaced resources separate.

For example, a cluster might have the following namespaces:

```text
Kubernetes Cluster
│
├── default
│   ├── nginx-pod
│   └── web-service
│
├── development
│   ├── app-pod
│   └── app-service
│
├── production
│   ├── payment-pod
│   └── payment-service
│
└── kube-system
    ├── coredns
    └── kube-proxy
```

Each namespace provides a separate naming scope for namespaced resources.

Namespaces also support resource quotas, access control, and other policies that help manage workloads in a shared cluster.

**Important:** Namespaces provide logical isolation, not complete security isolation. Network policies and appropriate access controls may be needed to restrict communication and access between workloads.

---

## 2. Why Do We Need Namespaces?

Imagine a Kubernetes cluster shared by development, testing, and production teams.

Without namespaces, all namespaced resources would share the same namespace. This could make it difficult to organize resources and avoid naming conflicts.

For example, two teams might both want to create a Pod named `nginx`.

Namespaces allow both teams to use that name:

```text
Kubernetes Cluster
│
├── development
│   └── nginx
│
└── production
    └── nginx
```

Both Pods can exist because they belong to different namespaces.

Namespaces are useful for:

* Organizing resources by environment or team.
* Avoiding naming conflicts between resources.
* Applying resource quotas.
* Controlling access through Role-Based Access Control (RBAC).
* Applying namespace-scoped policies.

---

## 3. Default Kubernetes Namespaces

When Kubernetes initializes a cluster, it creates several namespaces.

You can view them using:

```bash
kubectl get namespaces
```

Or:

```bash
kubectl get ns
```

Typical output:

```text
NAME              STATUS   AGE
default           Active   10d
kube-node-lease   Active   10d
kube-public       Active   10d
kube-system       Active   10d
```

### 3.1 `default`

The `default` namespace is used when you create a namespaced resource without specifying a namespace.

For example:

```bash
kubectl run nginx --image=nginx:1.27
```

This creates the Pod in the current namespace. In a typical initial configuration, that is `default`.

You can explicitly specify it:

```bash
kubectl run nginx --image=nginx:1.27 -n default
```

### 3.2 `kube-system`

The `kube-system` namespace contains resources used by Kubernetes system components and cluster add-ons.

Examples may include:

* CoreDNS
* kube-proxy
* Other cluster infrastructure components

Inspect it with:

```bash
kubectl get pods -n kube-system
```

Avoid deleting or modifying system resources unless you understand their purpose.

### 3.3 `kube-public`

The `kube-public` namespace is readable by all users, including unauthenticated users, by default.

It is intended for resources that should be publicly readable across the cluster.

It is not commonly used for application workloads.

### 3.4 `kube-node-lease`

This namespace contains Lease objects used by Kubernetes nodes to communicate their availability through heartbeats.

These heartbeats help the control plane detect when a node may be unavailable.

---

## 4. Creating a Namespace

You can create a namespace using either `kubectl` or a YAML manifest.

### 4.1 Create Using kubectl

Create a namespace named `development`:

```bash
kubectl create namespace development
```

Expected output:

```text
namespace/development created
```

Verify:

```bash
kubectl get namespaces
```

### 4.2 Create Using a YAML Manifest

Create a file named `development-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```

Apply it:

```bash
kubectl apply -f development-namespace.yaml
```

Verify:

```bash
kubectl get namespace development
```

Using a manifest is useful when namespace configuration needs to be stored in Git and managed declaratively.

---

## 5. Deploying Resources into a Namespace

A namespace does not automatically contain application resources. You must create or deploy resources into it.

There are two common ways to specify the namespace.

### 5.1 Specify the Namespace in the Manifest

Create `nginx-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: development
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

Apply the manifest:

```bash
kubectl apply -f nginx-pod.yaml
```

The Pod will be created in the `development` namespace.

Verify:

```bash
kubectl get pods -n development
```

### 5.2 Specify the Namespace Using kubectl

You can also specify the namespace directly in the command:

```bash
kubectl run nginx-pod \
  --image=nginx:1.27 \
  -n development
```

Verify:

```bash
kubectl get pods -n development
```

The `-n` option is shorthand for `--namespace`.

**Note:** If a manifest explicitly specifies a namespace, that namespace takes precedence over the namespace selected by the command-line context.

---

## 6. Viewing Resources in Namespaces

By default, `kubectl get` displays resources from the current namespace.

For example:

```bash
kubectl get pods
```

To view resources in a particular namespace:

```bash
kubectl get pods -n development
```

To view Pods across all namespaces:

```bash
kubectl get pods --all-namespaces
```

The shorthand is:

```bash
kubectl get pods -A
```

You can also list other resource types across all namespaces:

```bash
kubectl get deployments -A
```

```bash
kubectl get services -A
```

To view multiple resource types in one command:

```bash
kubectl get pods,services,deployments -n development
```

This is useful when troubleshooting applications deployed in a specific namespace.

---

## 7. Namespaced vs. Cluster-Scoped Resources

Not every Kubernetes object belongs to a namespace.

Kubernetes resources are generally classified as either **namespaced** or **cluster-scoped**.

### 7.1 Namespaced Resources

Namespaced resources exist within a particular namespace.

Examples include:

* Pods
* Deployments
* ReplicaSets
* Services
* ConfigMaps
* Secrets
* PersistentVolumeClaims
* Jobs

For example, two Deployments can have the same name if they belong to different namespaces.

```text
development
└── nginx-deployment

production
└── nginx-deployment
```

### 7.2 Cluster-Scoped Resources

Cluster-scoped resources exist at the cluster level and do not belong to a namespace.

Examples include:

* Nodes
* Namespaces
* PersistentVolumes
* StorageClasses
* ClusterRoles
* ClusterRoleBindings

For example, a Node is part of the cluster, not a particular application namespace.

```text
Kubernetes Cluster
│
├── Node 1
├── Node 2
│
├── development
│   └── nginx-pod
│
└── production
    └── payment-pod
```

You cannot use `-n` to move a cluster-scoped resource into a namespace.

### 7.3 How to Check Resource Scope

Use:

```bash
kubectl api-resources
```

The output includes a `NAMESPACED` column.

Example:

```text
NAME          SHORTNAMES   APIVERSION   NAMESPACED   KIND
pods          po           v1           true         Pod
services      svc          v1           true         Service
nodes         no           v1           false        Node
namespaces    ns           v1           false        Namespace
```

Here:

* `true` means the resource is namespaced.
* `false` means the resource is cluster-scoped.

You can filter the output for a particular resource:

```bash
kubectl api-resources --namespaced=true
```

Or:

```bash
kubectl api-resources --namespaced=false
```

---

## 8. Switching the Current Namespace

If you frequently work in a particular namespace, you can change the namespace associated with your current `kubectl` context.

For example:

```bash
kubectl config set-context --current --namespace=development
```

Now, commands such as:

```bash
kubectl get pods
```

will use the `development` namespace by default.

To check the current context:

```bash
kubectl config current-context
```

To inspect its namespace:

```bash
kubectl config view --minify
```

To return to the `default` namespace:

```bash
kubectl config set-context --current --namespace=default
```

**Important:** Changing the current namespace only changes the default namespace used by `kubectl`. It does not move existing resources or change their namespaces.

For commands where the target namespace matters, explicitly using `-n` can help prevent mistakes.

---

## 9. Namespace Isolation and Resource Management

Namespaces provide a foundation for separating workloads, but they do not automatically enforce all forms of isolation.

Additional Kubernetes features can help manage access and resource consumption.

### 9.1 ResourceQuota

A ResourceQuota limits the total amount of resources that can be consumed by objects in a namespace.

For example, an administrator can configure a namespace to limit:

* CPU requests and limits
* Memory requests and limits
* The number of Pods
* The number of Services

A simplified example:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-quota
  namespace: development
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 4Gi
```

This quota limits the namespace to 10 Pods and a total of 2 CPU cores and 4 GiB of requested memory.

The quota does not automatically assign resources to Pods. It enforces limits on the resources declared by objects.

### 9.2 Role-Based Access Control (RBAC)

RBAC controls what users and service accounts are allowed to do.

For example, a Role can grant a user permission to view Pods in the `development` namespace without granting access to the entire cluster.

Namespace-scoped access is useful when several teams share a cluster.

### 9.3 NetworkPolicy

A NetworkPolicy can control network traffic to and from Pods, provided the cluster's network plugin supports NetworkPolicy enforcement.

For example, policies can restrict which Pods are allowed to communicate with a database.

**Remember:** Namespaces organize resources. ResourceQuota, RBAC, and NetworkPolicy provide additional controls.

---

## 10. Hands-On: Create and Manage Namespaces

### Objective

Create separate namespaces, deploy Pods into them, and practice viewing resources by namespace.

### Step 1: Create Two Namespaces

```bash
kubectl create namespace development
kubectl create namespace production
```

Verify:

```bash
kubectl get namespaces
```

### Step 2: Create a Pod in Development

```bash
kubectl run nginx-dev \
  --image=nginx:1.27 \
  -n development
```

### Step 3: Create a Pod in Production

```bash
kubectl run nginx-prod \
  --image=nginx:1.27 \
  -n production
```

### Step 4: Verify Each Namespace

```bash
kubectl get pods -n development
```

```bash
kubectl get pods -n production
```

Each command should display the Pod in its respective namespace.

### Step 5: View Pods Across All Namespaces

```bash
kubectl get pods -A
```

You should see both Pods, along with any other Pods in your cluster.

### Step 6: Check Resource Scope

```bash
kubectl api-resources
```

Identify which of these resources are namespaced:

* Pods
* Deployments
* Services
* Nodes
* PersistentVolumes
* Namespaces

### Step 7: Delete the Test Resources

Delete the two namespaces:

```bash
kubectl delete namespace development
kubectl delete namespace production
```

Kubernetes will delete the namespaced resources contained in them as part of namespace termination.

Verify:

```bash
kubectl get namespaces
```

**Warning:** Deleting a namespace is destructive. It deletes the namespaced resources within it. Never delete a production namespace just to clean up an exercise.

---

## 11. Common Mistakes

### Mistake 1: Assuming All Resources Belong to Namespaces

Nodes and PersistentVolumes are cluster-scoped. They do not belong to application namespaces.

Check resource scope with:

```bash
kubectl api-resources
```

### Mistake 2: Forgetting the Namespace

Running:

```bash
kubectl get pods
```

only lists Pods in the current namespace.

If a Pod is missing, check other namespaces:

```bash
kubectl get pods -A
```

### Mistake 3: Assuming Namespaces Provide Complete Isolation

Namespaces alone do not prevent Pods in different namespaces from communicating over the network.

Use appropriate RBAC and NetworkPolicy configurations when isolation is required.

### Mistake 4: Deleting a Namespace to Remove One Pod

Deleting a namespace also deletes its other namespaced resources.

If you only want to delete a Pod, use:

```bash
kubectl delete pod nginx-pod -n development
```

---

## 12. Key Takeaways

| Concept                 | Description                                                                |
| ----------------------- | -------------------------------------------------------------------------- |
| Namespace               | Provides a logical scope for organizing resources                          |
| `default`               | Namespace used when no other namespace is specified in the current context |
| `kube-system`           | Contains Kubernetes system resources                                       |
| `-n`                    | Specifies the namespace for a command                                      |
| `-A`                    | Lists resources across all namespaces                                      |
| Namespaced resource     | Exists within a namespace                                                  |
| Cluster-scoped resource | Exists at the cluster level                                                |
| ResourceQuota           | Limits resource consumption within a namespace                             |
| RBAC                    | Controls access to Kubernetes resources                                    |
| NetworkPolicy           | Controls Pod network traffic when enforced by the network plugin           |

The central concept is:

**Namespaces organize namespaced resources within a Kubernetes cluster. They provide logical separation, while additional Kubernetes features enforce access, resource, and network policies.**

---

## 13. Practice Exercise

Before moving to the next article, make sure you can:

1. Create a namespace using `kubectl`.
2. Create a namespace using a YAML manifest.
3. Deploy a Pod into a specific namespace.
4. List Pods in one namespace.
5. List Pods across all namespaces.
6. Switch the current namespace.
7. Identify namespaced and cluster-scoped resources.
8. Explain why namespaces alone do not provide complete security isolation.

---

## 14. References

* [Kubernetes Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
* [Namespaces: Sharing a Cluster with Namespaces](https://kubernetes.io/docs/tasks/administer-cluster/namespaces/)
* [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
* [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
* [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
