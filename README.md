# Argo CD — GitOps Learning & Practice

This repository contains my hands-on learning and practice with **Argo CD**, a declarative GitOps continuous delivery tool for Kubernetes.

The goal of this documentation is to record the concepts, configurations, commands, troubleshooting steps, and practical implementations I have learned while working with Argo CD.


# 1. What is Argo CD?

**Argo CD** is a declarative, GitOps-based continuous delivery tool for Kubernetes.

Instead of manually applying Kubernetes manifests using:

```bash
kubectl apply -f deployment.yaml
```

we store the desired Kubernetes configuration in Git.

Argo CD continuously compares:

```text
Git Repository
      |
      | Desired State
      v
   Argo CD
      |
      | Reconciliation
      v
 Kubernetes Cluster
      |
      | Actual State
      v
   Workloads
```

The Git repository becomes the source of truth for the Kubernetes application's desired state.

---

# 2. What is GitOps?

GitOps is an operational model where Git is used as the source of truth for infrastructure and application configuration.

Instead of making changes directly inside Kubernetes, changes are committed to Git.

Example:

```text
Developer
    |
    | Git Commit
    v
GitHub Repository
    |
    | Argo CD detects change
    v
Argo CD
    |
    | Sync
    v
Kubernetes
```

## Traditional Deployment

```text
Developer
    |
    v
kubectl apply
    |
    v
Kubernetes
```

## GitOps Deployment

```text
Developer
    |
    v
Git
    |
    v
Argo CD
    |
    v
Kubernetes
```

This provides better traceability, version control, and reproducibility.

---

# 3. Why Argo CD?

Argo CD provides several important capabilities:

* GitOps-based Kubernetes deployments
* Declarative application management
* Automated synchronization
* Application health monitoring
* Drift detection
* Self-healing
* Rollbacks
* Multi-cluster deployment
* Helm support
* Kustomize support
* Directory-based manifest deployment
* ApplicationSet
* RBAC
* Project-based access control
* Integration with Git repositories

---

# 4. Argo CD Architecture

A simplified architecture looks like this:

```text
                    +----------------+
                    | Git Repository |
                    |    GitHub      |
                    +-------+--------+
                            |
                            |
                            v
                    +---------------+
                    |    Argo CD    |
                    +---------------+
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
   Application       Application       Application
       Dev               Prod              Other
          |                 |                 |
          +-----------------+-----------------+
                            |
                            v
                    +---------------+
                    |  Kubernetes   |
                    |    Cluster    |
                    +---------------+
```

Argo CD continuously monitors the desired state in Git and compares it with the actual state in Kubernetes.

---

# 5. Core Argo CD Components

Argo CD consists of several components.

## API Server

The API server provides the interface for:

* CLI
* Web UI
* API clients

Example:

```bash
argocd app list
```

---

## Repository Server

The repository server is responsible for:

* Connecting to Git repositories
* Fetching manifests
* Rendering Helm charts
* Rendering Kustomize configurations
* Generating Kubernetes manifests

---

## Application Controller

The application controller is responsible for:

* Comparing desired and live state
* Detecting differences
* Performing synchronization
* Monitoring application health
* Detecting drift

---

## Redis

Redis is used by Argo CD for caching.

---

## Dex

Dex can be used for identity integration and authentication.

---

# 6. Installing Argo CD

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify:

```bash
kubectl get pods -n argocd
```

Expected components include:

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-redis
argocd-dex-server
```

---

# 7. Accessing Argo CD

Check services:

```bash
kubectl get svc -n argocd
```

For a local Kubernetes cluster, port forwarding can be used:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then access:

```text
https://localhost:8080
```

---

# 8. Argo CD CLI

Login using:

```bash
argocd login localhost:8080 --insecure
```

Get the initial admin password:

```bash
argocd admin initial-password -n argocd
```

Or:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

---

# 9. Connecting a Git Repository

Argo CD applications normally use a Git repository as their source.

Example repository:

```text
https://github.com/example/gitops-repo.git
```

Add the repository:

```bash
argocd repo add https://github.com/example/gitops-repo.git
```

For private repositories, credentials may be required.

Verify:

```bash
argocd repo list
```

---

# 10. Argo CD Applications

An **Application** is one of the most important concepts in Argo CD.

An Application defines:

1. Where the manifests are located
2. Which Git revision should be used
3. Which Kubernetes cluster should receive the deployment
4. Which namespace should receive the resources
5. How synchronization should happen

Conceptually:

```text
Application
   |
   +-- Source
   |     |
   |     +-- Repository
   |     +-- Revision
   |     +-- Path
   |
   +-- Destination
   |     |
   |     +-- Cluster
   |     +-- Namespace
   |
   +-- Sync Policy
```

---

# 11. Application Sources

Argo CD can work with several configuration sources.

Common examples include:

* Directory
* Helm
* Kustomize
* Jsonnet
* Plugins

Two configurations I practiced extensively were:

```text
Directory
Helm
```

---

# 12. Directory Applications

A Directory application allows Argo CD to load Kubernetes manifests directly from a directory in Git.

Example repository:

```text
gitops-repo/
│
├── dev/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   └── service.yaml
│
└── prod/
    ├── namespace.yaml
    ├── deployment.yaml
    └── service.yaml
```

Argo CD can point to:

```text
/dev
```

or:

```text
/prod
```

and deploy the YAML files found there.

---

## Directory Application Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-application
  namespace: argocd
spec:
  project: argo-demo

  source:
    repoURL: https://github.com/example/gitops-repo.git
    targetRevision: main
    path: dev
    directory:
      recurse: true

  destination:
    server: https://kubernetes.default.svc
    namespace: dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

# 13. Include and Exclude Files

Directory applications can be configured to control which files are processed.

Example:

```yaml
directory:
  include: "*.yaml"
```

This allows YAML files to be included.

Files can also be excluded.

Example:

```yaml
directory:
  exclude: "test.yaml"
```

Multiple patterns can also be used depending on the configuration.

This is useful when a Git directory contains:

```text
deployment.yaml
service.yaml
configmap.yaml
test.yaml
README.md
```

but only selected manifests should be deployed.

---

# 14. Helm Applications

Argo CD can deploy Helm charts.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-helm
  namespace: argocd
spec:
  project: argo-demo

  source:
    repoURL: https://github.com/example/helm-repo.git
    targetRevision: main
    path: charts/nginx

    helm:
      releaseName: nginx-demo

  destination:
    server: https://kubernetes.default.svc
    namespace: dev
```

---

# 15. Helm Values Override

One useful feature I practiced was overriding Helm values from the Argo CD Application.

Example:

```yaml
source:
  repoURL: https://github.com/example/helm-repo.git
  targetRevision: main
  path: charts/nginx

  helm:
    releaseName: nginx-dev

    values: |
      replicaCount: 2

      service:
        type: ClusterIP
```

This allows environment-specific configuration without necessarily changing the original Helm chart.

---

# 16. Argo CD Projects

An **Argo CD Project** provides logical grouping and access control for Applications.

Projects can control:

* Which repositories Applications can use
* Which clusters Applications can deploy to
* Which namespaces Applications can deploy to
* Which Kubernetes resources Applications can create

Example:

```text
Argo CD
   |
   +-- Project: argo-demo
          |
          +-- dev-application
          |
          +-- prod-application
          |
          +-- monitoring-application
```

---

# 17. Creating an Argo CD Project

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: argo-demo
  namespace: argocd

spec:
  description: Demo Argo CD Project

  sourceRepos:
    - "*"

  destinations:
    - namespace: dev
      server: https://kubernetes.default.svc

    - namespace: prod
      server: https://kubernetes.default.svc

  clusterResourceWhitelist:
    - group: "*"
      kind: "*"

  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
```

Apply:

```bash
kubectl apply -f project.yaml
```

Verify:

```bash
argocd proj list
```

---

# 18. Dev and Prod Applications

During my hands-on practice, I created an Argo CD project:

```text
argo-demo
```

and created multiple Applications targeting:

```text
dev
prod
```

Example structure:

```text
argo-demo
│
├── dev-app-1
├── dev-app-2
└── prod-app
```

The Applications can use the same Git repository while targeting different namespaces.

Example:

```text
Git
 |
 +-- manifests/
      |
      +-- dev/
      |
      +-- prod/
```

This provides a clear separation between environments.

---

# 19. Sync in Argo CD

**Sync** is the process of making the Kubernetes cluster match the desired state defined in Git.

Example:

```text
Git
 |
 | Desired State
 v
Argo CD
 |
 | Compare
 v
Kubernetes
 |
 | Difference detected
 v
Sync
 |
 v
Updated Kubernetes resources
```

---

# 20. Sync Status

An Argo CD Application can have statuses such as:

```text
Synced
OutOfSync
Unknown
```

### Synced

The live Kubernetes state matches the desired Git state.

### OutOfSync

The live state differs from the desired Git state.

### Unknown

Argo CD cannot currently determine the application state.

---

# 21. Application Health

Applications can also have health states.

Examples include:

```text
Healthy
Progressing
Degraded
Suspended
Missing
Unknown
```

Health status helps determine whether the deployed Kubernetes resources are functioning as expected.

---

# 22. Automated Sync

Argo CD can automatically synchronize applications when Git changes.

Example:

```yaml
syncPolicy:
  automated: {}
```

More commonly:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

---

# 23. Self-Heal

Self-healing allows Argo CD to detect manual changes made directly to Kubernetes.

Example:

```text
Git
 |
 | replicas = 3
 v
Argo CD
 |
 v
Kubernetes
 |
 | Manual change
 v
replicas = 1
```

Argo CD detects the difference:

```text
Desired: 3
Live:    1
```

With self-healing enabled, Argo CD can reconcile the resource back to the Git-defined state.

---

# 24. Prune

Pruning removes Kubernetes resources that are no longer defined in Git.

Example:

Initially:

```text
deployment.yaml
service.yaml
configmap.yaml
```

If `configmap.yaml` is removed from Git, Argo CD can remove the corresponding Kubernetes resource when pruning is enabled.

```yaml
syncPolicy:
  automated:
    prune: true
```

---

# 25. Sync Options

Argo CD supports various synchronization options.

Example:

```yaml
syncOptions:
  - CreateNamespace=true
```

This allows Argo CD to create the destination namespace if it does not already exist.

Example:

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
```

Other options can control how resources are created, replaced, validated, or synchronized.

---

# 26. ApplicationSet

**ApplicationSet** is used to automatically generate multiple Argo CD Applications from a template.

This is useful when managing:

* Multiple environments
* Multiple clusters
* Multiple applications
* Multiple teams

Example:

```text
ApplicationSet
      |
      +-- dev-app
      |
      +-- staging-app
      |
      +-- prod-app
```

Instead of manually creating each Application, ApplicationSet can generate them.

---

# 27. ApplicationSet Example

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: applications
  namespace: argocd

spec:
  generators:
    - list:
        elements:
          - environment: dev
            namespace: dev

          - environment: prod
            namespace: prod

  template:
    metadata:
      name: "{{environment}}-application"

    spec:
      project: argo-demo

      source:
        repoURL: https://github.com/example/gitops-repo.git
        targetRevision: main
        path: "{{environment}}"

      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"

      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

# 28. Argo CD Image Updater

I also worked with **Argo CD Image Updater**.

Image Updater can monitor container images and update image versions based on configured policies.

Example:

```text
Developer
    |
    v
Build Docker Image
    |
    v
Container Registry
    |
    v
Argo CD Image Updater
    |
    v
Git / Argo CD
    |
    v
Kubernetes
```

This helps automate container image updates within a GitOps workflow.

---

# 29. GitOps Workflow

A typical workflow looks like:

```text
Developer
    |
    | Code Change
    v
Git Repository
    |
    | CI Pipeline
    v
Container Image
    |
    v
Container Registry
    |
    | Image Update
    v
Git / Image Updater
    |
    v
Argo CD
    |
    | Sync
    v
Kubernetes
```

---

# 30. Application Lifecycle

A typical Argo CD Application lifecycle:

```text
Create Application
        |
        v
Connect Git Repository
        |
        v
Read Manifests
        |
        v
Generate Desired State
        |
        v
Compare Live vs Desired
        |
        v
OutOfSync?
     /     \
   Yes      No
    |        |
    v        v
  Sync     Synced
    |
    v
Kubernetes Updated
```

---

# 31. Important Argo CD Commands

## Login

```bash
argocd login <ARGOCD_SERVER>
```

---

## List Applications

```bash
argocd app list
```

---

## Get Application

```bash
argocd app get <APP_NAME>
```

---

## Sync Application

```bash
argocd app sync <APP_NAME>
```

---

## Delete Application

```bash
argocd app delete <APP_NAME>
```

---

## Create Application

```bash
argocd app create <APP_NAME>
```

---

## Application History

```bash
argocd app history <APP_NAME>
```

---

## Application Diff

```bash
argocd app diff <APP_NAME>
```

---

## Application Logs

```bash
argocd app logs <APP_NAME>
```

---

## Refresh Application

```bash
argocd app get <APP_NAME> --refresh
```

---

# 32. Project Commands

List projects:

```bash
argocd proj list
```

Get project:

```bash
argocd proj get argo-demo
```

Create project:

```bash
argocd proj create argo-demo
```

---

# 33. Repository Commands

List repositories:

```bash
argocd repo list
```

Add repository:

```bash
argocd repo add <REPOSITORY_URL>
```

Remove repository:

```bash
argocd repo rm <REPOSITORY_URL>
```

---

# 34. Useful kubectl Commands

Check Argo CD pods:

```bash
kubectl get pods -n argocd
```

Check services:

```bash
kubectl get svc -n argocd
```

Check Applications:

```bash
kubectl get applications -n argocd
```

Check ApplicationSets:

```bash
kubectl get applicationsets -n argocd
```

Check Projects:

```bash
kubectl get appprojects -n argocd
```

Describe an Application:

```bash
kubectl describe application <APP_NAME> -n argocd
```

---

# 35. Troubleshooting

Troubleshooting Argo CD was an important part of my hands-on learning.

When an Application is not synchronizing correctly, the first step is to inspect its status.

```bash
argocd app get <APP_NAME>
```

Check:

```text
Sync Status
Health Status
Repository
Revision
Destination
Namespace
Resources
```

---

# 36. Checking Argo CD Logs

API server:

```bash
kubectl logs -n argocd deploy/argocd-server
```

Repository server:

```bash
kubectl logs -n argocd deploy/argocd-repo-server
```

Application controller:

```bash
kubectl logs -n argocd \
  statefulset/argocd-application-controller
```

These logs can help identify:

* Git repository problems
* Manifest rendering errors
* Authentication issues
* Kubernetes API errors
* Sync failures
* Resource validation errors

---

# 37. Common Sync Problems

## Problem 1 — Application is OutOfSync

Check:

```bash
argocd app diff <APP_NAME>
```

Then:

```bash
argocd app sync <APP_NAME>
```

---

## Problem 2 — Wrong Namespace

Verify:

```bash
argocd app get <APP_NAME>
```

Check:

```yaml
destination:
  namespace: dev
```

and ensure the namespace exists:

```bash
kubectl get namespace
```

---

## Problem 3 — Namespace Does Not Exist

Use:

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
```

or create it manually:

```bash
kubectl create namespace dev
```

---

## Problem 4 — Wrong Git Path

Verify the Application:

```yaml
source:
  repoURL: ...
  targetRevision: main
  path: dev
```

Make sure the path exists in Git.

---

## Problem 5 — Manifest Error

Check:

```bash
argocd app get <APP_NAME>
```

and:

```bash
kubectl describe application <APP_NAME> -n argocd
```

Repository server logs can also help:

```bash
kubectl logs -n argocd deploy/argocd-repo-server
```

---

# 38. Application Configuration Troubleshooting

When troubleshooting an Application, I check the following:

```text
1. Application name
2. Project
3. Repository URL
4. Git revision
5. Git path
6. Helm configuration
7. Helm values
8. Release name
9. Destination cluster
10. Destination namespace
11. Sync policy
12. Sync options
13. Application health
14. Kubernetes resources
15. Argo CD logs
```

This provides a structured troubleshooting approach.

---

# 39. Argo CD Project Troubleshooting

When an Application cannot deploy because of project restrictions, check:

```yaml
sourceRepos:
```

and:

```yaml
destinations:
```

For example:

```yaml
destinations:
  - namespace: dev
    server: https://kubernetes.default.svc
```

If the Application tries to deploy to:

```text
prod
```

but the Project only permits:

```text
dev
```

the deployment can be blocked by the Project configuration.

Therefore, Project configuration should match the intended deployment environments.

---

# 40. Practical Repository Structure

A GitOps repository can be structured like this:

```text
gitops-repository/
│
├── applications/
│   ├── dev-app.yaml
│   ├── prod-app.yaml
│   └── applicationset.yaml
│
├── projects/
│   └── argo-demo.yaml
│
├── environments/
│   │
│   ├── dev/
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   │
│   └── prod/
│       ├── namespace.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       └── configmap.yaml
│
└── helm/
    └── nginx/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

---

# 41. Example GitOps Structure

A more complete setup:

```text
GitHub
│
└── gitops-repository
    │
    ├── argocd
    │   ├── projects
    │   │   └── argo-demo.yaml
    │   │
    │   ├── applications
    │   │   ├── dev.yaml
    │   │   └── prod.yaml
    │   │
    │   └── applicationsets
    │       └── appset.yaml
    │
    ├── apps
    │   ├── dev
    │   │   ├── deployment.yaml
    │   │   └── service.yaml
    │   │
    │   └── prod
    │       ├── deployment.yaml
    │       └── service.yaml
    │
    └── helm
        └── application
            ├── Chart.yaml
            ├── values.yaml
            └── templates/
```

---

# 42. Argo CD vs kubectl

Without GitOps:

```bash
kubectl apply -f deployment.yaml
```

The change is directly applied to Kubernetes.

With Argo CD:

```text
Git
 |
 | Commit
 v
Argo CD
 |
 | Reconciliation
 v
Kubernetes
```

The Git repository provides a historical record of configuration changes.

---

# 43. Desired State vs Live State

One of the most important concepts in Argo CD is the difference between:

```text
Desired State
```

and:

```text
Live State
```

### Desired State

Defined in Git.

Example:

```yaml
replicas: 3
```

### Live State

Currently running in Kubernetes.

Example:

```text
replicas: 2
```

Argo CD identifies the difference:

```text
Desired = 3
Live    = 2
```

Therefore:

```text
Application = OutOfSync
```

After reconciliation:

```text
Desired = 3
Live    = 3
```

Application becomes:

```text
Synced
```

---

# 44. Drift Detection

Drift occurs when the Kubernetes cluster differs from Git.

Example:

```text
Git
replicas: 3

Kubernetes
replicas: 5
```

Argo CD detects this difference.

This is one of the key benefits of GitOps.

---

# 45. Rollbacks

Argo CD maintains application history.

View history:

```bash
argocd app history <APP_NAME>
```

A previous revision can be used for rollback when appropriate.

The exact rollback workflow should be chosen carefully depending on whether Git is being treated as the authoritative source and whether the change should also be represented in Git.

---

# 46. Multi-Environment Deployment

Argo CD can be used to manage:

```text
Development
     |
     v
Staging
     |
     v
Production
```

Each environment can have:

* Separate namespace
* Separate values
* Separate Application
* Separate configuration
* Separate deployment policies

Example:

```text
dev
 └── dev-app

staging
 └── staging-app

prod
 └── prod-app
```

---

# 47. Security and RBAC

Argo CD supports access control through:

* Projects
* Roles
* Policies
* Repository restrictions
* Destination restrictions
* Kubernetes RBAC
* Argo CD RBAC

Projects can be used to restrict where applications are allowed to deploy.

For example:

```text
Project: development
    |
    +-- dev namespace

Project: production
    |
    +-- prod namespace
```

This helps separate application environments and deployment permissions.

---

# 48. Important Concepts Learned

Through my hands-on work, I learned the following Argo CD concepts:

### Applications

Applications define the relationship between:

```text
Git Source
+
Kubernetes Destination
+
Synchronization Policy
```

### Projects

Projects provide:

```text
Application Organization
+
Access Control
+
Repository Restrictions
+
Destination Restrictions
```

### Helm

Helm applications allow:

```text
Chart Deployment
+
Release Name
+
Values Override
```

### Directory

Directory applications allow:

```text
Multiple YAML Files
+
Recursive Directory Processing
+
Include / Exclude Rules
```

### Sync

Sync reconciles:

```text
Git Desired State
        |
        v
Kubernetes Live State
```

### ApplicationSet

ApplicationSet automates creation of multiple Applications.

### Image Updater

Image Updater can automate container image update workflows.

---

# 49. My Argo CD Hands-On Work

During my practical learning, I worked on:

* Installing and configuring Argo CD
* Connecting Git repositories
* Creating Argo CD Applications
* Working with Directory-type Applications
* Working with Helm-type Applications
* Overriding Helm values
* Configuring Helm release names
* Using include/exclude rules for Directory applications
* Creating Argo CD Projects
* Creating a project named `argo-demo`
* Creating multiple Applications inside the project
* Deploying Applications to `dev` and `prod` namespaces
* Troubleshooting synchronization issues
* Troubleshooting configuration differences
* Understanding `Synced` and `OutOfSync` states
* Working with ApplicationSet
* Connecting Git repositories containing Kubernetes manifests
* Testing Git-based changes and observing Argo CD reconciliation
* Working with Argo CD Image Updater

---

# 50. Example End-to-End GitOps Flow

The complete workflow I practiced can be represented as:

```text
                   Developer
                       |
                       |
                  Code Change
                       |
                       v
                 Git Repository
                       |
                       |
                  Git Commit
                       |
                       v
                    Argo CD
                       |
             +---------+---------+
             |                   |
             v                   v
         Dev App             Prod App
             |                   |
             v                   v
        dev namespace       prod namespace
             |                   |
             +---------+---------+
                       |
                       v
                  Kubernetes
```

---

# 51. Troubleshooting Methodology

My general troubleshooting process for Argo CD is:

```text
Step 1
Check Application status

        ↓

Step 2
Check Sync status

        ↓

Step 3
Check Health status

        ↓

Step 4
Check Git repository and revision

        ↓

Step 5
Check Application path

        ↓

Step 6
Check namespace and destination

        ↓

Step 7
Check Helm values/configuration

        ↓

Step 8
Run application diff

        ↓

Step 9
Check Kubernetes resources

        ↓

Step 10
Check Argo CD logs
```

Useful commands:

```bash
argocd app get <APP_NAME>

argocd app diff <APP_NAME>

kubectl get applications -n argocd

kubectl describe application <APP_NAME> -n argocd

kubectl get pods -n <NAMESPACE>

kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

---

# 52. Best Practices

## Use Git as the Source of Truth

Avoid making permanent manual changes directly in Kubernetes.

Prefer:

```text
Git → Argo CD → Kubernetes
```

---

## Separate Environments

Use separate configurations for:

```text
dev
staging
prod
```

---

## Use Projects

Use Projects to control:

* Allowed repositories
* Allowed clusters
* Allowed namespaces
* Allowed resource types

---

## Use Automated Sync Carefully

Automated sync can reduce manual work, but production environments should have appropriate controls around changes.

---

## Keep Repository Structure Organized

Use a predictable structure:

```text
applications/
projects/
environments/
helm/
```

---

## Monitor Application Health

Do not rely only on:

```text
Synced
```

Also check:

```text
Health
```

An Application can be synchronized while the deployed workload is unhealthy.

---

#  Takeaways

The main concepts I learned from Argo CD are:

```text
GitOps
   ↓
Git as Source of Truth
   ↓
Argo CD Application
   ↓
Desired State
   ↓
Sync
   ↓
Kubernetes
```

The major Argo CD building blocks are:

```text
Applications
Projects
Repositories
Helm
Directory
Kustomize
ApplicationSet
Sync
Self-Heal
Prune
RBAC
Image Updater
```



#   Reference

| Concept                | Purpose                                                   |
| ---------------------- | --------------------------------------------------------- |
| Application            | Defines a Kubernetes deployment managed by Argo CD        |
| Project                | Organizes applications and controls deployment boundaries |
| Repository             | Source of desired configuration                           |
| Directory              | Deploys Kubernetes manifests from a directory             |
| Helm                   | Deploys applications using Helm charts                    |
| Values                 | Customizes Helm deployments                               |
| Sync                   | Reconciles Kubernetes with Git                            |
| Self-Heal              | Corrects detected drift                                   |
| Prune                  | Removes resources no longer defined                       |
| ApplicationSet         | Generates multiple Applications                           |
| Image Updater          | Automates image update workflows                          |
| Repository Server      | Retrieves and renders manifests                           |
| Application Controller | Performs reconciliation                                   |
| API Server             | Provides Argo CD API/UI/CLI access                        |
| Health                 | Shows application/resource health                         |
| Sync Status            | Shows desired vs live state                               |

---

#  Useful Commands Cheat Sheet

```bash
# Applications
argocd app list
argocd app get <APP>
argocd app sync <APP>
argocd app diff <APP>
argocd app history <APP>
argocd app delete <APP>

# Projects
argocd proj list
argocd proj get argo-demo

# Repositories
argocd repo list
argocd repo add <REPO>

# Kubernetes
kubectl get applications -n argocd
kubectl get appprojects -n argocd
kubectl get applicationsets -n argocd

kubectl get pods -n argocd
kubectl get svc -n argocd

# Troubleshooting
kubectl describe application <APP> -n argocd

kubectl logs -n argocd deploy/argocd-server

kubectl logs -n argocd deploy/argocd-repo-server

kubectl logs -n argocd \
  statefulset/argocd-application-controller
```

