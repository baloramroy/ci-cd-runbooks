# What Is a "Manifest" in Kubernetes?

Great question — it's one of those words that gets thrown around constantly once you start working with Kubernetes, but rarely defined clearly.

## Short Answer

A **manifest** is just a **file that describes a Kubernetes Object** — usually written in **YAML** (sometimes JSON).

That's it. When someone says *"apply the manifest"*, they mean *"feed this YAML file to Kubernetes."*

---

## The Word Itself

"Manifest" comes from shipping and logistics. A **ship's manifest** is a document listing everything on board — what's being carried, how much, where it's going.

A **Kubernetes manifest** works the same way. It's a **declaration of what should exist** in your cluster:

> "Here is a list of the things I want you to run."

Kubernetes reads it and makes reality match it.

---

## What It Looks Like

This is a manifest:

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

That whole YAML block *is* the manifest. It's a plain text file, conventionally saved as something like:

```text
nginx-pod.yaml
```

And applied with:

```bash
kubectl apply -f nginx-pod.yaml
```

The `-f` flag stands for **file** — as in *"here's the file, read the manifest inside it."*

---

## A Manifest Can Contain Multiple Objects

A single manifest file can describe more than one object. You separate them with `---`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.27
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
```

One file → two objects created. Useful when things belong together.

---

## Manifest vs Object vs Resource

These three words get used almost interchangeably, but there's a subtle difference:

| Term | What it means | Example |
|------|---------------|---------|
| **Manifest** | The **file** you write | `nginx-pod.yaml` |
| **Object** | The **thing** Kubernetes stores and manages | The Pod named `nginx-pod` inside the cluster |
| **Resource** | The **type** of thing | "Pod" as a category, "Deployment" as a category |

Think of it like a form at a government office:

- The **form** you fill in → manifest
- The **record** the clerk files → object
- The **type of form** (passport, driver's license, tax return) → resource

You write the manifest. Kubernetes turns it into an object. The resource is the *kind* of object it is.


---

## Why the Word Matters

In the original document, you'll notice phrases like:

> *"This manifest tells Kubernetes: Create a Pod named `my-app`..."*

The word **manifest** signals a specific idea: this is a **declarative file**, stored in Git, versioned, reviewed, and applied. It's not a script, not a command, not an imperative instruction — it's a **written declaration of desired state**.

That's why in CI/CD and GitOps (like Argo CD), you'll hear:

- *"The manifests live in the `deploy/` folder."*
- *"Argo CD syncs the manifests from Git to the cluster."*
- *"Render the manifests with Helm or Kustomize before applying."*

In every case, "manifest" means **the YAML (or JSON) files that describe Kubernetes Objects**.

---

## One-Liner to Remember

> **A Kubernetes manifest is a YAML file that declares what you want Kubernetes to make true.**

If that clicks, everything else — `kubectl apply`, GitOps, Helm charts, Kustomize overlays — will make a lot more sense, because they're all just different ways of producing or managing manifests.