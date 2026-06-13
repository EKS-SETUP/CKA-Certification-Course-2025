# Day 14: Kubernetes Namespaces Explained
## Isolation & Resource Management | CKA Course 2025

> 📺 [Watch the video](https://www.youtube.com/watch?v=uDlhbtGy1AU&ab_channel=CloudWithVarJosh)

---

## Table of Contents

1. [What Are Namespaces?](#1-what-are-namespaces)
2. [Default Namespaces in Kubernetes](#2-default-namespaces-in-kubernetes)
3. [Namespace Isolation — How It Works](#3-namespace-isolation--how-it-works)
4. [Cross-Namespace Communication](#4-cross-namespace-communication)
5. [Working with Namespaces](#5-working-with-namespaces)
6. [Deploying Frontend & Backend in a Namespace](#6-deploying-frontend--backend-in-a-namespace)
7. [Testing Namespace Isolation](#7-testing-namespace-isolation)
8. [Setting a Default Namespace](#8-setting-a-default-namespace)
9. [Best Practices](#9-best-practices)
10. [Summary](#10-summary)

---

## 1. What Are Namespaces?

A **namespace** is a logical partition inside a Kubernetes cluster. Think of it as creating separate rooms in a large shared house — each family (workload) gets its own private space.

### The House Analogy

```mermaid
flowchart TB
    subgraph CLUSTER["🏠 Kubernetes Cluster (the house)"]
        subgraph NS1["🚪 app1-ns (room 1)"]
            A1["📦 frontend pod"]
            A2["📦 backend pod"]
            A3["⚙️ backend-svc"]
        end
        subgraph NS2["🚪 app2-ns (room 2)"]
            B1["📦 frontend pod"]
            B2["📦 backend pod"]
            B3["⚙️ backend-svc"]
        end
        subgraph NS3["🚪 kube-system (room 3)"]
            C1["⚙️ kube-dns"]
            C2["⚙️ kube-proxy"]
            C3["⚙️ coredns"]
        end
        subgraph NS4["🚪 default (room 4)"]
            D1["📦 any pods without\nexplicit namespace"]
        end
    end

    style CLUSTER fill:transparent,stroke:#888780,stroke-dasharray:5 4
    style NS1 fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style NS2 fill:transparent,stroke:#534AB7,stroke-dasharray:3 2
    style NS3 fill:transparent,stroke:#993C1D,stroke-dasharray:3 2
    style NS4 fill:transparent,stroke:#888780,stroke-dasharray:3 2
```

> Without namespaces = all families in one open room → no privacy, naming conflicts, no resource control.
> With namespaces = each family has its own room → isolation, security, organisation.

### Why Use Namespaces?

```mermaid
mindmap
  root((Namespaces))
    Security
      Network policies per NS
      RBAC scoped to NS
      Limit access between teams
    Resource Management
      CPU quotas per NS
      Memory limits per NS
      Storage limits per NS
    Environment Isolation
      dev-ns
      test-ns
      prod-ns
    Multi-Tenancy
      Team A namespace
      Team B namespace
      No naming conflicts
    Organisational Clarity
      Group related resources
      Easier kubectl filtering
      Clear ownership
```

---

## 2. Default Namespaces in Kubernetes

```mermaid
flowchart LR
    subgraph CLUSTER["Kubernetes Cluster"]
        subgraph DEF["default"]
            D["Your workloads\nif no NS specified"]
        end
        subgraph KS["kube-system"]
            KS1["kube-dns"]
            KS2["kube-proxy"]
            KS3["etcd, api-server\nscheduler, controller"]
        end
        subgraph KP["kube-public"]
            KP1["Cluster info\nReadable by ALL\neven unauthenticated"]
        end
        subgraph KNL["kube-node-lease"]
            KNL1["Node heartbeats\nControl plane uses\nfor health checks"]
        end
        subgraph LPS["local-path-storage\n(KIND only)"]
            LPS1["Persistent storage\nLocal Path Provisioner"]
        end
    end

    style DEF fill:transparent,stroke:#888780,stroke-dasharray:3 2
    style KS fill:transparent,stroke:#993C1D,stroke-dasharray:3 2
    style KP fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style KNL fill:transparent,stroke:#BA7517,stroke-dasharray:3 2
    style LPS fill:transparent,stroke:#534AB7,stroke-dasharray:3 2
```

| Namespace | Purpose | Touch it? |
|---|---|---|
| `default` | Your workloads go here if no NS is specified | ✅ Yes, for dev/testing |
| `kube-system` | Kubernetes control plane components | ❌ Never modify |
| `kube-public` | Publicly readable cluster info | ❌ Read-only |
| `kube-node-lease` | Node heartbeat objects | ❌ Never modify |
| `local-path-storage` | KIND persistent storage (KIND clusters only) | ❌ Leave alone |

```bash
# See all default namespaces
kubectl get namespaces

# NAME                 STATUS   AGE
# default              Active   2d
# kube-node-lease      Active   2d
# kube-public          Active   2d
# kube-system          Active   2d
# local-path-storage   Active   2d   ← KIND only
```

---

## 3. Namespace Isolation — How It Works

### Same Name, Different Namespace — No Conflict

```mermaid
flowchart TB
    subgraph CLUSTER["Kubernetes Cluster"]
        subgraph APP1["app1-ns"]
            A_SVC["⚙️ backend-svc\nClusterIP: 10.96.1.10"]
            A_POD["📦 backend pod\nimage: app-v1"]
        end
        subgraph APP2["app2-ns"]
            B_SVC["⚙️ backend-svc\nClusterIP: 10.96.2.20"]
            B_POD["📦 backend pod\nimage: app-v2"]
        end
        subgraph DEFAULT["default"]
            T["🧪 test-pod\nbusybox"]
        end
    end

    A_SVC --> A_POD
    B_SVC --> B_POD

    T -->|"curl backend-svc:9090\n❌ FAILS — wrong namespace"| A_SVC
    T -->|"curl backend-svc.app1-ns:9090\n✅ WORKS — full DNS name"| A_SVC

    style APP1 fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style APP2 fill:transparent,stroke:#534AB7,stroke-dasharray:3 2
    style DEFAULT fill:transparent,stroke:#888780,stroke-dasharray:3 2
```

> Both namespaces have a service called `backend-svc` — they don't conflict because they live in separate namespaces.

### Resource Isolation Model

```mermaid
flowchart LR
    subgraph NS["app1-ns"]
        direction TB
        RQ["📊 ResourceQuota\ncpu: 4 cores max\nmemory: 8Gi max"]
        LR["📏 LimitRange\nper pod: 500m CPU\nper pod: 512Mi RAM"]
        NP["🛡️ NetworkPolicy\nblock ingress from\nother namespaces"]

        subgraph PODS["Pods (bounded by above)"]
            P1["📦 frontend"]
            P2["📦 backend"]
        end

        RQ -.->|"enforces"| PODS
        LR -.->|"enforces"| PODS
        NP -.->|"controls traffic"| PODS
    end

    style NS fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style PODS fill:transparent,stroke:#888780,stroke-dasharray:2 2
```

---

## 4. Cross-Namespace Communication

### DNS Format for Cross-Namespace Access

```mermaid
flowchart LR
    subgraph DEFAULT["default namespace"]
        TP["🧪 test-pod"]
    end

    subgraph APP1["app1-ns"]
        SVC["⚙️ backend-svc\n:9090"]
        POD["📦 backend pod\n:5678"]
    end

    subgraph DNS_BOX["CoreDNS resolution"]
        FULL["backend-svc.app1-ns.svc.cluster.local"]
        SHORT["backend-svc.app1-ns"]
    end

    TP -->|"curl backend-svc:9090\n❌ DNS not found"| SVC
    TP -->|"curl backend-svc.app1-ns:9090\n✅ Cross-NS works"| SVC
    SVC --> POD

    FULL -.->|"full FQDN"| SVC
    SHORT -.->|"short form"| SVC

    style DEFAULT fill:transparent,stroke:#888780,stroke-dasharray:3 2
    style APP1 fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style DNS_BOX fill:transparent,stroke:#BA7517,stroke-dasharray:3 2
```

### DNS Name Format Breakdown

```mermaid
flowchart LR
    A["backend-svc"]
    B[".app1-ns"]
    C[".svc"]
    D[".cluster.local"]

    A -->|"service name"| B
    B -->|"namespace"| C
    C -->|"resource type"| D
    D -->|"cluster domain"| E["✅ Full FQDN"]

    style A fill:#9FE1CB,color:#04342C
    style B fill:#AFA9EC,color:#26215C
    style C fill:#FAC775,color:#412402
    style D fill:#B5D4F4,color:#042C53
```

| Scope | Format | Example |
|---|---|---|
| Same namespace | `<service-name>:<port>` | `backend-svc:9090` |
| Cross-namespace (short) | `<service>.<namespace>:<port>` | `backend-svc.app1-ns:9090` |
| Full FQDN | `<service>.<namespace>.svc.cluster.local` | `backend-svc.app1-ns.svc.cluster.local` |

---

## 5. Working with Namespaces

### Create a Namespace

#### Imperative (quick)
```bash
kubectl create namespace app1-ns
```

#### Declarative (recommended for production)
```yaml
# app1-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app1-ns
  labels:
    team: backend
    environment: dev
```
```bash
kubectl apply -f app1-ns.yaml
```

### View, Describe, Delete

```bash
# List all namespaces
kubectl get namespaces
kubectl get ns                     # short alias

# Describe a namespace
kubectl describe namespace app1-ns

# Delete namespace (⚠️ deletes ALL resources inside!)
kubectl delete namespace app1-ns
```

> **Warning:** `kubectl delete namespace app1-ns` removes every pod, service, configmap, secret, and deployment inside it. There is no confirmation prompt.

### Namespace Flags Reference

```mermaid
flowchart LR
    CMD["kubectl get pods"]

    CMD -->|"-n app1-ns"| NS["Shows pods in\napp1-ns only"]
    CMD -->|"--namespace app1-ns"| NS
    CMD -->|"-A"| ALL["Shows pods in\nALL namespaces"]
    CMD -->|"--all-namespaces"| ALL

    style NS fill:#9FE1CB,color:#04342C
    style ALL fill:#AFA9EC,color:#26215C
```

```bash
# Pods in a specific namespace
kubectl get pods -n app1-ns
kubectl get pods --namespace app1-ns

# Pods across ALL namespaces
kubectl get pods -A
kubectl get pods --all-namespaces

# All resources in a namespace
kubectl get all -n app1-ns

# Services across all namespaces
kubectl get services -A
```

---

## 6. Deploying Frontend & Backend in a Namespace

### Architecture

```mermaid
flowchart TB
    subgraph CLUSTER["Kubernetes Cluster"]
        subgraph APP1["app1-ns"]
            subgraph FE_DEP["frontend-deploy (3 replicas)"]
                FP1["📦 frontend pod 1\nnginx:latest"]
                FP2["📦 frontend pod 2\nnginx:latest"]
                FP3["📦 frontend pod 3\nnginx:latest"]
            end
            subgraph BE_DEP["backend-deploy (3 replicas)"]
                BP1["📦 backend pod 1\nhttp-echo:5678"]
                BP2["📦 backend pod 2\nhttp-echo:5678"]
                BP3["📦 backend pod 3\nhttp-echo:5678"]
            end
            FSVC["⚙️ frontend-svc\nClusterIP :80"]
            BSVC["⚙️ backend-svc\nClusterIP :9090"]
        end
    end

    FP1 & FP2 & FP3 -->|"http://backend-svc:9090"| BSVC
    BSVC --> BP1 & BP2 & BP3
    FSVC --> FP1 & FP2 & FP3

    style APP1 fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style FE_DEP fill:transparent,stroke:#888780,stroke-dasharray:2 2
    style BE_DEP fill:transparent,stroke:#888780,stroke-dasharray:2 2
```

### YAML Manifests

#### Namespace
```yaml
# app1-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app1-ns
```

#### Frontend Deployment
```yaml
# frontend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
  namespace: app1-ns        # ← specify namespace here
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend-container
          image: nginx:latest
          ports:
            - containerPort: 80
```

#### Backend Deployment
```yaml
# backend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
  namespace: app1-ns        # ← specify namespace here
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend-container
          image: hashicorp/http-echo
          args:
            - "-text=Hello from Backend in app1-ns"
```

#### Backend ClusterIP Service
```yaml
# backend-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: app1-ns        # ← specify namespace here
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 5678
  selector:
    app: backend
```

### Deploy Commands

```bash
# Step 1: Create the namespace first
kubectl apply -f app1-ns.yaml

# Step 2: Deploy everything into app1-ns
kubectl apply -f frontend-deploy.yaml -n app1-ns
kubectl apply -f backend-deploy.yaml  -n app1-ns
kubectl apply -f backend-svc.yaml     -n app1-ns

# OR — if namespace is in the YAML metadata, just apply directly
kubectl apply -f frontend-deploy.yaml
kubectl apply -f backend-deploy.yaml
kubectl apply -f backend-svc.yaml

# Step 3: Verify everything is running
kubectl get all -n app1-ns

# Expected output:
# NAME                                   READY   STATUS    RESTARTS   AGE
# pod/backend-deploy-xxx-yyy             1/1     Running   0          30s
# pod/frontend-deploy-xxx-yyy            1/1     Running   0          30s
# NAME                  TYPE        CLUSTER-IP      PORT(S)    AGE
# service/backend-svc   ClusterIP   10.96.26.155    9090/TCP   30s
```

---

## 7. Testing Namespace Isolation

### Isolation Test Flow

```mermaid
flowchart TB
    subgraph DEFAULT["default namespace"]
        TP["🧪 test-pod\nbusybox"]
    end

    subgraph APP1["app1-ns"]
        BSVC["⚙️ backend-svc\n:9090"]
        BP["📦 backend pod"]
    end

    subgraph APP1_TEST["app1-ns"]
        TP2["🧪 test-pod\nbusybox"]
        BSVC2["⚙️ backend-svc\n:9090"]
        BP2["📦 backend pod"]
    end

    TP -->|"curl backend-svc:9090\n❌ FAILS — service not in default NS"| BSVC
    TP -->|"curl backend-svc.app1-ns:9090\n✅ WORKS — cross-NS DNS"| BSVC
    BSVC --> BP

    TP2 -->|"curl backend-svc:9090\n✅ WORKS — same namespace"| BSVC2
    BSVC2 --> BP2

    style DEFAULT fill:transparent,stroke:#888780,stroke-dasharray:3 2
    style APP1 fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style APP1_TEST fill:transparent,stroke:#534AB7,stroke-dasharray:3 2
```

### Test 1 — From `default` Namespace (cross-namespace)

```bash
# Launch a test pod in the default namespace
kubectl run test-pod --image=busybox -it --rm --restart=Never -- /bin/sh

# Inside the pod:
curl backend-svc:9090
# ❌ curl: (6) Could not resolve host: backend-svc
# Reason: backend-svc lives in app1-ns, not default

curl http://backend-svc.app1-ns:9090
# ✅ Hello from Backend in app1-ns
# Format: <service-name>.<namespace>:<port>
```

### Test 2 — From `app1-ns` (same namespace)

```bash
# Launch a test pod inside app1-ns
kubectl run test-pod -n app1-ns --image=busybox -it --rm --restart=Never -- /bin/sh

# Inside the pod:
curl backend-svc:9090
# ✅ Hello from Backend in app1-ns
# Works — test-pod and backend-svc are in the same namespace
```

### Summary of Isolation Tests

| Test | Command | Result | Reason |
|---|---|---|---|
| Same NS | `curl backend-svc:9090` | ✅ Works | Service found via short DNS |
| Cross NS (wrong) | `curl backend-svc:9090` from default | ❌ Fails | DNS only resolves within same NS |
| Cross NS (correct) | `curl backend-svc.app1-ns:9090` | ✅ Works | Full namespace-scoped DNS |
| Full FQDN | `curl backend-svc.app1-ns.svc.cluster.local:9090` | ✅ Works | Always works from anywhere |

---

## 8. Setting a Default Namespace

### The Problem Without a Default

```mermaid
flowchart LR
    A["kubectl get pods -n app1-ns"]
    B["kubectl get svc -n app1-ns"]
    C["kubectl logs pod/x -n app1-ns"]
    D["kubectl exec -it pod/x -n app1-ns -- sh"]

    A & B & C & D --> PAIN["😤 Typing -n app1-ns\nevery single time"]

    style PAIN fill:#F7C1C1,color:#501313
```

### The Fix — Set Context Default Namespace

```bash
# Set app1-ns as the default namespace for your current context
kubectl config set-context --current --namespace=app1-ns

# Verify the change
kubectl config get-contexts
# CURRENT   NAME                 CLUSTER   AUTHINFO   NAMESPACE
# *         kind-my-cluster      kind      kind       app1-ns   ← updated

# Now these work WITHOUT -n app1-ns
kubectl get pods          # shows app1-ns pods
kubectl get svc           # shows app1-ns services
kubectl get all           # shows everything in app1-ns
```

### How Context Default Namespace Works

```mermaid
sequenceDiagram
    participant U as You (kubectl)
    participant CTX as kubeconfig context
    participant API as kube-apiserver
    participant NS as app1-ns

    U->>CTX: kubectl config set-context --current --namespace=app1-ns
    CTX-->>U: Context updated

    U->>API: kubectl get pods (no -n flag)
    API->>CTX: Which namespace?
    CTX-->>API: app1-ns (from context default)
    API->>NS: List pods in app1-ns
    NS-->>U: Pod list
```

```bash
# Reset back to default namespace
kubectl config set-context --current --namespace=default

# Check current active namespace
kubectl config view --minify | grep namespace
```

---

## 9. Best Practices

```mermaid
flowchart TB
    BP["Namespace Best Practices"]

    BP --> B1["🏷️ Naming convention\nteam-app-env format\nfrontend-prod-ns\nbackend-dev-ns"]
    BP --> B2["📊 Resource Quotas\nAlways set CPU + memory\nlimits per namespace\nprevent runaway pods"]
    BP --> B3["🛡️ Network Policies\nDefault-deny all ingress\nExplicitly allow only\nneeded traffic"]
    BP --> B4["🔐 RBAC per NS\nTeam A only accesses\nteam-a-ns\nTeam B only team-b-ns"]
    BP --> B5["🌿 Environment isolation\ndev-ns test-ns prod-ns\nNever mix envs\nin one namespace"]
    BP --> B6["⚠️ Careful deletion\nDeletes ALL resources\nNo confirmation prompt\nBackup first"]

    style BP fill:#AFA9EC,color:#26215C
    style B1 fill:#9FE1CB,color:#04342C
    style B2 fill:#9FE1CB,color:#04342C
    style B3 fill:#9FE1CB,color:#04342C
    style B4 fill:#9FE1CB,color:#04342C
    style B5 fill:#9FE1CB,color:#04342C
    style B6 fill:#F7C1C1,color:#501313
```

### ResourceQuota Example (per namespace)
```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: app1-quota
  namespace: app1-ns
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

```bash
kubectl apply -f resource-quota.yaml
kubectl describe resourcequota app1-quota -n app1-ns
```

---

## 10. Summary

### Complete Namespace Communication Map

```mermaid
flowchart TB
    subgraph CLUSTER["Kubernetes Cluster"]
        subgraph KS["kube-system"]
            DNS["🔍 CoreDNS\nresolves all service DNS"]
        end

        subgraph APP1["app1-ns"]
            A_FE["📦 frontend pods"]
            A_BE["📦 backend pods"]
            A_SVC["⚙️ backend-svc :9090"]
        end

        subgraph APP2["app2-ns"]
            B_FE["📦 frontend pods"]
            B_BE["📦 backend pods"]
            B_SVC["⚙️ backend-svc :9090"]
        end

        subgraph DEF["default"]
            TP["🧪 test-pod"]
        end
    end

    A_FE -->|"backend-svc:9090\n✅ same NS"| A_SVC --> A_BE
    B_FE -->|"backend-svc:9090\n✅ same NS"| B_SVC --> B_BE

    TP -->|"backend-svc:9090 ❌"| A_SVC
    TP -->|"backend-svc.app1-ns:9090 ✅"| A_SVC
    TP -->|"backend-svc.app2-ns:9090 ✅"| B_SVC

    DNS -.->|"resolves"| A_SVC
    DNS -.->|"resolves"| B_SVC

    style CLUSTER fill:transparent,stroke:#888780,stroke-dasharray:5 4
    style KS fill:transparent,stroke:#993C1D,stroke-dasharray:3 2
    style APP1 fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style APP2 fill:transparent,stroke:#534AB7,stroke-dasharray:3 2
    style DEF fill:transparent,stroke:#888780,stroke-dasharray:3 2
```

### Key Takeaways

| Concept | Key Point |
|---|---|
| Namespace = logical room | Isolates resources, names, and access within a cluster |
| Same NS DNS | `service-name:port` — short form works |
| Cross NS DNS | `service.namespace:port` — namespace scope required |
| Full FQDN | `service.namespace.svc.cluster.local` — always works |
| Default context NS | `kubectl config set-context --current --namespace=X` |
| Deletion warning | `kubectl delete namespace` removes ALL resources inside |
| kube-system | Never touch — holds control plane components |

---

## References
- [Kubernetes Namespaces Documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [ResourceQuota Documentation](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [NetworkPolicy Documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
