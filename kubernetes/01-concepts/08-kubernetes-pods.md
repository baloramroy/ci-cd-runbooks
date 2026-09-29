# Article 08 — Kubernetes Pods

## 1. Objective

Understand how Kubernetes runs applications using Pods, how Pods interact with containers, and how to create, inspect, and manage them.

By the end of this article, you should be able to:

* Explain what a Pod is and why Kubernetes uses it.
* Understand the relationship between Pods and containers.
* Create and inspect a Pod using `kubectl` and YAML.
* Understand Pod phases, networking, and restart policies.
* Recognize why standalone Pods are not suitable for managing production workloads.

## 2. What Is a Pod?

A **Pod** is the smallest deployable unit in Kubernetes. It represents one or more containers that Kubernetes schedules and manages together.

Although containers run the application, Kubernetes schedules Pods onto nodes rather than scheduling individual containers directly.

A Pod typically contains one application container. However, it can also contain multiple containers that need to work closely together.

For example:

```text
Kubernetes Cluster
|
+-- Worker Node
    |
    +-- Pod
        |
        +-- Application Container
```

A Pod provides a shared environment for its containers, including:

* Network namespace and Pod IP address
* Shared storage volumes, when configured
* A common lifecycle and scheduling location

### Pod vs. Container

| Container                                               | Pod                                              |
| ------------------------------------------------------- | ------------------------------------------------ |
| Runs an application or process                          | Provides the environment in which containers run |
| Can run independently of Kubernetes                     | Is a Kubernetes workload unit                    |
| Has its own isolated filesystem and process environment | Can contain one or more containers               |
| Does not inherently have a Kubernetes Pod IP            | Has a Pod IP when networking is configured       |
| Can be restarted by a container runtime                 | Is managed by Kubernetes                         |

A Pod is not a replacement for a container. It is a Kubernetes abstraction that groups containers and provides them with shared resources.

## 3. Why Does Kubernetes Use Pods?

Kubernetes needs a way to manage applications that may consist of multiple closely related processes.

For example, an application container might need a helper container to collect logs or provide supporting functionality.

Instead of scheduling these containers independently, Kubernetes can place them in the same Pod.

Containers in the same Pod:

* Are scheduled together on the same node.
* Share the Pod's network namespace.
* Can communicate through `localhost`.
* Can share mounted volumes when configured.

This is useful for tightly coupled containers, such as an application and a sidecar container.

However, most applications should start with one container per Pod. Multiple containers are appropriate when the containers have a close operational relationship.

## 4. Pod Architecture

A Pod can contain one or more containers.

### Single-Container Pod

This is the most common arrangement.

```text
+----------------------------------+
|              Pod                 |
|                                  |
|  +----------------------------+  |
|  |   Application Container    |  |
|  |                            |  |
|  |   Nginx                    |  |
|  +----------------------------+  |
|                                  |
|  Pod IP: 10.244.1.10             |
+----------------------------------+
```

The Pod has its own IP address. The container uses the Pod's network namespace.

### Multi-Container Pod

A multi-container Pod may contain an application container and a sidecar container.

```text
+---------------------------------------+
|                  Pod                  |
|                                       |
|  +----------------+  +-------------+ |
|  | Application    |  | Sidecar     | |
|  | Container      |  | Container   | |
|  |                |  |             | |
|  +----------------+  +-------------+ |
|                                       |
|  Shared network namespace             |
|  Shared volumes (if configured)       |
+---------------------------------------+
```

For example, a sidecar might collect logs from a shared volume and forward them to a logging system.

Both containers run in the same Pod and can communicate through `localhost`.

A multi-container Pod is not the same as two independent Pods. Its containers share the Pod's network identity and are scheduled together.

## 5. Pod Manifest Structure

Kubernetes Pods are commonly defined using YAML manifests.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.29
      ports:
        - containerPort: 80
```

### Important Fields

| Field                 | Purpose                                             |
| --------------------- | --------------------------------------------------- |
| `apiVersion`          | Specifies the Kubernetes API version                |
| `kind`                | Defines the resource type                           |
| `metadata.name`       | Specifies the Pod's name                            |
| `metadata.labels`     | Adds labels to the Pod                              |
| `spec`                | Defines the desired configuration                   |
| `spec.containers`     | Lists the containers in the Pod                     |
| `name`                | Specifies the container's name                      |
| `image`               | Specifies the container image                       |
| `ports.containerPort` | Documents the port the container is expected to use |

### Important: `containerPort` Does Not Expose a Pod

The `containerPort` field documents the port a container uses. It does not publish that port outside the Pod or make the application accessible from outside the cluster.

To expose an application to other Pods or external clients, Kubernetes uses networking resources such as Services and, where applicable, Ingress.

We will cover those in later articles.

## 6. Creating a Pod

There are two common ways to create a Pod.

### 6.1 Imperative Method

You can create a Pod directly using `kubectl run`.

```bash
kubectl run nginx-pod --image=nginx:1.29
```

Verify that the Pod exists:

```bash
kubectl get pods
```

Example output:

```text
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          10s
```

The Pod may initially show `Pending` or `ContainerCreating`. Give Kubernetes time to schedule the Pod and start its container.

### 6.2 Declarative Method

Create a manifest:

```bash
vim nginx-pod.yaml
```

Add the following configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.29
      ports:
        - containerPort: 80
```

Apply the manifest:

```bash
kubectl apply -f nginx-pod.yaml
```

Verify:

```bash
kubectl get pod nginx-pod
```

The declarative approach keeps the desired configuration in a file. This makes it easier to review, version-control, and reuse the configuration in a CI/CD or GitOps workflow.

For the rest of this article, use the declarative method in your hands-on exercise.

## 7. Understanding Pod Status and Phases

A Pod moves through different phases during its lifecycle.

Check the phase using:

```bash
kubectl get pods
```

For more details:

```bash
kubectl get pod nginx-pod -o wide
```

The common Pod phases are:

| Phase       | Meaning                                                                                                                              |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `Pending`   | The Pod has been accepted, but its containers have not all been started. It may be waiting for scheduling or image downloads.        |
| `Running`   | The Pod has been bound to a node, and its containers have been created. At least one container is running or starting or restarting. |
| `Succeeded` | All containers have terminated successfully and will not be restarted.                                                               |
| `Failed`    | All containers have terminated, and at least one terminated unsuccessfully.                                                          |
| `Unknown`   | Kubernetes cannot determine the Pod's state, usually because it cannot communicate with the node.                                    |

A Pod's phase is not the same as its readiness.

For example, a Pod can be in the `Running` phase while its application is not ready to receive traffic. Readiness probes help Kubernetes determine whether an application is ready to serve requests. We will cover probes in a later article.

## 8. Inspecting a Pod

Kubernetes provides several commands to inspect a Pod.

### 8.1 Get Pod Details

```bash
kubectl get pod nginx-pod -o wide
```

This displays additional information, including the Pod IP and the node on which the Pod is running.

Example:

```text
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE
nginx-pod   1/1     Running   0          2m    10.244.1.10  worker-01
```

The actual IP and node depend on your cluster.

### 8.2 Describe a Pod

```bash
kubectl describe pod nginx-pod
```

This command provides detailed information, including:

* Pod metadata and labels
* Node assignment
* Pod IP
* Container image and state
* Restart count
* Events related to scheduling and container startup

The Events section is particularly useful when a Pod fails to start.

For example, an image-pull problem may appear as an event indicating that Kubernetes cannot pull the container image.

### 8.3 View Container Logs

```bash
kubectl logs nginx-pod
```

This displays the logs from the Pod's container.

For a multi-container Pod, specify the container name:

```bash
kubectl logs <pod-name> -c <container-name>
```

Logs are useful for troubleshooting application startup failures and runtime errors.

### 8.4 Execute a Command Inside a Container

You can execute commands inside a running container using `kubectl exec`.

For example:

```bash
kubectl exec -it nginx-pod -- /bin/sh
```

Inside the container, run:

```bash
hostname
```

You can also check the Nginx response locally:

```bash
curl http://localhost
```

If `curl` is not installed in the container, use another available HTTP client or perform the test from a separate Pod.

Exit the container:

```bash
exit
```

`kubectl exec` is useful for troubleshooting, but it should not be used as the normal way to configure or maintain applications. Changes made manually inside a container may disappear when the container is replaced.

## 9. Pod Networking

Every Pod receives its own IP address from the cluster's Pod network.

Containers inside the same Pod share the network namespace. They can communicate with each other using `localhost`.

For example, if a Pod contains two containers:

```text
Pod IP: 10.244.1.10
|
+-- Application Container
|       |
|       +-- Listens on port 8080
|
+-- Sidecar Container
        |
        +-- Connects to localhost:8080
```

The sidecar can reach the application using:

```text
localhost:8080
```

However, containers in the same Pod must avoid conflicting port usage because they share the same network namespace.

Pods are also able to communicate with other Pods through the cluster network, subject to the cluster's network configuration and any applicable network policies.

**Important:** A Pod IP is not a permanent address. When a Pod is replaced, the replacement may receive a different IP. This is one reason applications typically use Services for stable network access rather than relying directly on Pod IP addresses.

## 10. Pod Restart Policy

A Pod's `restartPolicy` determines how Kubernetes handles terminated containers within that Pod.

The supported values are:

| Policy      | Behavior                                                                   |
| ----------- | -------------------------------------------------------------------------- |
| `Always`    | Restart a container whenever it terminates, regardless of its exit status. |
| `OnFailure` | Restart a container when it terminates unsuccessfully.                     |
| `Never`     | Do not restart a terminated container.                                     |

The default restart policy is `Always`.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  restartPolicy: Always
  containers:
    - name: nginx
      image: nginx:1.29
```

The restart policy applies to containers within the Pod. It does not mean Kubernetes will automatically recreate a deleted standalone Pod.

For example, if you delete this Pod:

```bash
kubectl delete pod nginx-pod
```

Kubernetes will not recreate it simply because its restart policy is `Always`.

A workload controller, such as a Deployment, is responsible for maintaining the desired number of Pods and creating replacements when necessary.

## 11. Pod Lifecycle and Ephemeral Nature

Pods are designed to be replaceable.

A Pod may be terminated or replaced for several reasons, including:

* A user deleting it.
* A node becoming unavailable.
* A workload controller replacing it.
* A change in the workload configuration that requires replacement.

When a Pod is replaced, the new Pod is a different Kubernetes object and may have a different UID and IP address.

This means you should not treat a Pod as a permanent server.

For example:

```text
Deployment
    |
    +-- Pod A (10.244.1.10)
            |
            +-- Container
```

If Pod A is deleted, a Deployment can create a replacement:

```text
Deployment
    |
    +-- Pod B (10.244.2.15)
            |
            +-- Container
```

Pod B is a new Pod. It may have a different name and IP address, depending on how the workload is managed.

This is why production applications are usually managed through controllers such as Deployments rather than as standalone Pods.

## 12. Hands-On Lab: Create and Manage a Pod

In this exercise, you will create a standalone Nginx Pod, inspect it, test the application, and observe what happens when you delete it.

### Step 1: Create a Working Directory

```bash
mkdir -p ~/kubernetes-lab/pods
cd ~/kubernetes-lab/pods
```

### Step 2: Create the Manifest

```bash
vim nginx-pod.yaml
```

Add:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: lab
spec:
  containers:
    - name: nginx
      image: nginx:1.29
      ports:
        - containerPort: 80
```

Save the file.

### Step 3: Create the Pod

```bash
kubectl apply -f nginx-pod.yaml
```

Expected output:

```text
pod/nginx-pod created
```

### Step 4: Verify the Pod

```bash
kubectl get pods -o wide
```

Wait until the Pod reaches the `Running` phase.

Check its details:

```bash
kubectl describe pod nginx-pod
```

Review the container state, Pod IP, node assignment, and Events section.

### Step 5: Check the Application

Execute a command inside the container:

```bash
kubectl exec nginx-pod -- nginx -v
```

Expected output will show the installed Nginx version.

You can also test the web server from inside the container:

```bash
kubectl exec nginx-pod -- /bin/sh -c 'wget -qO- http://localhost'
```

If successful, the command returns the default Nginx HTML page.

### Step 6: View the Logs

```bash
kubectl logs nginx-pod
```

You may see little or no output if no HTTP requests have been made. Nginx access logs are typically generated when requests arrive.

### Step 7: Delete the Pod

```bash
kubectl delete pod nginx-pod
```

Verify that it has been deleted:

```bash
kubectl get pods
```

The Pod should no longer appear.

### Step 8: Observe Standalone Pod Behavior

Wait a few seconds and run:

```bash
kubectl get pods
```

The Pod will not be recreated automatically.

This demonstrates the difference between a standalone Pod and a Pod managed by a controller.

### Step 9: Clean Up

If the Pod still exists, delete it:

```bash
kubectl delete -f nginx-pod.yaml --ignore-not-found
```

Your manifest remains available for future practice.

## 13. Common Pod Troubleshooting Scenarios

| Status or symptom                      | Possible cause                                   | What to check                                         |
| -------------------------------------- | ------------------------------------------------ | ----------------------------------------------------- |
| `Pending`                              | Scheduling constraints or insufficient resources | `kubectl describe pod`                                |
| `ImagePullBackOff`                     | Image does not exist or registry access failed   | Image name, registry credentials, Events              |
| `ErrImagePull`                         | Kubernetes could not pull the image              | Pod Events and image configuration                    |
| `CrashLoopBackOff`                     | Container repeatedly exits and is restarted      | `kubectl logs` and `kubectl describe pod`             |
| `Running`, but application unavailable | Application may not be listening or ready        | Container logs, application configuration, networking |
| `Unknown`                              | Kubernetes cannot determine the Pod's state      | Node status and cluster connectivity                  |

When troubleshooting, start with:

```bash
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

These commands provide a useful initial picture of the Pod's state and its container behavior.

## 14. Pods in CI/CD

In a CI/CD workflow, Kubernetes ultimately runs application containers inside Pods.

A simplified deployment flow looks like this:

```text
Developer
    |
    v
Git Repository
    |
    v
Jenkins Pipeline
    |
    +-- Build Application
    |
    +-- Build Container Image
    |
    +-- Scan Image
    |
    +-- Push Image to Registry
    |
    v
Kubernetes Deployment
    |
    v
Pod
    |
    v
Application Container
```

Jenkins can build and publish a container image, while a deployment process updates the Kubernetes workload configuration.

Kubernetes then creates or replaces Pods to match the desired state.

In a GitOps workflow, a tool such as Argo CD can apply the desired configuration from Git to the cluster.

You will learn how Deployments manage Pods and perform rolling updates in the next article.

## 15. Key Takeaways

* A Pod is the smallest deployable workload unit in Kubernetes.
* A Pod contains one or more containers that are scheduled and managed together.
* Containers in the same Pod share a network namespace and can communicate through `localhost`.
* Each Pod has its own IP address, but that IP is not permanent.
* `containerPort` documents a container port; it does not expose the application externally.
* Pod phases describe lifecycle state, not necessarily application readiness.
* A standalone Pod is not automatically recreated when deleted.
* Controllers such as Deployments manage Pod replicas and replacements.
* Production applications are generally managed through workload controllers rather than standalone Pods.

## 16. References

* [Kubernetes: Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
* [Kubernetes: Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
* [Kubernetes: Running a Pod](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/)
* [Kubernetes: kubectl Reference](https://kubernetes.io/docs/reference/kubectl/)

---

**Next Article:** `09-kubernetes-deployments.md` — Kubernetes Deployments
