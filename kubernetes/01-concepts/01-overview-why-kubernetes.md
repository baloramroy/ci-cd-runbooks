# Kubernetes Overview

Kubernetes is an **open-source platform for deploying, managing, and scaling containerized applications**.

Running a container is simple. Running hundreds or thousands of containers across multiple servers is much more difficult. You need to handle tasks such as:

* Starting and stopping containers
* Replacing failed containers
* Scaling applications
* Distributing workloads across servers
* Managing application configuration and secrets
* Providing networking and service discovery
* Updating applications without unnecessary downtime

Kubernetes provides a platform that automates many of these tasks.

Instead of manually managing individual containers, you define the **desired state** of your application, and Kubernetes continuously works to keep the actual environment in that state.

For example, you can tell Kubernetes:

> "I want 3 replicas of my application running."

If one Pod fails, Kubernetes detects the difference between the desired state and the actual state and takes action to restore the required number of replicas.

This makes Kubernetes useful for running **reliable, scalable, and distributed containerized applications**.

## Why Kubernetes?

Without Kubernetes, managing containers across multiple servers can become increasingly difficult as an application grows.

Kubernetes provides capabilities such as:

* **Self-healing:** Restarts or replaces failed workloads.
* **Scaling:** Increases or decreases application replicas.
* **Service discovery:** Allows applications to communicate through stable service names.
* **Load balancing:** Distributes traffic across application Pods.
* **Automated rollouts and rollbacks:** Helps deploy new application versions and return to previous versions when required.
* **Storage orchestration:** Connects workloads to persistent storage.
* **Configuration and secret management:** Manages application configuration and sensitive information.
* **Resource management:** Places workloads on nodes based on available resources and requirements.

## Kubernetes and Containers

Kubernetes does **not** replace containers or build container images.

A typical workflow looks like this:

```text
Application Source Code
        |
        v
   Build & Test
        |
        v
   Container Image
        |
        v
      Registry
        |
        v
    Kubernetes
        |
        v
   Running Pods
```

The container image is built using tools such as Docker or other container build tools. Kubernetes then manages the deployment and lifecycle of those containers through **Pods**.

In a CI/CD environment, a pipeline may build and scan an image, push it to a container registry, and then deploy that image to Kubernetes.

## Kubernetes Architecture

At a high level, a Kubernetes cluster has two primary architectural parts:

* **Control Plane:** Makes cluster-level decisions and manages the desired state of the cluster.
* **Worker Nodes:** Provide the environment where application workloads run.

Each architectural part contains different Kubernetes components that work together to operate the cluster.

Kubernetes also supports **Add-ons**, which provide additional functionality such as DNS, monitoring, and logging.

We will examine these components and how they communicate with each other in the next section.
