# Article 10 — Kubernetes Services

## 1. Objective

Understand how Kubernetes Services provide stable network access to applications running inside Pods.

By the end of this article, you should be able to:

* Explain why Kubernetes needs Services.
* Understand how Services use labels and selectors to identify Pods.
* Understand the different Service types: ClusterIP, NodePort, LoadBalancer, and ExternalName.
* Create and manage Services using YAML and `kubectl`.
* Verify Service connectivity and troubleshoot common networking problems.
* Understand how Services fit into a Kubernetes-based CI/CD workflow.

## 2. Why Do We Need Services?

In the previous articles, you learned how to create Pods and manage them through Deployments.

However, Pods are temporary resources. When a Pod is deleted or replaced, its IP address can change.

Consider a Deployment with three Nginx replicas:

```text
Deployment
    |
    v
ReplicaSet
    |
    +-- Pod A: 10.244.1.10
    |
    +-- Pod B: 10.244.2.15
    |
    +-- Pod C: 10.244.3.20
```

Suppose another application needs to communicate with these Pods.

If it connects directly to a Pod IP, it must know which Pod is available. When a Pod is replaced, the application may need to discover its new IP.

This creates several problems:

* Pod IP addresses are not permanent.
* The number of Pods may change when a Deployment scales.
* Applications need a reliable way to discover their backend Pods.
* Traffic needs to be distributed across eligible replicas.

A Kubernetes **Service** solves this problem by providing a stable network endpoint for a group of Pods.

Instead of connecting directly to individual Pods, clients connect to the Service.

```text
                  Client
                    |
                    v
              Kubernetes Service
                    |
          +---------+---------+
          |         |         |
          v         v         v
        Pod A     Pod B     Pod C
```

The Service provides a stable way to reach the selected Pods, even when individual Pods are replaced.

## 3. What Is a Kubernetes Service?

A Kubernetes Service is an abstraction that exposes a group of Pods over the network.

A Service typically uses a label selector to identify the Pods that should receive traffic.

For example, a Service with the selector:

```yaml
selector:
  app: nginx
```

selects Pods that have the label:

```yaml
labels:
  app: nginx
```

Kubernetes tracks the eligible backend Pods and updates the Service's endpoints as Pods become eligible or are removed.

A Service provides:

* A stable virtual IP address for applicable Service types.
* A stable DNS name within the cluster.
* A way to discover backend Pods.
* Traffic distribution across eligible endpoints.

A Service does not create Pods. It provides network access to existing workloads.

## 4. How a Service Works

Consider a Deployment with three Nginx Pods.

Each Pod has the label `app: nginx`.

```text
                 Deployment
                      |
                      v
                 ReplicaSet
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
        Pod A       Pod B       Pod C
        app=nginx   app=nginx   app=nginx
          |           |           |
          +-----------+-----------+
                      |
                      v
                Nginx Service
                Selector:
                app=nginx
                      |
                      v
                  Client
```

The Service selects the Pods based on their labels.

When a client sends traffic to the Service, Kubernetes networking directs it to an eligible backend Pod.

If one Pod is deleted and replaced, Kubernetes updates the Service's backend endpoints as the replacement becomes eligible.

The client can continue using the same Service address.

### Important Components

| Component                                   | Responsibility                                                       |
| ------------------------------------------- | -------------------------------------------------------------------- |
| Service                                     | Provides a stable network endpoint                                   |
| Selector                                    | Identifies the Pods that belong to the Service                       |
| EndpointSlice                               | Tracks the network endpoints associated with the Service             |
| Pod                                         | Runs the application receiving traffic                               |
| kube-proxy or an alternative implementation | Implements Service traffic forwarding in many cluster configurations |

Modern Kubernetes uses EndpointSlices to represent backend endpoints. A Service controller maintains these resources based on the Service configuration and the eligible Pods.

The exact traffic-forwarding implementation depends on the cluster's networking configuration. Many kubeadm clusters use kube-proxy, but some use a CNI implementation that provides equivalent functionality.

## 5. Service Manifest Structure

A Service can be defined using YAML.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### Important Fields

| Field              | Purpose                                              |
| ------------------ | ---------------------------------------------------- |
| `apiVersion`       | Specifies the Kubernetes API version                 |
| `kind`             | Defines the resource type                            |
| `metadata.name`    | Specifies the Service name                           |
| `spec.type`        | Defines how the Service is exposed                   |
| `spec.selector`    | Identifies the backend Pods                          |
| `ports.protocol`   | Specifies the network protocol                       |
| `ports.port`       | The port exposed by the Service                      |
| `ports.targetPort` | The port on the backend Pod to which traffic is sent |

### Understanding `port` and `targetPort`

These two fields are important.

For example:

```yaml
ports:
  - port: 8080
    targetPort: 80
```

Here:

* `port: 8080` is the port clients use on the Service.
* `targetPort: 80` is the port on the selected Pod.

The traffic flow is:

```text
Client
  |
  | Connects to Service port 8080
  v
Service
  |
  | Forwards traffic to targetPort 80
  v
Pod
  |
  v
Application listening on port 80
```

The Service port and the Pod's application port do not have to be the same.

## 6. Service Types

Kubernetes provides four main Service types:

1. ClusterIP
2. NodePort
3. LoadBalancer
4. ExternalName

Each type serves a different networking purpose.

### 6.1 ClusterIP

`ClusterIP` is the default Service type.

It exposes an application through an internal virtual IP address that is generally reachable from within the cluster.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Architecture:

```text
              Kubernetes Cluster
        +-----------------------------+
        |                             |
        |  Client Pod                 |
        |       |                     |
        |       v                     |
        |  ClusterIP Service          |
        |       |                     |
        |       v                     |
        |  +----+----+                |
        |  |         |                |
        |  v         v                |
        | Pod A     Pod B             |
        |                             |
        +-----------------------------+
```

Use ClusterIP when an application needs to be accessed internally by other workloads in the cluster.

Typical examples include:

* Backend APIs
* Internal microservices
* Database services
* Internal application components

A ClusterIP Service is not directly exposed to external clients by default.

### 6.2 NodePort

`NodePort` exposes a Service on a port of each eligible node's IP address.

For example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Architecture:

```text
          External Client
                 |
                 v
       Worker Node IP:30080
                 |
                 v
           NodePort Service
                 |
          +------+------+
          |             |
          v             v
        Pod A         Pod B
```

You can access the Service using a node's IP address and the assigned NodePort, provided that network routing and firewall rules allow access.

For example:

```text
http://192.168.0.32:30080
```

The IP address above is only an example. Use an address assigned to one of your cluster nodes.

The default NodePort range is `30000–32767`, although the range can be configured on the API server.

NodePort is useful for learning and certain infrastructure setups. In production, external traffic is often handled through a load balancer or an ingress gateway instead.

### 6.3 LoadBalancer

`LoadBalancer` requests an external load balancer for a Service.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Architecture:

```text
             External Client
                    |
                    v
            External Load Balancer
                    |
                    v
             Kubernetes Service
                    |
             +------+------+
             |             |
             v             v
           Pod A         Pod B
```

In a cloud environment, a cloud integration can provision the external load balancer.

In a local kubeadm lab, creating a LoadBalancer Service does not automatically provide an external load balancer. You need an appropriate implementation, such as MetalLB, or another supported load-balancing solution.

Without one, the Service may remain in a pending state for its external IP.

### 6.4 ExternalName

`ExternalName` maps a Kubernetes Service name to an external DNS name.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  type: ExternalName
  externalName: api.example.com
```

A client inside the cluster can use the Kubernetes Service DNS name, and DNS returns a CNAME pointing to the external hostname.

Unlike the other Service types, ExternalName does not select Pods or create a virtual IP for forwarding traffic.

It is useful when applications should refer to an external service through a Kubernetes DNS name.

## 7. Service DNS

Kubernetes provides DNS-based service discovery.

A Service can normally be reached by its name from within the same namespace.

For example, a client Pod in the same namespace can access:

```text
http://nginx-service
```

A Service in another namespace can be addressed using its namespace:

```text
http://nginx-service.production
```

The fully qualified Service DNS name follows this format:

```text
<service-name>.<namespace>.svc.cluster.local
```

For example:

```text
nginx-service.default.svc.cluster.local
```

The cluster domain is configurable. `cluster.local` is the common default, but you should not assume every cluster uses it.

DNS allows applications to use stable service names rather than depending on changing Pod IP addresses.

## 8. Service Selectors and Labels

Services commonly use label selectors to find their backend Pods.

Consider a Pod with these labels:

```yaml
metadata:
  labels:
    app: nginx
    environment: production
```

A Service with this selector:

```yaml
selector:
  app: nginx
```

selects the Pod because it has the `app=nginx` label.

A Service with this selector:

```yaml
selector:
  app: nginx
  environment: production
```

also selects the Pod because it matches both labels.

A Service with this selector:

```yaml
selector:
  app: apache
```

does not select the Pod.

### Important: Matching Labels Are Essential

If a Service selector does not match any Pods, the Service can exist but have no backend endpoints.

For example:

```bash
kubectl get svc nginx-service
```

may show the Service as active even though no Pods are selected.

Check its endpoints using:

```bash
kubectl get endpointslice \
  -l kubernetes.io/service-name=nginx-service
```

You can also inspect the Service:

```bash
kubectl describe service nginx-service
```

If there are no eligible Pods, verify the Service selector and the Pod labels.

## 9. Creating a ClusterIP Service

Let's create a Deployment and expose it using a ClusterIP Service.

### Step 1: Create a Deployment

Create a file:

```bash
vim nginx-deployment.yaml
```

Add:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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

Apply it:

```bash
kubectl apply -f nginx-deployment.yaml
```

Verify that the Pods are ready:

```bash
kubectl get pods -l app=nginx
```

### Step 2: Create the Service

Create a file:

```bash
vim nginx-service.yaml
```

Add:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f nginx-service.yaml
```

### Step 3: Verify the Service

```bash
kubectl get service nginx-service
```

Example output:

```text
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.96.120.10    <none>        80/TCP    10s
```

The ClusterIP shown is an example. Your cluster will assign its own address.

### Step 4: Inspect the Service

```bash
kubectl describe service nginx-service
```

Check:

* Service type
* ClusterIP
* Selector
* Port and targetPort
* Endpoints

The endpoints should contain the IP addresses and ports of the eligible backend Pods.

### Step 5: Test the Service from Another Pod

Create a temporary client Pod:

```bash
kubectl run curl-client \
  --image=curlimages/curl:8.12.1 \
  --restart=Never \
  --command -- sleep 3600
```

Wait for it to become ready:

```bash
kubectl get pod curl-client
```

Test the Service:

```bash
kubectl exec curl-client -- \
  curl -sS http://nginx-service
```

The response should contain the default Nginx HTML page.

This confirms that the client Pod can access the Nginx application through the Service DNS name.

## 10. Testing Service Load Distribution

A Service can distribute traffic across eligible backend Pods.

To observe this, we can make requests to the Service and inspect the backend response.

However, the default Nginx welcome page does not identify which Pod handled a request.

For a simple test, you can temporarily use a container that returns its hostname, or inspect the access logs of the backend Pods.

For now, verify that the Service has multiple endpoints:

```bash
kubectl get endpointslice \
  -l kubernetes.io/service-name=nginx-service \
  -o wide
```

You should see the IP addresses of the eligible Pods.

The Service's traffic distribution is implemented by the cluster's networking components. It does not guarantee an equal number of requests to each Pod.

Factors such as connection reuse, session affinity, and the networking implementation can affect observed traffic distribution.

## 11. Creating a NodePort Service

A NodePort Service allows access through a node's IP address and a specific port.

Create a manifest:

```bash
vim nginx-nodeport.yaml
```

Add:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

Apply it:

```bash
kubectl apply -f nginx-nodeport.yaml
```

Verify:

```bash
kubectl get service nginx-nodeport
```

Example output:

```text
NAME             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-nodeport   NodePort   10.96.130.20    <none>        80:30080/TCP   10s
```

You can test it from a machine that can reach the node:

```bash
curl http://<node-ip>:30080
```

Replace `<node-ip>` with the IP address of a cluster node.

If the request fails, check:

* Whether the Service has endpoints.
* Whether the node is reachable.
* Whether the NodePort is allowed by the host firewall.
* Whether the cluster's networking implementation is functioning correctly.

## 12. Updating a Service

You can modify a Service by updating its manifest and applying the changes.

For example, you can change the Service port:

```yaml
ports:
  - protocol: TCP
    port: 8080
    targetPort: 80
```

Apply the updated manifest:

```bash
kubectl apply -f nginx-service.yaml
```

The Service will now accept traffic on port `8080` and forward it to port `80` on the backend Pods.

Verify:

```bash
kubectl get service nginx-service
```

You can also inspect the full configuration:

```bash
kubectl get service nginx-service -o yaml
```

Some Service fields are immutable or have restrictions on updates. For example, changing the Service type or ClusterIP has specific constraints. Consult the Kubernetes API documentation before modifying these fields in production.

## 13. Service and Deployment Relationship

A Deployment and a Service serve different purposes.

A Deployment manages the application Pods. A Service provides stable network access to eligible Pods.

```text
              Deployment
                  |
                  v
              ReplicaSet
                  |
          +-------+-------+
          |       |       |
          v       v       v
        Pod A   Pod B   Pod C
          |       |       |
          +-------+-------+
                  |
                  v
              Service
                  |
                  v
                Client
```

The Service does not depend on the generated Pod names. It uses labels to identify eligible Pods.

When a Deployment replaces a Pod, the Service can continue routing traffic to the remaining eligible Pods and later include the replacement when it becomes eligible.

This separation makes it possible to scale or replace Pods without changing the address that clients use.

## 14. Hands-On Lab: Deployment and Service

In this lab, you will create a three-replica Deployment, expose it through a ClusterIP Service, test connectivity, and observe how the Service responds when Pods are replaced.

### Step 1: Create a Working Directory

```bash
mkdir -p ~/kubernetes-lab/services
cd ~/kubernetes-lab/services
```

### Step 2: Create the Deployment Manifest

Create `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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

Apply it:

```bash
kubectl apply -f nginx-deployment.yaml
```

Wait for the Deployment to become ready:

```bash
kubectl rollout status deployment/nginx-deployment
```

### Step 3: Create the Service Manifest

Create `nginx-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f nginx-service.yaml
```

### Step 4: Verify the Service

```bash
kubectl get svc nginx-service
kubectl describe svc nginx-service
```

Confirm that the Service has a ClusterIP and that its selector is `app=nginx`.

### Step 5: Verify the Backend Endpoints

```bash
kubectl get endpointslice \
  -l kubernetes.io/service-name=nginx-service
```

Confirm that the EndpointSlices contain the addresses of the backend Pods.

### Step 6: Create a Client Pod

```bash
kubectl run curl-client \
  --image=curlimages/curl:8.12.1 \
  --restart=Never \
  --command -- sleep 3600
```

Wait for it to become ready:

```bash
kubectl get pod curl-client
```

### Step 7: Test Service Connectivity

```bash
kubectl exec curl-client -- \
  curl -sS http://nginx-service
```

You should receive the Nginx welcome page.

Now test using the Service's fully qualified DNS name:

```bash
kubectl exec curl-client -- \
  curl -sS http://nginx-service.default.svc.cluster.local
```

This should return the same application page, assuming your cluster uses the default `cluster.local` DNS domain.

### Step 8: Delete One Backend Pod

List the Nginx Pods:

```bash
kubectl get pods -l app=nginx
```

Choose one Pod and delete it:

```bash
kubectl delete pod <pod-name>
```

Because the Pods are managed by a Deployment, the ReplicaSet will create a replacement.

Check the Pods:

```bash
kubectl get pods -l app=nginx
```

Wait until all three replicas are ready:

```bash
kubectl rollout status deployment/nginx-deployment
```

### Step 9: Verify the Service Again

Check the EndpointSlices:

```bash
kubectl get endpointslice \
  -l kubernetes.io/service-name=nginx-service
```

Test the Service again:

```bash
kubectl exec curl-client -- \
  curl -sS http://nginx-service
```

The Service name remains the same, even though one of the backend Pods has been replaced.

### Step 10: Clean Up

Delete the resources:

```bash
kubectl delete -f nginx-service.yaml
kubectl delete -f nginx-deployment.yaml
kubectl delete pod curl-client --ignore-not-found
```

Verify:

```bash
kubectl get svc
kubectl get deployments
kubectl get pods
```

## 15. Common Service Troubleshooting Scenarios

| Symptom                           | Possible cause                                       | What to check                                    |
| --------------------------------- | ---------------------------------------------------- | ------------------------------------------------ |
| Service exists, but traffic fails | No eligible backend Pods                             | Check EndpointSlices                             |
| Service has no endpoints          | Selector does not match Pod labels                   | Compare Service selector with Pod labels         |
| DNS name does not resolve         | DNS configuration or CoreDNS problem                 | Test DNS from a client Pod                       |
| ClusterIP is unreachable          | Cluster networking problem or incorrect port         | Check Service configuration and networking       |
| NodePort is unreachable           | Firewall, routing, or node networking issue          | Check node connectivity and firewall rules       |
| LoadBalancer has no external IP   | No load balancer implementation is available         | Check the cluster's load-balancer configuration  |
| Requests fail after a rollout     | New Pods are not ready or the application is failing | Check Deployment status, Pod readiness, and logs |

Useful commands:

```bash
kubectl get svc
kubectl describe svc <service-name>
kubectl get endpointslice
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

For DNS troubleshooting, run a lookup from a client Pod:

```bash
kubectl exec curl-client -- \
  nslookup nginx-service
```

If the client image does not contain `nslookup`, use an appropriate diagnostic image.


## 16. Key Takeaways

* A Service provides a stable network endpoint for a group of Pods.
* Services commonly use label selectors to identify backend Pods.
* EndpointSlices track the endpoints associated with a Service.
* `port` is the Service port; `targetPort` is the backend Pod port.
* ClusterIP provides internal access within the cluster.
* NodePort exposes a Service through a port on cluster nodes.
* LoadBalancer requests an external load balancer when a suitable implementation is available.
* ExternalName maps a Service name to an external DNS name.
* Kubernetes DNS allows applications to communicate using Service names.
* Services and Deployments have separate responsibilities: Deployments manage Pods, while Services provide network access to them.
* A Service's stable endpoint helps applications continue communicating when Pods are replaced.

## 18. References

* [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
* [Service Discovery](https://kubernetes.io/docs/concepts/services-networking/service/#discovering-services)
* [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
* [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
* [Debugging Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)

---

**Next Article:** `11-kubernetes-configmaps.md` — Kubernetes ConfigMaps
