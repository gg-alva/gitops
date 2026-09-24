# Argo CD – Work Done Yesterday

## Overview

I focused on understanding **Argo CD Applications, application sources, and Projects** as part of my GitOps learning.

## Work Completed

* Learned about **Argo CD Applications** and how they define the desired state of Kubernetes resources.
* Explored different application source types, particularly **Helm** and **Directory**.
* Practiced using **Helm applications** and learned how to override Helm chart values and configure the **release name**.
* Worked with **Directory applications** to load multiple Kubernetes YAML manifests from a Git repository.
* Learned how to use **include and exclude configurations** to control which manifest files Argo CD processes.
* Studied **Argo CD Projects** and how they can be used to organize applications and control repositories, destinations, and resources.
* Reviewed how Git acts as the **source of truth** in a GitOps workflow.

## Key Learnings

```text
Git Repository
      |
      v
Argo CD Application
      |
      +-- Helm
      |     └── Values / Release Name
      |
      +-- Directory
            └── Multiple YAML Files
                    |
                    v
              Kubernetes
```

