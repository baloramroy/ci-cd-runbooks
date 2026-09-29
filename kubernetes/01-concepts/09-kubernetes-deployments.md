# Article 09 — Kubernetes Deployments

## 1. Objective

Understand how Kubernetes Deployments manage application workloads and why they are generally preferred over standalone Pods.

By the end of this article, you should be able to:

* Explain what a Deployment is and how it relates to Pods and ReplicaSets.
* Create and manage a Deployment using YAML and `kubectl`.
* Scale an application by changing its replica count.
* Perform and monitor rolling updates.
* Roll back a Deployment to a previous revision.
* Troubleshoot common Deployment issues.

## 2. Why Do We Need Deployments?

In the previous article, you learned how to create a standalone Pod.

A standalone Pod runs an application, but it does not provide a mechanism to maintain a desired number of replicas. If you delete it, Kubernetes does not automatically recreate it.

Consider an application running in a single Pod:

```text
Kubernetes Cluster
|
+-- Worker Node
    |
    +-- nginx-pod
        |
        +-- Nginx Container
```

If the Pod is deleted, the application stops running.

For production workloads, applications commonly need:

* Multiple replicas for availability and capacity.
* Automatic replacement of failed or deleted Pods.
* Controlled application updates.
* The ability to roll back an unsuccessful update.

A Kubernetes **Deployment** provides these capabilities by managing ReplicaSets, which in turn manage Pods.

## 3. What Is a Deployment?

A Deployment is a Kubernetes workload resource used to manage stateless applications.

You declare the desired state of your application, such as:

* The number of replicas.
* The container image.
* The Pod configuration.
* The update strategy.

Kubernetes continuously works to make the actual state match the desired state.

For example, if you configure three replicas, Kubernetes attempts to maintain three Pods.

```text
                 Deployment
                      |
                      v
                 ReplicaSet
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
        Pod 1       Pod 2       Pod 3
          |           |           |
          v           v           v
       Nginx        Nginx        Nginx
```

The Deployment does not directly create or manage individual Pods. It manages a ReplicaSet, and the ReplicaSet maintains the required number of Pods.

## 4. Deployment, ReplicaSet, and Pod Relationship

Understanding this relationship is important before working with Deployments.

| Resource   | Responsibility                                     |
| ---------- | -------------------------------------------------- |
| Deployment | Manages application rollout, scaling, and rollback |
| ReplicaSet | Ensures the required number of Pod replicas exists |
| Pod        | Runs the application's containers                  |

The hierarchy is:

```text
Deployment
    |
    +-- ReplicaSet
            |
            +-- Pod
            |
            +-- Pod
            |
            +-- Pod
```

When you update a Deployment's Pod template, Kubernetes generally creates a new ReplicaSet and gradually replaces the old Pods.

The old ReplicaSet is retained according to the Deployment's revision history settings, allowing a rollback when a previous revision is available.

You should normally manage the application through the Deployment rather than manually modifying its ReplicaSet or Pods.

## 5. Deployment Manifest Structure

A Deployment is defined using a YAML manifest.

Create a file:

```bash
vim nginx-deployment.yaml
```

Add the following configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
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

| Field                      | Purpose                                       |
| -------------------------- | --------------------------------------------- |
| `apiVersion`               | Specifies the Kubernetes API version          |
| `kind`                     | Defines the resource type                     |
| `metadata.name`            | Specifies the Deployment name                 |
| `spec.replicas`            | Defines the desired number of Pods            |
| `spec.selector`            | Identifies the Pods managed by the Deployment |
| `spec.template`            | Defines the Pod template used to create Pods  |
| `template.metadata.labels` | Assigns labels to the Pods                    |
| `template.spec.containers` | Defines the containers in each Pod            |
| `containers.image`         | Specifies the container image                 |

### Important: Selector and Pod Labels Must Match

The Deployment uses a selector to identify the Pods it manages.

In this example:

```yaml
selector:
  matchLabels:
    app: nginx
```

The Pod template contains:

```yaml
template:
  metadata:
    labels:
      app: nginx
```

These values must match. Otherwise, Kubernetes cannot correctly associate the Deployment with its Pods.

For an `apps/v1` Deployment, the selector is required and immutable after creation.

## 6. Create a Deployment

Apply the manifest:

```bash
kubectl apply -f nginx-deployment.yaml
```

Expected output:

```text
deployment.apps/nginx-deployment created
```

Check the Deployment:

```bash
kubectl get deployments
```

Example output:

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           30s
```

The columns indicate:

| Column       | Meaning                                                           |
| ------------ | ----------------------------------------------------------------- |
| `READY`      | Number of ready replicas compared with the desired count          |
| `UP-TO-DATE` | Number of replicas updated to the latest Deployment configuration |
| `AVAILABLE`  | Number of replicas available to serve requests                    |
| `AGE`        | How long the Deployment has existed                               |

Now check the Pods:

```bash
kubectl get pods -o wide
```

You should see three Pods created by the Deployment.

Their names will resemble:

```text
nginx-deployment-xxxxxxxxxx-xxxxx
```

The generated names are assigned by Kubernetes. Do not depend on specific generated names in scripts or configuration.

## 7. Understanding Desired State and Reconciliation

A Deployment declares the desired number of replicas.

For example:

```yaml
spec:
  replicas: 3
```

This means Kubernetes should maintain three replicas.

Suppose one Pod is deleted manually:

```bash
kubectl delete pod <pod-name>
```

The ReplicaSet detects that fewer Pods exist than desired and creates a replacement.

The process looks like this:

```text
Desired replicas: 3
Actual replicas:  3
        |
        v
One Pod is deleted
        |
        v
Actual replicas:  2
        |
        v
ReplicaSet detects the difference
        |
        v
Replacement Pod is created
        |
        v
Actual replicas:  3
```

This behavior is an example of Kubernetes reconciliation.

The Deployment and ReplicaSet continuously work toward the desired state rather than relying on a one-time command.

**Important:** A replacement Pod is a new Pod. It may have a different name and IP address.

## 8. Scaling a Deployment

Scaling means increasing or decreasing the number of application replicas.

For example, to increase the number of replicas from three to five:

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Verify:

```bash
kubectl get deployment nginx-deployment
```

Then check the Pods:

```bash
kubectl get pods
```

You should see five replicas once the scaling operation completes successfully.

### Scale Down

To reduce the replicas to two:

```bash
kubectl scale deployment nginx-deployment --replicas=2
```

Kubernetes terminates excess Pods until the desired replica count is reached.

### Scale Using YAML

You can also change the replica count in the manifest:

```yaml
spec:
  replicas: 5
```

Then apply the updated configuration:

```bash
kubectl apply -f nginx-deployment.yaml
```

For declarative workflows, keeping the desired replica count in version-controlled YAML helps ensure the configuration remains consistent.

**Note:** If you manually scale a Deployment but later apply a manifest that specifies a different replica count, the manifest's value becomes the desired count.

## 9. Deployment Rolling Updates

Applications often need to be updated with a new container image.

For example, you may need to update an Nginx Deployment from one image version to another.

A Deployment supports rolling updates, which gradually replace old Pods with new ones.

This can help maintain application availability during an update, provided that the application and cluster have sufficient capacity and the Pods become ready.

### 9.1 Update the Container Image

You can update the image using:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.30
```

Here:

* `nginx-deployment` is the Deployment name.
* `nginx` is the container name.
* `nginx:1.30` is the new image.

Check the rollout:

```bash
kubectl rollout status deployment/nginx-deployment
```

Example output:

```text
deployment "nginx-deployment" successfully rolled out
```

Check the Deployment:

```bash
kubectl get deployment nginx-deployment
```

Inspect the Pods:

```bash
kubectl get pods
```

Kubernetes gradually replaces the old Pods with Pods created from the updated template.

### 9.2 Rolling Update Strategy

A Deployment uses a rolling update strategy by default.

Two important settings control how many Pods can be unavailable or created above the desired replica count during an update.

| Setting          | Purpose                                                                               |
| ---------------- | ------------------------------------------------------------------------------------- |
| `maxUnavailable` | Maximum number of Pods that can be unavailable during the rollout                     |
| `maxSurge`       | Maximum number of additional Pods that can be created above the desired replica count |

For example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

Suppose the Deployment has three replicas.

With these settings, Kubernetes can create up to one additional Pod beyond the desired count and allow up to one Pod to be unavailable during the rollout.

The exact rollout sequence depends on Pod readiness, scheduling, resource availability, and other cluster conditions.

### 9.3 Configure the Update Strategy in YAML

The complete Deployment can include the strategy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.30
          ports:
            - containerPort: 80
```

Apply it:

```bash
kubectl apply -f nginx-deployment.yaml
```

The Deployment will reconcile the cluster to the updated configuration.

**Important:** A rolling update does not automatically guarantee zero downtime. Application readiness, available capacity, and the ability to run old and new versions concurrently all matter.

## 10. Monitoring Deployment Rollouts

Kubernetes provides commands to inspect rollout progress and history.

### Check Rollout Status

```bash
kubectl rollout status deployment/nginx-deployment
```

This waits for the Deployment rollout to complete or report a problem.

### View Rollout History

```bash
kubectl rollout history deployment/nginx-deployment
```

Example:

```text
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

The revision history tracks changes to the Deployment's Pod template.

By default, Kubernetes retains a limited number of old ReplicaSets for rollback purposes. The retention limit can be configured with `revisionHistoryLimit`.

### Inspect a Specific Revision

```bash
kubectl rollout history deployment/nginx-deployment --revision=1
```

This displays details of the specified revision, if it is still available.

The `CHANGE-CAUSE` field may be empty unless you record a cause or use a workflow that populates it. Do not rely on it as a complete audit trail.

## 11. Rolling Back a Deployment

Sometimes an application update introduces a problem.

For example:

* The new image fails to start.
* The application has a configuration error.
* The new version does not behave as expected.

You can roll back to a previous Deployment revision.

### 11.1 Roll Back to the Previous Revision

```bash
kubectl rollout undo deployment/nginx-deployment
```

Check the rollout:

```bash
kubectl rollout status deployment/nginx-deployment
```

Then inspect the image:

```bash
kubectl get deployment nginx-deployment \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

The image should reflect the restored Pod template.

### 11.2 Roll Back to a Specific Revision

First, inspect the history:

```bash
kubectl rollout history deployment/nginx-deployment
```

Then roll back to a specific revision:

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

Verify the rollout:

```bash
kubectl rollout status deployment/nginx-deployment
```

A rollback is itself a new change to the Deployment's current state. It does not erase the history of what happened.

**Important:** Rollback restores the previous Pod template. It does not automatically restore external state, such as database changes or data stored outside the Deployment.

## 12. Deployment Resource Management

A Deployment can also specify resource requests and limits for its containers.

For example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

* **Requests** describe the resources Kubernetes should account for when scheduling a container.
* **Limits** define resource ceilings enforced by Kubernetes and the container runtime.

Resource requests and limits are important for predictable scheduling and resource control.

They are especially relevant when running multiple replicas or deploying workloads to nodes with limited capacity.

Detailed resource management will be covered in a later article.

## 13. Hands-On Lab: Deployment Management

In this lab, you will create a Deployment, scale it, update its image, and practice rollback.

### Step 1: Create a Working Directory

```bash
mkdir -p ~/kubernetes-lab/deployments
cd ~/kubernetes-lab/deployments
```

### Step 2: Create the Deployment Manifest

```bash
vim nginx-deployment.yaml
```

Add:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.29
          ports:
            - containerPort: 80
```

### Step 3: Create the Deployment

```bash
kubectl apply -f nginx-deployment.yaml
```

Verify:

```bash
kubectl get deployment nginx-deployment
kubectl get replicasets
kubectl get pods -o wide
```

Confirm that the Deployment has three ready replicas.

### Step 4: Test Automatic Pod Replacement

List the Pods:

```bash
kubectl get pods
```

Choose one Pod and delete it:

```bash
kubectl delete pod <pod-name>
```

Immediately check the Pods:

```bash
kubectl get pods
```

You may briefly see the old Pod terminating and a replacement Pod being created.

Wait until the Deployment returns to three ready replicas:

```bash
kubectl get deployment nginx-deployment
```

This confirms that the ReplicaSet is maintaining the desired replica count.

### Step 5: Scale the Deployment

Increase the replicas to five:

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Verify:

```bash
kubectl get deployment nginx-deployment
kubectl get pods
```

Confirm that five replicas become ready.

Now scale down to two:

```bash
kubectl scale deployment nginx-deployment --replicas=2
```

Verify that the Deployment converges to two ready replicas.

### Step 6: Update the Container Image

Update the Deployment to use Nginx 1.30:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.30
```

Monitor the rollout:

```bash
kubectl rollout status deployment/nginx-deployment
```

Inspect the Deployment's image:

```bash
kubectl get deployment nginx-deployment \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Expected output:

```text
nginx:1.30
```

### Step 7: Inspect the Rollout History

```bash
kubectl rollout history deployment/nginx-deployment
```

Identify the revisions created during your exercise.

### Step 8: Roll Back

Roll back to the previous revision:

```bash
kubectl rollout undo deployment/nginx-deployment
```

Wait for completion:

```bash
kubectl rollout status deployment/nginx-deployment
```

Check the image again:

```bash
kubectl get deployment nginx-deployment \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

The image should return to the previous version, `nginx:1.29`, assuming that was the last revision before the update.

### Step 9: Restore the Manifest's Desired State

Your manifest still specifies three replicas and the original image.

Apply it:

```bash
kubectl apply -f nginx-deployment.yaml
```

Verify:

```bash
kubectl get deployment nginx-deployment
kubectl get pods
```

The Deployment should converge to the configuration in the manifest.

### Step 10: Clean Up

Delete the Deployment:

```bash
kubectl delete -f nginx-deployment.yaml
```

Verify that its Pods are removed:

```bash
kubectl get deployments
kubectl get pods
```

## 14. Common Deployment Troubleshooting Scenarios

| Symptom                                          | Possible cause                                                    | What to check                                           |
| ------------------------------------------------ | ----------------------------------------------------------------- | ------------------------------------------------------- |
| Deployment has fewer ready replicas than desired | Pods are pending, failing, or not ready                           | `kubectl get pods`, `kubectl describe pod`              |
| `ImagePullBackOff`                               | Image unavailable or registry access failure                      | Pod Events and image configuration                      |
| `CrashLoopBackOff`                               | Application container repeatedly exits                            | Container logs and Pod Events                           |
| Rollout does not complete                        | New Pods are not becoming ready or cannot be scheduled            | `kubectl rollout status`, `kubectl describe deployment` |
| Old Pods remain during rollout                   | New Pods may not be ready, or rollout is constrained              | Deployment status and Pod readiness                     |
| Deployment has unexpected replica count          | Another configuration or scaling action changed the desired state | `kubectl get deployment -o yaml`                        |
| Rollback is unavailable                          | Previous revision is no longer retained                           | `kubectl rollout history`                               |

Useful troubleshooting commands:

```bash
kubectl get deployment nginx-deployment
kubectl describe deployment nginx-deployment
kubectl get replicasets
kubectl get pods -o wide
kubectl rollout status deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment
```

For a specific Pod:

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

Start with the Deployment status, then inspect its ReplicaSets and Pods to identify where the problem is occurring.

## 15. Deployments in CI/CD

Deployments are a key part of Kubernetes-based Continuous Delivery.

A typical workflow is:

```text
Developer
    |
    v
Git Repository
    |
    v
Jenkins
    |
    +-- Checkout Source Code
    |
    +-- Build Application
    |
    +-- Run Tests
    |
    +-- Build Container Image
    |
    +-- Scan Image
    |
    +-- Push Image to Registry
    |
    v
Update Deployment Configuration
    |
    v
Kubernetes Deployment
    |
    v
ReplicaSet
    |
    v
Application Pods
```

In a GitOps workflow, Jenkins may update the image tag in a Git repository, and Argo CD reconciles the cluster with the updated configuration.

The Deployment then performs the rollout.

This separation allows:

* CI to build, test, scan, and publish images.
* Git to store the desired deployment configuration.
* Argo CD to synchronize the configuration with Kubernetes.
* Kubernetes to manage the application rollout.

For production, use immutable image tags or digests rather than relying on mutable tags such as `latest`. This makes it easier to identify exactly which image version is deployed and to reproduce a release.

## 16. Key Takeaways

* A Deployment manages stateless application workloads.
* A Deployment manages ReplicaSets, which maintain the desired number of Pods.
* A Deployment can replace deleted Pods through its ReplicaSet.
* Scaling changes the desired replica count.
* Rolling updates gradually replace old Pods with new ones.
* `maxUnavailable` and `maxSurge` control the rollout's availability and extra capacity.
* Rollout history helps track Deployment revisions.
* Rollbacks restore a previous Pod template when the revision is available.
* A successful rollout does not by itself guarantee that the application is functioning correctly.
* In CI/CD, a Deployment is commonly the Kubernetes resource updated to release a new application image.

## 17. References

* [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* [Kubernetes ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
* [Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)
* [Rolling Back a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)
* [kubectl rollout](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)

---

**Next Article:** `10-kubernetes-services.md` — Kubernetes Services
