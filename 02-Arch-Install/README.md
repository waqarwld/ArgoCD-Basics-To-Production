# Argo CD Architecture Deep Dive & Step-by-Step Installation

## Video reference for this lecture is the following:

[![Watch the video](https://img.youtube.com/vi/0dTRGi1oCfU/maxresdefault.jpg)](https://www.youtube.com/watch?v=0dTRGi1oCfU&ab_channel=CloudWithVarJosh)


---
## ⭐ Support the Project  
If this **repository** helps you, give it a ⭐ to show your support and help others discover it! 

---

## Table of Contents

- [Introduction](#introduction)
- [What is GitOps (Quick Recap)](#what-is-gitops-quick-recap)
  - [GitOps Principles (Recap)](#gitops-principles-recap)
- [What Is Argo CD?](#what-is-argo-cd)
- [Understanding Argo CD Through the Diagram](#understanding-argo-cd-through-the-diagram)
  - [Git as the Single Source of Truth](#git-as-the-single-source-of-truth)
  - [Argo CD as the GitOps Control Plane](#argo-cd-as-the-gitops-control-plane)
  - [Application-Level View](#application-level-view)
  - [Pull-Based Reconciliation vs Push-Based Deployment](#pull-based-reconciliation-vs-push-based-deployment)
- [Argo CD Architecture](#argo-cd-architecture)
  - [Core Components of Argo CD](#core-components-of-argo-cd)
    - [API Server](#1-api-server)
    - [Repository Server](#2-repository-server)
    - [Application Controller](#3-application-controller)
  - [How DevOps Engineer and CI Fit Into This Architecture](#how-devops-engineer-and-ci-fit-into-this-architecture-diagram-context)
  - [Supporting Components](#supporting-components)
- [Lab: Installing Argo CD (Local Setup Using KIND)](#lab-installing-argo-cd-local-setup-using-kind)
  - [Prerequisites](#prerequisites)
  - [Create a KIND Cluster](#step-1-create-a-kind-cluster)
  - [Add the Argo CD Helm Repository](#step-2-add-the-argo-cd-helm-repository)
  - [Create Namespace for Argo CD](#step-3-create-namespace-for-argo-cd)
  - [Install Argo CD Using Helm](#step-4-install-argo-cd-using-helm)
  - [Verify Argo CD Installation](#step-5-verify-argo-cd-installation)
  - [Access Argo CD UI](#step-6-access-argo-cd-ui-port-forwarding)
  - [Retrieve Initial Admin Password](#step-7-retrieve-initial-admin-password)
  - [Login and Reset Password](#step-8-login-and-reset-password)
- [Conclusion](#conclusion)
- [References (Official)](#references-official)

---

## Introduction

Before deploying applications with Argo CD, it is essential to understand **why Argo CD exists**, **what problem it solves**, and **how it fits into a GitOps-based Kubernetes architecture**.

In this lecture, we start with a **focused GitOps recap**, revisiting only the principles required to understand Argo CD’s role. We then introduce Argo CD as a **Kubernetes-native GitOps control plane**, explaining what it does, what it deliberately does not do, and how it differs from traditional CI/CD deployment models.

We walk through the **Argo CD architecture**, covering its core and supporting components, and explain how DevOps engineers, CI systems, Git repositories, and Kubernetes interact in a pull-based GitOps workflow. Finally, we install Argo CD on a local Kubernetes cluster using **KIND and Helm**, verify the installation, and access the Argo CD UI.

This lecture establishes the **foundational mental model** required before working with Application CRDs and real GitOps deployments.

---

## What is GitOps (Quick Recap)

Before introducing Argo CD, it is important to restate GitOps **only at the level required** to understand why Argo CD exists.

GitOps is an operating model where **Git declares the desired state of the system**, and **automated agents continuously reconcile the running system** to match what is defined in Git. Deployments are **pull-based**, Git remains the **single source of truth**, and any drift from the declared state is automatically detected and corrected.

A true GitOps architecture is built on a small set of **non-negotiable principles**.

### GitOps Principles (Recap)

Reference: https://opengitops.dev/

**Declarative:**
Desired state is declared in configuration, not enforced through imperative commands.

**Versioned and Immutable:**
Desired state is stored in Git with full history, immutability, and auditability.

**Pulled Automatically:**
Automated agents pull desired state from Git, environments are never pushed to.

**Continuously Reconciled:**
Agents continuously detect drift and enforce the declared desired state.

These principles imply the need for a **long-running controller** that can observe Git, observe the cluster, and continuously reconcile the two. This is the gap Argo CD fills.



---

## What Is Argo CD?

![Alt text](/images/2a.png)

Argo CD is a **Kubernetes-native GitOps control plane** that implements GitOps for Kubernetes workloads by continuously reconciling the desired state declared in Git with the live state of a Kubernetes cluster. It runs inside the cluster and enforces state through **continuous comparison and reconciliation**, rather than event-driven deployments.

To understand Argo CD clearly, it helps to break down **what it is responsible for and what it deliberately does not do**.

**Type:**
It is a **Kubernetes-native GitOps control plane**. Argo CD runs as a long-lived control-plane component inside Kubernetes, follows the Kubernetes controller pattern, and is responsible for enforcing state, not executing pipelines or deployment jobs.
>Argo CD is **Kubernetes-native** because it runs inside the cluster and extends Kubernetes using CRDs, rather than operating as an external deployment tool.

**Purpose:**
Argo CD exists to **continuously reconcile cluster state with Git**. It compares the desired state stored in Git with the live state observed through the Kubernetes API and converges the system back to the declared state whenever drift is detected.

**Model:**
Argo CD follows a **pull-based deployment model**, where it pulls desired state from Git repositories. CI systems never push manifests or authenticate to the cluster, preserving a strict **Git to cluster** control flow.

**Security:**
Because Argo CD runs inside the cluster, **CI systems do not require direct cluster credentials**. Access to production environments is governed through Git permissions, reviews, and approvals, significantly reducing blast radius and credential exposure.

**Scope:**
Argo CD enforces state **only for applications explicitly configured under its management**. Other workloads may exist in the same cluster without being governed by Argo CD, allowing selective and gradual GitOps adoption.

**Behavior:**
Argo CD continuously **detects drift and enforces the declared desired state**. Manual or out-of-band changes to managed resources are observed and reconciled back to what is defined in Git.

**Role:**
Argo CD **enforces GitOps but does not build or deploy applications**. It does not compile code, create artifacts, or replace CI systems. Its responsibility begins only after desired state is committed to Git.

---

## Understanding Argo CD Through the Diagram

![Alt text](/images/2a.png)
The diagram shows a **single Kubernetes cluster** with **two applications currently managed by Argo CD**: `app1` and `app2`. It also intentionally leaves room for additional applications (for example `app3`, `app4`) that are **not yet managed by Argo CD** and continue to follow a **traditional push-based CI/CD model**.

This reflects how GitOps is adopted in real environments: **incrementally, not all at once**.

---

On the left, we have the **CI systems**, whose responsibility is limited strictly to **build-time concerns**. CI checks out source code, runs tests and scans, builds artifacts, and publishes them. CI **never deploys directly to the Kubernetes cluster**. Its only interaction point is Git.

---

### Git as the Single Source of Truth

In the center, the **Git repository layer** represents the **single source of truth** and is intentionally split into **two distinct repository types per application**:

* **Application repositories** (`app1-repo`, `app2-repo`)
  These contain application source code and CI definitions. CI systems update these repositories as part of normal development workflows.

* **Configuration repositories** (`app1-config-repo`, `app2-config-repo`)
  These contain **declarative Kubernetes manifests** that describe how the application should run in the cluster. These repositories declare **desired runtime state**, not deployment instructions.

CI may propose changes to configuration repositories via pull requests, but **Git itself does not deploy anything**. Git only declares intent.

---

### Argo CD as the GitOps Control Plane

Inside the Kubernetes cluster, Argo CD runs as a **control-plane component** following the Kubernetes controller pattern.

Argo CD is configured to **watch only the configuration repositories**. It continuously pulls desired state from `app1-config-repo` and `app2-config-repo`, compares it with the live state observed via the Kubernetes API, and reconciles any differences.

Argo CD does **not read application source repositories**, does **not react to CI events**, and does **not require CI systems to authenticate to the cluster**.

---

### Application-Level View

* `app1-ns` and `app2-ns` are namespaces **reconciled by Argo CD**.
  Their desired state is continuously enforced based on what is declared in their respective configuration repositories.

* Applications such as `app3` or `app4` are **not shown in the diagram**, but may still exist in the same cluster.
  These applications are **not managed by Argo CD** and typically follow a **traditional push-based CI/CD approach**, where an orchestrator or deployment pipeline applies manifests or runs Helm directly against the cluster.

This mixed model is common during GitOps adoption and is fully supported.

---

### Pull-Based Reconciliation vs Push-Based Deployment

For applications managed by Argo CD:

* Desired state is **pulled from Git**
* Drift is continuously detected
* Manual or out-of-band changes are reverted
* Enforcement never stops

For applications not managed by Argo CD:

* Deployments are **push-based**
* CI/CD pipelines or orchestrators apply changes
* Drift detection and correction are manual or pipeline-driven

The diagram intentionally shows **only the pull-based GitOps flow**, while acknowledging that push-based workflows may still coexist.

>Argo CD does not own the entire cluster.
It enforces state **only for applications explicitly configured under its management**, while other applications may continue to use traditional CI/CD workflows. This makes GitOps **practical, incremental, and production-friendly**.

---

## Argo CD Architecture

![Alt text](/images/2b.png)

At a high level, Argo CD follows the **Kubernetes control-plane pattern**. It is not a single binary but a set of cooperating components, each with a clearly defined responsibility.

In a real setup, **DevOps engineers and CI systems do not deploy applications directly**. Instead, they interact with Git and the Argo CD control plane, while Argo CD itself is the only component that reconciles state inside the Kubernetes cluster.

Understanding these components explains how Argo CD connects **DevOps workflows, CI pipelines, Git repositories, and Kubernetes**, and why reconciliation works the way it does.

---

### Core Components of Argo CD



#### **1. API Server**

* The Argo CD API Server is the **entry point into Argo CD**. Just as the Kubernetes API server is the front door to the cluster, the Argo CD API server is the front door to Argo CD itself.
* **DevOps engineers** access Argo CD through the **UI or CLI**, and **CI orchestrators such as Jenkins** communicate with Argo CD through this API to **observe** application state.
* The API server exposes both **REST and gRPC interfaces**, allowing systems like **Jenkins** (or GitLab CI, GitHub Actions, etc.) to **query application status, sync state, and health** without deploying anything directly to the cluster.

> **Note:**
> From both a CI and day-to-day operations perspective, Argo CD is simply another application deployed in Kubernetes.
> Even **Argo CD administrators** typically interact only through the **UI or CLI** and do **not require direct Kubernetes cluster access**.
> Cluster access is needed only in exceptional cases, such as when Argo CD itself is unhealthy and requires infrastructure-level troubleshooting.

>**REST:** Human-readable, HTTP-based APIs commonly used by UIs and scripts for request–response interactions.
**gRPC:** High-performance, binary protocol used for efficient, structured service-to-service communication.

> *As shown in the diagram, interaction with Argo CD is control-plane driven and does not imply Kubernetes API access.*

---



#### **2. Repository Server**

* The Repository Server is responsible for **interacting with Git repositories**, which represent the **desired state** of applications.
* When a DevOps engineer or CI pipeline (for example, Jenkins) pushes changes to a configuration repository, the Repository Server authenticates to Git, checks out the repository, and renders manifests from plain YAML, Helm charts, or Kustomize overlays.
* This component does **not** communicate with the Kubernetes API. Its sole responsibility is answering the question:
  *based on Git, what should the cluster look like?*

  > *In the diagram, this is shown as Argo CD pulling desired state from the app configuration repository.*

---

#### **3. Application Controller**

* The Application Controller is the **heart of Argo CD**. It is responsible for ensuring that the **current state of the cluster matches the desired state** defined in Git.
* It receives the desired state from the Repository Server and observes live state directly from the Kubernetes API. By continuously comparing the two, it detects drift and decides when reconciliation is required.
* Features such as **self-healing, automated reconciliation, and application lifecycle hooks** are implemented here.

  > *As shown in the diagram, this component applies and reconciles resources into application namespaces such as `app1-ns`.*

This reconciliation loop runs continuously and does not stop after a deployment completes.

---

### How DevOps Engineer and CI Fit Into This Architecture (Diagram Context)

* **DevOps engineers** define application structure, environments, and Kubernetes manifests, and push changes to Git.
* **Jenkins** (or any CI system such as GitLab CI or GitHub Actions) is responsible for **building, testing, scanning artifacts**, and updating Git repositories with new configuration or image references.
* Neither DevOps engineers nor CI systems deploy workloads directly to Kubernetes.
  **Git is the handoff point, and Argo CD is the enforcer.**

---


## Supporting Components

These components are not involved directly in reconciliation logic but are essential for **scalability, performance, and enterprise integrations**.

**1. argocd-redis**

* Provides a **high-performance cache** for rendered Git state and observed cluster state used by the reconciliation engine.
* Improves scalability and UI/API responsiveness without acting as a source of truth.

> Argo CD polls Git and watches Kubernetes to keep its cache fresh.
> Reconciliation compares **cached desired state with cached live state**, while Git and Kubernetes remain the authoritative sources.

**2. argocd-applicationset-controller**

* Reconciles the **ApplicationSet** resource, enabling dynamic generation of multiple Application objects.
* Commonly used for large-scale, multi-cluster deployments driven by Git-based generators.

**3. argocd-dex-server**

* Integrates Argo CD with **external identity providers** using OIDC or SAML.
* Keeps authentication concerns separate from core reconciliation and authorization logic.

---


## Lab: Installing Argo CD (Local Setup Using KIND)

In this lab, we will install **Argo CD** on a local Kubernetes cluster created using **KIND**.
This setup is intentionally simple and learning-focused. Later in the course, we will move to **production-grade clusters** such as self-managed Kubernetes or **Amazon EKS**.

---

### Prerequisites

Before installing Argo CD, ensure the following tools are available on your system.

---

#### 1. Docker Desktop

Docker Desktop is required because **KIND runs Kubernetes clusters inside Docker containers**.

* Official installation guide:
  [https://docs.docker.com/desktop/](https://docs.docker.com/desktop/)

> **Note:**
> If you are using a **cloud-based Ubuntu VM or any Linux server**, you can install **Docker Engine directly** instead of Docker Desktop, since a GUI is not required.

Verify installation:

```bash
docker version
```

---

#### 2. kubectl

`kubectl` is the Kubernetes CLI used to interact with the cluster.

* Official installation guide:
  [https://kubernetes.io/docs/tasks/tools/](https://kubernetes.io/docs/tasks/tools/)

Verify installation:

```bash
kubectl version --client
```

---

#### 3. Helm

Helm is the Kubernetes package manager.
We will use Helm to install Argo CD in a predictable and repeatable way.

* Official installation guide:
  [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

Verify installation:

```bash
helm version
```

---

#### 4. KIND (Kubernetes IN Docker)

KIND allows us to spin up a **local Kubernetes cluster** for learning and experimentation.

* Installation Reference: https://kind.sigs.k8s.io/docs/user/quick-start#installation

For a detailed walkthrough:

* YouTube: [https://youtu.be/wBF3YCMgZ7U](https://youtu.be/wBF3YCMgZ7U)
* GitHub Notes: [https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2008](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2008)

---

#### 5. IDE (Recommended)

A proper IDE improves productivity when working with YAML and Kubernetes manifests.

* Visual Studio Code:
  [https://code.visualstudio.com/](https://code.visualstudio.com/)

---

### Step 1: Create a KIND Cluster

Let’s start by creating a **simple single-node Kubernetes cluster**.

```bash
kind create cluster --name cwvj-k8s
```

**Explanation:**

* `kind create cluster` creates a Kubernetes cluster using Docker
* `--name cwvj-k8s` gives the cluster a readable name
* By default, KIND creates a **single control-plane node**, which is sufficient for this lab

Verify cluster access:

```bash
kubectl cluster-info
kubectl get nodes
```

---

### Step 2: Add the Argo CD Helm Repository

Argo CD is distributed via an **official Helm chart** maintained by the Argo project.

Open **Artifact Hub**:

* [https://artifacthub.io/](https://artifacthub.io/)

Search for **argo-cd**, filter by **Official**, and note the repository details.

```bash
# Add the official Argo CD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm

# Verify that the repository was added successfully
helm repo list

# Similar to `apt update`, refreshes local cache of chart versions from all added Helm repositories
helm repo update

# Search for Argo-related charts across all added repositories
helm search repo argo

# List available versions of the official Argo CD Helm chart
helm search repo argo/argo-cd --versions
```

---

### Step 3: Create Namespace for Argo CD

We will deploy Argo CD into its own namespace.

```bash
kubectl create ns argocd
```

**Why a separate namespace?**

* Logical segregation of Argo CD components
* Easier RBAC and access control
* Cleaner cluster organization
  In real environments, **only Argo administrators** should have access to this namespace.

---

### Step 4: Install Argo CD Using Helm

Now install Argo CD using Helm.

```bash
helm install my-argo-cd argo/argo-cd --version 9.1.9 -n argocd
```

**Explanation:**

* `helm install` installs a Helm release
* `my-argo-cd` is the release name
* `argo/argo-cd` refers to the official chart
* `--version 9.1.9` pins the exact version for predictability
* `-n argocd` installs Argo CD into the `argocd` namespace

> We could omit the version and install the latest, but **version pinning is critical in CI and production environments** to ensure repeatability.

---

### Step 5: Verify Argo CD Installation

Check that all Argo CD pods are running:

```bash
kubectl get pods -n argocd
```

You should see components such as:

* `argocd-server`
* `argocd-repo-server`
* `argocd-application-controller`
* `argocd-redis`

Wait until all pods are in the `Running` state.

---

### Step 6: Access Argo CD UI (Port Forwarding)

For local learning, we will use **kubectl port-forward**.

First, list services:

```bash
kubectl get svc -n argocd
```

Port-forward the Argo CD server:

```bash
kubectl port-forward service/my-argo-cd-argocd-server -n argocd 8080:443
```

**Explanation:**

* `kubectl port-forward` forwards traffic from your local machine to the cluster
* `service/my-argo-cd-argocd-server` is the Argo CD API/UI service
* `8080` is the local port
* `443` is the HTTPS port exposed by the Argo CD service

Now access Argo CD UI:

```
http://localhost:8080/
```

> ⚠️ **Important Note**
> Port-forwarding is **not used in production**.
> This is purely for learning and quick access.
> Later in the course, we will use **Ingress, LoadBalancers, and production-grade access patterns**.

---

### Step 7: Retrieve Initial Admin Password

Argo CD creates an initial admin password as a Kubernetes secret.

Retrieve it using:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

**Explanation:**

* The password is stored in a Kubernetes Secret
* It is Base64-encoded by default
* We decode it before use

---

### Step 8: Login and Reset Password

1. Open the Argo CD UI:
   `http://localhost:8080/`

2. Login using:

   * **Username:** `admin`
   * **Password:** (retrieved in previous step)

3. Reset the password:

   * Click **User Info**
   * Select **Update Password**
   * Set a new password of your choice

---


## Conclusion

By the end of this lecture, we have established a **clear and correct understanding of Argo CD** in the context of GitOps.

We saw how GitOps principles naturally require a **long-running reconciliation controller**, and how Argo CD fulfills that role by continuously enforcing desired state declared in Git. We explored Argo CD’s internal architecture, understanding the responsibilities of the API Server, Repository Server, and Application Controller, along with supporting components that enable scalability and enterprise readiness.

We also clarified the boundaries between **DevOps engineers, CI systems, Git, and Kubernetes**, reinforcing that CI builds artifacts and updates Git, while Argo CD alone reconciles state inside the cluster. Finally, we installed Argo CD using Helm on a KIND cluster and accessed the UI, preparing the environment for hands-on GitOps workflows.

With Argo CD now running in the cluster, we are ready to move forward and understand **Application CRDs**, which is how we declaratively tell Argo CD *what* to manage and *where* to deploy. That is where GitOps becomes tangible.

---

## References (Official)

The following official documentation was used as reference and is recommended for further reading:

* Argo CD Official Documentation
  [https://argo-cd.readthedocs.io/en/stable/](https://argo-cd.readthedocs.io/en/stable/)

* Argo CD Architecture Overview
  [https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/](https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/)

* Argo CD Declarative Setup (Applications)
  [https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)

* Argo Project (CNCF)
  [https://argoproj.github.io/](https://argoproj.github.io/)

* GitOps Principles (OpenGitOps)
  [https://opengitops.dev/](https://opengitops.dev/)

* Helm Official Documentation
  [https://helm.sh/docs/](https://helm.sh/docs/)



---
