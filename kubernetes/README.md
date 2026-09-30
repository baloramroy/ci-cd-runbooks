## The most important mental model

```text
                            kubectl
                               │
                               ▼
                        ┌─────────────┐
                        │ API Server  │
                        └──────┬──────┘
                               │
                ┌──────────────┼──────────────┐
                ▼              ▼              ▼
              etcd         Scheduler     Controller
                │
                │
                └────────── Cluster State
        
        
                      Worker Nodes
                ┌───────────────────────┐
                │                       │
                │ kubelet               │
                │    │                  │
                │    ▼                  │
                │ containerd            │
                │    │                  │
                │    ▼                  │
                │   Pods                │
                │                       │
                │ CNI ← Pod networking  │
                └───────────────────────┘

```