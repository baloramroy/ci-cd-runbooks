## Manual CD diagram flow

```
                +------------------+
                |   Docker Hub     |
                |------------------|
                | myapp:v1         |
                +--------+---------+
                         |
                         | (Image reference only)
                         |
                         ▼
                +----------------------+
                | Git Repository       |
                |----------------------|
                | deployment.yaml      |
                | image: myapp:v1      |
                +----------+-----------+
                           ▲
                           |
                    git push from laptop
                           |
                           ▼
                +----------------------+
                |      Argo CD         |
                |----------------------|
                | Watches Git          |
                +----------+-----------+
                           |
                           ▼
                +----------------------+
                | Kubernetes API Server|
                +----------+-----------+
                           |
                  Schedules Pods
                           |
             +-------------+-------------+
             |                           |
             ▼                           ▼
      +--------------+            +--------------+
      | Worker Node1 |            | Worker Node2 |
      |--------------|            |--------------|
      | myapp:v1 Pod |            | myapp:v1 Pod |
      +--------------+            +--------------+

```
