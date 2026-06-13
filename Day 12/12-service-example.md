# Day 12: Kubernetes Services In-Depth
## ClusterIP · NodePort · LoadBalancer · ExternalName | CKA Course 2025

> 📺 [Watch the video](https://www.youtube.com/watch?v=92NB8oQBtnc&ab_channel=CloudWithVarJosh)

---

## Table of Contents

1. [Why Do We Need Services?](#1-why-do-we-need-services)
2. [ClusterIP Service](#2-clusterip-service)
3. [NodePort Service](#3-nodeport-service)
4. [LoadBalancer Service](#4-loadbalancer-service)
5. [ExternalName Service](#5-externalname-service)
6. [Service Comparison Summary](#6-service-comparison-summary)

---

## 1. Why Do We Need Services?

Pods are **ephemeral** — they get new IP addresses every time they restart. Services solve two problems:

- Provide a **stable IP and DNS name** so pods can always be found
- Enable **load balancing** across multiple pod replicas

### Without Services vs With Services

```
❌ Without Services:
User ──► Frontend Pod (IP changes) ──► Backend Pod (IP changes)
         Problem: IPs are dynamic, hardcoding breaks

✅ With Services:
User ──► frontend-svc ──► Frontend Pod ──► backend-svc ──► Backend Pod
         Stable DNS name                   Stable DNS name
```

---

## 2. ClusterIP Service

### What is it?
The **default** service type. Exposes pods **internally within the cluster only**. Pods outside cannot reach it. Perfect for service-to-service communication.

### Office Analogy
> Internal phone extensions inside a building complex — HR dials extension 11 to reach Finance. No outside calls allowed.

### Communication Diagram

```mermaid
flowchart LR
    subgraph "Kubernetes Cluster"
        FP["🖥️ Frontend Pod\nnginx"]

        subgraph "backend-svc · ClusterIP · 9090"
            CS["⚙️ ClusterIP\n10.96.26.155:9090"]
        end

        BP1["📦 Backend Pod 1\n10.244.1.10:5678"]
        BP2["📦 Backend Pod 2\n10.244.2.23:5678"]
        BP3["📦 Backend Pod 3\n10.244.3.14:5678"]
    end

    FP -->|"http://backend-svc:9090\nCoreDNS resolves"| CS
    CS -->|"kube-proxy\nload balances"| BP1
    CS -->|"kube-proxy\nload balances"| BP2
    CS -->|"kube-proxy\nload balances"| BP3

    style CS fill:#AFA9EC,color:#26215C
    style FP fill:#9FE1CB,color:#04342C
    style BP1 fill:#B5D4F4,color:#042C53
    style BP2 fill:#B5D4F4,color:#042C53
    style BP3 fill:#B5D4F4,color:#042C53
```

### Step-by-Step Request Flow

```mermaid
sequenceDiagram
    participant FP as Frontend Pod
    participant DNS as CoreDNS
    participant SVC as backend-svc (ClusterIP)
    participant BP as Backend Pod

    FP->>DNS: Resolve "backend-svc"
    DNS-->>FP: 10.96.26.155
    FP->>SVC: HTTP GET :9090
    SVC->>BP: Forward to :5678 (kube-proxy)
    BP-->>FP: "Hello from Backend"
```

### Key Characteristics

| Property | Value |
|---|---|
| Access scope | Internal cluster only |
| DNS name | `backend-svc` |
| Stable IP | Yes (ClusterIP) |
| Load balancing | Yes (kube-proxy) |
| External access | ❌ No |

### YAML Manifests

#### Frontend Deployment
```yaml
# frontend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
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
          image: nginx
```

#### Backend Deployment
```yaml
# backend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
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
          - "-text=Hello from Backend"
```

#### Backend ClusterIP Service
```yaml
# backend-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP          # Default — can omit this line
  ports:
    - protocol: TCP
      port: 9090           # Service port (what callers use)
      targetPort: 5678     # Container port (what pod listens on)
  selector:
    app: backend
```

### Deploy & Test
```bash
kubectl apply -f frontend-deploy.yaml
kubectl apply -f backend-deploy.yaml
kubectl apply -f backend-svc.yaml

# Verify
kubectl get svc backend-svc
# NAME          TYPE        CLUSTER-IP      PORT(S)    AGE
# backend-svc   ClusterIP   10.96.26.155    9090/TCP   10s

# Test from inside a pod
kubectl exec -it <frontend-pod-name> -- curl http://backend-svc:9090
# Response: Hello from Backend

# Check CoreDNS resolution
kubectl exec -it <frontend-pod-name> -- nslookup backend-svc
```

---

## 3. NodePort Service

### What is it?
Exposes pods **externally** using a **fixed port on every worker node's IP**. Access via `NodeIP:NodePort`. Built on top of ClusterIP.

### Office Analogy
> Each building has a front-desk number. External callers dial the front desk, who connects to the right department extension. If one building is down, caller must manually dial the other building's front desk.

### Communication Diagram

```mermaid
flowchart LR
    User["👤 External User\ncurl 172.18.0.4:31000"]

    subgraph "Kubernetes Cluster"
        subgraph "Worker Node · 172.18.0.4"
            NP["🔀 NodePort\n:31000"]
        end

        subgraph "frontend-svc · ClusterIP · 80"
            CS["⚙️ ClusterIP\n10.96.45.120:80"]
        end

        FP1["📦 Frontend Pod 1\n10.244.1.5:80"]
        FP2["📦 Frontend Pod 2\n10.244.2.10:80"]
        FP3["📦 Frontend Pod 3\n10.244.3.8:80"]
    end

    User -->|"HTTP :31000"| NP
    NP -->|"Forwards to ClusterIP"| CS
    CS -->|"load balance"| FP1
    CS -->|"load balance"| FP2
    CS -->|"load balance"| FP3

    style User fill:#D3D1C7,color:#2C2C2A
    style NP fill:#F0997B,color:#4A1B0C
    style CS fill:#AFA9EC,color:#26215C
    style FP1 fill:#B5D4F4,color:#042C53
    style FP2 fill:#B5D4F4,color:#042C53
    style FP3 fill:#B5D4F4,color:#042C53
```

### Step-by-Step Request Flow

```mermaid
sequenceDiagram
    participant U as External User
    participant WN as Worker Node :31000
    participant SVC as frontend-svc (ClusterIP)
    participant FP as Frontend Pod

    U->>WN: GET http://172.18.0.4:31000
    Note over WN: NodePort receives request
    WN->>SVC: Forward to ClusterIP :80
    SVC->>FP: kube-proxy picks pod
    FP-->>U: NGINX response (200 OK)

    Note over U,WN: Same request also works via<br/>http://172.18.0.5:31000 (other node)
```

### KIND Cluster Setup for NodePort

```yaml
# kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.31.4@sha256:2cb39f7295fe7eafee0842b1052a599a4fb0f8bcf3f83d96c7f4864c357c6c30
    extraPortMappings:
      - containerPort: 31000   # Port inside KIND container
        hostPort: 31000        # Port on your local machine
  - role: worker
    image: kindest/node:v1.31.4@sha256:2cb39f7295fe7eafee0842b1052a599a4fb0f8bcf3f83d96c7f4864c357c6c30
  - role: worker
    image: kindest/node:v1.31.4@sha256:2cb39f7295fe7eafee0842b1052a599a4fb0f8bcf3f83d96c7f4864c357c6c30
```

### Key Characteristics

| Property | Value |
|---|---|
| Access scope | External (any node IP) |
| Port range | 30000–32767 |
| URL format | `http://<NodeIP>:<NodePort>` |
| Built on | ClusterIP |
| Fault tolerance | Manual — if node is down, switch IP |
| External access | ✅ Yes |

### YAML Manifests

#### Frontend Deployment
```yaml
# frontend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
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

#### Frontend NodePort Service
```yaml
# frontend-svc-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80         # ClusterIP service port
      targetPort: 80   # Container port inside the pod
      nodePort: 31000  # External port (30000-32767)
                       # Omit this to let Kubernetes auto-assign
```

### Deploy & Test
```bash
# Create KIND cluster with port mapping first
kind delete cluster --name my-first-cluster
kind create cluster --name my-second-cluster --config kind-cluster.yaml

kubectl apply -f frontend-deploy.yaml
kubectl apply -f frontend-svc-nodeport.yaml

# Verify
kubectl get svc frontend-svc
# NAME           TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# frontend-svc   NodePort   10.96.45.120    <none>        80:31000/TCP   5s

# Test from your local machine
curl http://localhost:31000
# Or using node IP
kubectl get nodes -o wide
curl http://172.18.0.4:31000
```

---

## 4. LoadBalancer Service

### What is it?
Provides a **single public IP** from the cloud provider (AWS ELB, GCP LB, Azure LB). Automatically routes to any healthy node. Built on top of NodePort + ClusterIP.

### Office Analogy
> A call center with a single easy-to-remember number. Caller dials the call center, which automatically routes to any healthy building's front desk, which connects to the right department.

### Communication Diagram

```mermaid
flowchart LR
    User["👤 External User\nhttps://example.com"]
    ELB["☁️ AWS ELB\n34.x.x.x:80\nAuto-provisioned"]

    subgraph "Kubernetes Cluster"
        subgraph "Worker Node 1 · 172.18.0.4"
            NP1["🔀 NodePort :31000"]
        end
        subgraph "Worker Node 2 · 172.18.0.5"
            NP2["🔀 NodePort :31000"]
        end

        subgraph "frontend-svc · ClusterIP"
            CS["⚙️ ClusterIP :80"]
        end

        FP1["📦 Pod 1 :80"]
        FP2["📦 Pod 2 :80"]
        FP3["📦 Pod 3 :80"]
    end

    User -->|"HTTP :80"| ELB
    ELB -->|"auto failover"| NP1
    ELB -->|"auto failover"| NP2
    NP1 -->|"forwards"| CS
    NP2 -->|"forwards"| CS
    CS -->|"load balance"| FP1
    CS -->|"load balance"| FP2
    CS -->|"load balance"| FP3

    style User fill:#D3D1C7,color:#2C2C2A
    style ELB fill:#FAC775,color:#412402
    style NP1 fill:#F0997B,color:#4A1B0C
    style NP2 fill:#F0997B,color:#4A1B0C
    style CS fill:#AFA9EC,color:#26215C
    style FP1 fill:#B5D4F4,color:#042C53
    style FP2 fill:#B5D4F4,color:#042C53
    style FP3 fill:#B5D4F4,color:#042C53
```

### Step-by-Step Request Flow

```mermaid
sequenceDiagram
    participant U as External User
    participant ELB as AWS ELB (34.x.x.x)
    participant NP as NodePort :31000
    participant SVC as ClusterIP :80
    participant FP as Frontend Pod :80

    U->>ELB: GET http://34.x.x.x:80
    Note over ELB: Cloud LB auto-provisioned<br/>by Kubernetes
    ELB->>NP: Route to healthy node
    NP->>SVC: Forward to ClusterIP
    SVC->>FP: kube-proxy picks pod
    FP-->>U: Response (200 OK)

    Note over ELB,NP: If node goes down,<br/>ELB auto-routes to another node
```

### How the Layers Stack

```mermaid
flowchart TB
    LB["LoadBalancer\nPublic IP :80\nCloud provider ELB"]
    NP["NodePort\nAll nodes :31000\nExternal access layer"]
    CI["ClusterIP\nInternal IP :80\nInter-pod routing"]
    PD["Pods\n:80\nActual workload"]

    LB -->|"wraps"| NP
    NP -->|"wraps"| CI
    CI -->|"routes to"| PD

    style LB fill:#FAC775,color:#412402
    style NP fill:#F0997B,color:#4A1B0C
    style CI fill:#AFA9EC,color:#26215C
    style PD fill:#B5D4F4,color:#042C53
```

### Key Characteristics

| Property | Value |
|---|---|
| Access scope | External (public internet) |
| Public IP | Auto-assigned by cloud provider |
| Fault tolerance | Automatic (ELB handles failover) |
| Built on | NodePort + ClusterIP |
| Cost | Cloud LB charges apply |
| Production use | Common — but Ingress preferred for HTTP |

### YAML Manifest

```yaml
# frontend-svc-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80           # External port (what users hit)
      targetPort: 80     # Container port inside pod
      nodePort: 31000    # NodePort used internally (optional — auto-assigned if omitted)
```

### Deploy & Test (EKS)
```bash
kubectl apply -f frontend-svc-loadbalancer.yaml

# Watch ELB being provisioned (takes ~2 min on EKS)
kubectl get svc frontend-svc -w
# NAME           TYPE           CLUSTER-IP     EXTERNAL-IP        PORT(S)        AGE
# frontend-svc   LoadBalancer   10.96.45.120   <pending>          80:31000/TCP   10s
# frontend-svc   LoadBalancer   10.96.45.120   34.x.x.x           80:31000/TCP   2m

# Test with public IP
curl http://34.x.x.x

# Check what AWS created
aws elb describe-load-balancers --region us-east-1
```

> **Production note:** In real production, you typically won't use `LoadBalancer` service directly. Use an **Ingress Controller** instead — it provides path-based routing, SSL termination, and host-based routing with a single ELB.

---

## 5. ExternalName Service

### What is it?
Maps an **internal Kubernetes service name** to an **external DNS name** via a CNAME record. No proxy, no ClusterIP — just pure DNS aliasing. Ideal for connecting to external databases or APIs.

### Office Analogy
> Dialing `111` always connects you to the external IT Support team — you don't need to remember their actual phone number. If they change numbers, only the directory needs updating, not every employee.

### Communication Diagram

```mermaid
flowchart LR
    subgraph "Kubernetes Cluster"
        BP["🖥️ Backend Pod\nconnects to cka-db-svc:3306"]
        DNS["🔍 CoreDNS\nreturns CNAME record"]
        ES["📋 cka-db-svc\nExternalName Service\n(no ClusterIP, no proxy)"]
    end

    subgraph "AWS Cloud"
        RDS["🗄️ Amazon RDS\ncka-db.judhtdmxwly6\n.us-east-1.rds.amazonaws.com"]
    end

    BP -->|"DNS lookup\ncka-db-svc"| DNS
    DNS -->|"CNAME points to"| ES
    ES -->|"redirects to\nexternal DNS"| RDS

    style BP fill:#9FE1CB,color:#04342C
    style DNS fill:#D3D1C7,color:#2C2C2A
    style ES fill:#AFA9EC,color:#26215C
    style RDS fill:#FAC775,color:#412402
```

### Step-by-Step Request Flow

```mermaid
sequenceDiagram
    participant BP as Backend Pod
    participant DNS as CoreDNS
    participant ES as ExternalName Service
    participant RDS as Amazon RDS

    BP->>DNS: Resolve "cka-db-svc"
    DNS->>ES: Lookup ExternalName record
    ES-->>DNS: CNAME → cka-db.xyz.rds.amazonaws.com
    DNS-->>BP: Return RDS DNS name
    BP->>RDS: Connect directly :3306
    RDS-->>BP: DB connection established

    Note over ES: No proxy involved —<br/>pure DNS redirect
```

### Why This Is Useful

```mermaid
flowchart TB
    subgraph "Without ExternalName"
        A1["App Code\nconn = 'cka-db.abc123.us-east-1\n.rds.amazonaws.com:3306'"]
        A2["RDS changes DNS?\nUpdate every deployment,\nevery config map, every env var"]
    end

    subgraph "With ExternalName"
        B1["App Code\nconn = 'cka-db-svc:3306'"]
        B2["RDS changes DNS?\nUpdate only ONE\nExternalName service manifest"]
    end

    A1 --> A2
    B1 --> B2

    style A2 fill:#F7C1C1,color:#501313
    style B2 fill:#C0DD97,color:#173404
```

### Key Characteristics

| Property | Value |
|---|---|
| Access scope | Internal → External |
| Mechanism | CNAME DNS record |
| Proxy | ❌ None |
| ClusterIP | ❌ None |
| Update flexibility | Change only service, not app code |
| Best for | External DBs, legacy APIs, third-party services |

### YAML Manifest

```yaml
# externalname-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: cka-db-svc
  namespace: default
spec:
  type: ExternalName
  externalName: cka-db.judhtdmxwly6.us-east-1.rds.amazonaws.com
  # No selector needed — no pods targeted
  # No ports needed — DNS passthrough
```

### Deploy & Test
```bash
kubectl apply -f externalname-svc.yaml

# Verify
kubectl get svc cka-db-svc
# NAME         TYPE           CLUSTER-IP   EXTERNAL-IP                                    PORT(S)   AGE
# cka-db-svc   ExternalName   <none>       cka-db.judhtdmxwly6.us-east-1.rds.amazonaws.com   <none>    5s

# Test DNS resolution from inside a pod
kubectl run dns-test --image=busybox --rm -it --restart=Never -- sh
nslookup cka-db-svc
# Returns CNAME pointing to RDS endpoint

# App connects using internal name — Kubernetes handles the rest
mysql -h cka-db-svc -P 3306 -u admin -p
```

---

## 6. Service Comparison Summary

### All Four Service Types Side-by-Side

```mermaid
flowchart TB
    User["👤 External User"]
    Internet["🌐 Internet"]

    subgraph "Kubernetes Cluster"
        subgraph "ExternalName"
            EN["📋 cka-db-svc\nCNAME alias only"]
        end
        subgraph "LoadBalancer (wraps NodePort + ClusterIP)"
            LB["☁️ Cloud ELB\nPublic IP"]
            NP["🔀 NodePort\n:31000 on all nodes"]
            CI_LB["⚙️ ClusterIP :80"]
            P_LB["📦 Pods"]
        end
        subgraph "NodePort (wraps ClusterIP)"
            NP2["🔀 NodePort :31000\non specific node"]
            CI_NP["⚙️ ClusterIP :80"]
            P_NP["📦 Pods"]
        end
        subgraph "ClusterIP (internal only)"
            CI["⚙️ ClusterIP\ninternal IP only"]
            P_CI["📦 Pods"]
        end
        ExtSvc["🗄️ External Service\n(RDS, API, etc.)"]
    end

    User -->|"single IP\nauto failover"| LB
    User -->|"NodeIP:Port\nmanual"| NP2
    Internet -.->|"no direct access"| CI
    LB --> NP --> CI_LB --> P_LB
    NP2 --> CI_NP --> P_NP
    CI --> P_CI
    EN -.->|"CNAME redirect"| ExtSvc
```

### Quick Reference Table

| Feature | ClusterIP | NodePort | LoadBalancer | ExternalName |
|---|---|---|---|---|
| External access | ❌ | ✅ | ✅ | ❌ (outbound) |
| Single public IP | ❌ | ❌ | ✅ | N/A |
| Auto failover | N/A | ❌ Manual | ✅ Auto | N/A |
| Port range | Any | 30000-32767 | Any | N/A |
| DNS resolution | Internal | Internal | Internal | CNAME |
| Cloud LB cost | No | No | Yes | No |
| Built on | — | ClusterIP | NodePort+CIP | DNS only |
| Best for | Pod-to-pod | Dev/testing | Production | External services |

### Complete Request Flow Comparison

```mermaid
flowchart LR
    subgraph "ClusterIP"
        direction LR
        C1["Pod A"] -->|"DNS"| C2["ClusterIP svc"] --> C3["Pod B"]
    end

    subgraph "NodePort"
        direction LR
        N1["User"] -->|"NodeIP:31000"| N2["NodePort"] --> N3["ClusterIP"] --> N4["Pod"]
    end

    subgraph "LoadBalancer"
        direction LR
        L1["User"] -->|"PublicIP:80"| L2["ELB"] --> L3["NodePort"] --> L4["ClusterIP"] --> L5["Pod"]
    end

    subgraph "ExternalName"
        direction LR
        E1["Pod"] -->|"CNAME"| E2["ExternalName svc"] -->|"DNS redirect"| E3["External DB"]
    end
```

---

## References

- [KIND Extra Port Mappings](https://kind.sigs.k8s.io/docs/user/configuration/#extra-port-mappings)
- [Kubernetes Services Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [CoreDNS in Kubernetes](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
