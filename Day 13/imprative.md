# Day 13: Imperative Commands — Deployments, Services & Troubleshooting
## Senior Engineer Practical Guide | CKA Course 2025

> 📺 [Watch the video](https://www.youtube.com/watch?v=ZDFqVFbX0Ic&ab_channel=CloudWithVarJosh)

---

## Table of Contents

1. [Why Imperative Commands Matter in Real Production](#1-why-imperative-commands-matter-in-real-production)
2. [dry-run Deep Dive — Client vs Server](#2-dry-run-deep-dive--client-vs-server)
3. [Backend — Deployment + ClusterIP Service](#3-backend--deployment--clusterip-service)
4. [Frontend — Deployment + NodePort Service](#4-frontend--deployment--nodeport-service)
5. [Full Architecture — How Everything Communicates](#5-full-architecture--how-everything-communicates)
6. [Labels — What Senior Engineers Know](#6-labels--what-senior-engineers-know)
7. [Test Pods for Troubleshooting](#7-test-pods-for-troubleshooting)
8. [Real-World Troubleshooting Workflows](#8-real-world-troubleshooting-workflows)
9. [Imperative vs Declarative — When to Use What](#9-imperative-vs-declarative--when-to-use-what)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. Why Imperative Commands Matter in Real Production

In a real job, you will switch between imperative and declarative constantly. Here is when each is used:

```mermaid
flowchart LR
    subgraph IMPER["⚡ Imperative — Use When"]
        I1["🔥 Production incident\nquick test pod to check\nDB connectivity RIGHT NOW"]
        I2["📋 CKA exam\ngenerate YAML in 10 seconds\nnot 3 minutes"]
        I3["🧪 Local dev testing\nthrow up a service\ncheck it works and delete"]
        I4["🔍 Debugging\ncurl from inside cluster\ncheck DNS resolution"]
    end

    subgraph DECLAR["📄 Declarative — Use When"]
        D1["🏭 Production deploys\ngit-tracked YAML\nreviewed via PR"]
        D2["♻️ Repeatable infra\nsame manifest applied\nto dev/test/prod"]
        D3["🤝 Team collaboration\nothers can review\nwhat gets deployed"]
        D4["🔄 GitOps / ArgoCD\nmanifests as source\nof truth"]
    end

    style IMPER fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style DECLAR fill:transparent,stroke:#534AB7,stroke-dasharray:3 2
```

> **Senior engineer mindset:** Use imperative to *generate* the YAML fast, then commit the YAML to git. Best of both worlds.

---

## 2. dry-run Deep Dive — Client vs Server

Understanding `--dry-run` properly separates juniors from seniors.

```mermaid
flowchart TD
    CMD["kubectl create deployment ... --dry-run=??? -o yaml"]

    CMD --> CLIENT["--dry-run=client\nValidation happens LOCALLY\nin kubectl binary only\nNever contacts API server"]
    CMD --> SERVER["--dry-run=server\nRequest goes to API server\nFull admission webhook validation\nCRD schema checks\nBut nothing persisted"]

    CLIENT --> C_USE["✅ Use for:\n• Quick YAML generation\n• Offline environments\n• CI pipelines without cluster access"]
    CLIENT --> C_MISS["❌ Misses:\n• Admission controller rejections\n• Custom CRD validation\n• Namespace existence checks"]

    SERVER --> S_USE["✅ Use for:\n• Pre-flight before real apply\n• Validating webhook policies\n• OPA/Kyverno policy testing"]
    SERVER --> S_MISS["❌ Requires:\n• Live cluster access\n• Correct RBAC permissions"]

    style CLIENT fill:#9FE1CB,color:#04342C
    style SERVER fill:#AFA9EC,color:#26215C
    style C_MISS fill:#F7C1C1,color:#501313
    style S_MISS fill:#FAC775,color:#412402
```

```mermaid
sequenceDiagram
    participant K as kubectl
    participant API as kube-apiserver
    participant ADM as Admission Webhooks
    participant ETCD as etcd

    Note over K,ETCD: --dry-run=client
    K->>K: Validate locally (schema only)
    K-->>K: Print YAML — API never contacted

    Note over K,ETCD: --dry-run=server
    K->>API: Submit request (dry-run flag set)
    API->>ADM: Run admission controllers
    ADM-->>API: Approved / Rejected
    API-->>K: Return result — nothing written to etcd

    Note over K,ETCD: Real apply (no dry-run)
    K->>API: Submit request
    API->>ADM: Run admission controllers
    ADM-->>API: Approved
    API->>ETCD: Persist object
    ETCD-->>K: Object created
```

### Real-World Example — Catching a Policy Violation

```bash
# --dry-run=client passes (kubectl doesn't know your OPA policies)
kubectl create deployment test --image=nginx --dry-run=client -o yaml
# ✅ Outputs YAML — looks fine locally

# --dry-run=server catches the Kyverno/OPA policy rejection
kubectl create deployment test --image=nginx --dry-run=server
# ❌ Error: admission webhook "kyverno.io" denied: image must use digest, not tag

# This saves you from a failed real deployment
```

---

## 3. Backend — Deployment + ClusterIP Service

### What ClusterIP Means in Practice

```mermaid
flowchart LR
    subgraph CLUSTER["Kubernetes Cluster"]
        subgraph BE_DEP["backend-deploy · 3 replicas"]
            BP1["📦 backend pod 1\nhashicorp/http-echo\n:5678"]
            BP2["📦 backend pod 2\nhashicorp/http-echo\n:5678"]
            BP3["📦 backend pod 3\nhashicorp/http-echo\n:5678"]
        end

        BSVC["⚙️ backend-svc\nClusterIP\n10.96.x.x:9090"]

        FP["📦 frontend pod\nnginx\ncalls backend-svc:9090"]
        TP["🧪 test-pod\nbusybox/nginx\nfor debugging"]
    end

    EXT["🌍 External traffic\nInternet / Users"]

    FP -->|"http://backend-svc:9090\nCoreDNS resolves internally"| BSVC
    TP -->|"curl backend-svc:9090\ntroubleshooting"| BSVC
    BSVC -->|"kube-proxy\nload balances"| BP1
    BSVC -->|"kube-proxy\nload balances"| BP2
    BSVC -->|"kube-proxy\nload balances"| BP3
    EXT -.-x|"❌ BLOCKED\nClusterIP not\nexternally accessible"| BSVC

    style BSVC fill:#AFA9EC,color:#26215C
    style BP1 fill:#B5D4F4,color:#042C53
    style BP2 fill:#B5D4F4,color:#042C53
    style BP3 fill:#B5D4F4,color:#042C53
    style EXT fill:#F7C1C1,color:#501313
    style CLUSTER fill:transparent,stroke:#888780,stroke-dasharray:4 3
```

### Port Mapping — What Each Port Means

```mermaid
flowchart LR
    CALLER["Caller\n(frontend pod\nor test-pod)"]
    SVC["backend-svc\nport: 9090"]
    POD["backend pod\ncontainerPort: 5678"]

    CALLER -->|"connects to :9090\n(service port)"| SVC
    SVC -->|"forwards to :5678\n(targetPort = containerPort)"| POD

    style SVC fill:#AFA9EC,color:#26215C
    style POD fill:#B5D4F4,color:#042C53
```

> **Senior tip:** `port` (9090) is what callers use. `targetPort` (5678) is what the container actually listens on. They don't need to match — and often shouldn't, to allow service-layer flexibility.

### Step 1 — Generate backend YAML imperatively

```bash
# Generate the deployment YAML — does NOT create anything yet
kubectl create deployment backend-deploy \
  --image=hashicorp/http-echo \
  --replicas=3 \
  --port=5678 \
  --dry-run=client -o yaml > backend-deploy.yaml
```

**What each flag does in a real context:**

| Flag | Value | Why |
|---|---|---|
| `--image` | `hashicorp/http-echo` | Lightweight HTTP server — responds with static text. Perfect for API mock |
| `--replicas` | `3` | HA baseline — survive 1 pod crash with 2 still serving |
| `--port` | `5678` | Documents the container port in the spec (informational, not enforced) |
| `--dry-run=client` | — | No cluster call — just prints YAML to stdout |
| `-o yaml > file` | — | Redirect to file for editing before applying |

### Step 2 — Append service to same file

```bash
kubectl expose deployment backend-deploy \
  --type=ClusterIP \
  --port=9090 \
  --target-port=5678 \
  --name=backend-svc \
  --dry-run=client -o yaml >> backend-deploy.yaml
```

> Note: `>>` appends (adds to end of file). `>` overwrites. Use `>>` here because the deployment is already in the file.

### Step 3 — Manual edits required (what kubectl cannot do imperatively)

```bash
# Open the file and make these 3 edits:
vim backend-deploy.yaml
```

**Edit 1:** Add `---` separator between deployment and service objects

**Edit 2:** Add the `args` field to make http-echo respond with a message

**Edit 3:** Clean up any auto-generated `status: {}` and `creationTimestamp: null` fields (optional but clean)

### Final `backend-deploy.yaml` (from your uploaded file)

```yaml
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
        - image: hashicorp/http-echo
          name: http-echo
          ports:
            - containerPort: 5678     # ← what the container listens on
          args:
            - "-text=Hello from Backend"   # ← manually added after dry-run

---

apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  ports:
  - port: 9090           # ← what callers use
    protocol: TCP
    targetPort: 5678     # ← forwarded to container port
  selector:
    app: backend         # ← matches pod label app: backend
  type: ClusterIP        # ← internal only, no external access
```

### Apply & Verify

```bash
# Apply both deployment and service from one file
kubectl apply -f backend-deploy.yaml

# Verify deployment
kubectl get deployment backend-deploy
# NAME             READY   UP-TO-DATE   AVAILABLE
# backend-deploy   3/3     3            3

# Verify pods with labels visible
kubectl get pods --show-labels -l app=backend
# NAME                              READY   STATUS    LABELS
# backend-deploy-xxx-aaa            1/1     Running   app=backend,pod-template-hash=xxx
# backend-deploy-xxx-bbb            1/1     Running   app=backend,pod-template-hash=xxx
# backend-deploy-xxx-ccc            1/1     Running   app=backend,pod-template-hash=xxx

# Verify service and ClusterIP assigned
kubectl get svc backend-svc
# NAME          TYPE        CLUSTER-IP      PORT(S)    AGE
# backend-svc   ClusterIP   10.96.26.155    9090/TCP   10s

# Describe to see endpoint mapping
kubectl describe svc backend-svc
# Port:       9090/TCP
# TargetPort: 5678/TCP
# Endpoints:  10.244.1.5:5678, 10.244.2.8:5678, 10.244.3.3:5678
```

---

## 4. Frontend — Deployment + NodePort Service

### What NodePort Means in Practice

```mermaid
flowchart LR
    USER["👤 External User\nbrowser / curl"]
    KIND["🖥️ localhost:31000\n(KIND hostPort mapping)"]
    NODE["Worker Node\n172.18.0.4:31000\nNodePort"]

    subgraph CLUSTER["Kubernetes Cluster"]
        FSVC["⚙️ frontend-svc\nNodePort\n:31000 → :80"]
        FP1["📦 frontend pod 1\nnginx :80"]
        FP2["📦 frontend pod 2\nnginx :80"]
        FP3["📦 frontend pod 3\nnginx :80"]
    end

    USER -->|"curl localhost:31000\n(KIND)"| KIND
    KIND -->|"port-mapped to\ncontainer:31000"| NODE
    NODE -->|"NodePort forwards\nto ClusterIP internally"| FSVC
    FSVC -->|"kube-proxy\nload balances"| FP1
    FSVC -->|"kube-proxy\nload balances"| FP2
    FSVC -->|"kube-proxy\nload balances"| FP3

    style FSVC fill:#F0997B,color:#4A1B0C
    style FP1 fill:#B5D4F4,color:#042C53
    style FP2 fill:#B5D4F4,color:#042C53
    style FP3 fill:#B5D4F4,color:#042C53
    style CLUSTER fill:transparent,stroke:#888780,stroke-dasharray:4 3
```

### NodePort Port Mapping Breakdown

```mermaid
flowchart LR
    A["External request\n:31000"]
    B["nodePort: 31000\nListens on every\nworker node"]
    C["port: 80\nClusterIP service port\ninternal layer"]
    D["targetPort: 80\nContainer port\nnginx listens here"]
    E["📦 nginx pod\nserves response"]

    A --> B --> C --> D --> E

    style A fill:#D3D1C7,color:#2C2C2A
    style B fill:#F0997B,color:#4A1B0C
    style C fill:#AFA9EC,color:#26215C
    style D fill:#9FE1CB,color:#04342C
    style E fill:#B5D4F4,color:#042C53
```

### Step 1 — Generate frontend YAML imperatively

```bash
kubectl create deployment frontend-deploy \
  --image=nginx \
  --replicas=3 \
  --port=80 \
  --dry-run=client -o yaml > frontend-deploy.yaml
```

### Step 2 — Append NodePort service

```bash
kubectl expose deployment frontend-deploy \
  --type=NodePort \
  --port=80 \
  --target-port=80 \
  --name=frontend-svc \
  --dry-run=client -o yaml >> frontend-deploy.yaml
```

> **Important:** `--node-port` flag does NOT exist in `kubectl expose`. You **must** manually add `nodePort: 31000` to the YAML after generation. This is a common CKA trap.

### Step 3 — Manual edits required

```bash
vim frontend-deploy.yaml
# Add --- separator between deployment and service
# Add nodePort: 31000 under the ports section of the service
```

### Final `frontend-deploy.yaml` (from your uploaded file)

```yaml
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
        - image: nginx
          name: nginx
          ports:
            - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  ports:
  - port: 80             # ← ClusterIP layer port
    protocol: TCP
    targetPort: 80       # ← nginx container port
    nodePort: 31000      # ← manually added! cannot set imperatively
  selector:
    app: frontend        # ← matches pod label app: frontend
  type: NodePort         # ← external access via node IP
```

### Apply & Verify

```bash
kubectl apply -f frontend-deploy.yaml

# Verify deployment
kubectl get deployment frontend-deploy
# NAME              READY   UP-TO-DATE   AVAILABLE
# frontend-deploy   3/3     3            3

# Verify NodePort service
kubectl get svc frontend-svc
# NAME           TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# frontend-svc   NodePort   10.96.45.120    <none>        80:31000/TCP   5s
#                                                             ↑ port:nodePort format

# Test from your local machine (KIND)
curl http://localhost:31000
# <!DOCTYPE html><html>...nginx welcome page...

# Get node IP for direct NodePort access
kubectl get nodes -o wide
curl http://172.18.0.4:31000
```

---

## 5. Full Architecture — How Everything Communicates

### Complete Request Flow Diagram

```mermaid
flowchart TD
    USER["👤 External User\ncurl localhost:31000"]

    subgraph KIND_HOST["KIND Host Machine"]
        HP["hostPort: 31000\nKIND port mapping"]
    end

    subgraph CLUSTER["Kubernetes Cluster"]
        subgraph FE_LAYER["Frontend Layer"]
            FSVC["⚙️ frontend-svc\nNodePort :31000→:80\nClusterIP: 10.96.45.120"]
            FP1["📦 nginx pod 1\n:80"]
            FP2["📦 nginx pod 2\n:80"]
            FP3["📦 nginx pod 3\n:80"]
        end

        subgraph BE_LAYER["Backend Layer"]
            BSVC["⚙️ backend-svc\nClusterIP :9090→:5678\nClusterIP: 10.96.26.155"]
            BP1["📦 http-echo pod 1\n:5678\n'Hello from Backend'"]
            BP2["📦 http-echo pod 2\n:5678\n'Hello from Backend'"]
            BP3["📦 http-echo pod 3\n:5678\n'Hello from Backend'"]
        end

        DNS["🔍 CoreDNS\nresolves backend-svc\n→ 10.96.26.155"]

        TP["🧪 test-pod\nbusybox or nginx\nfor debugging"]
    end

    USER -->|"1 — HTTP :31000"| HP
    HP -->|"2 — forward to\nworker node :31000"| FSVC
    FSVC -->|"3 — kube-proxy\npicks a pod"| FP1
    FP1 -->|"4 — nginx needs\nbackend data\nDNS lookup"| DNS
    DNS -->|"5 — returns\n10.96.26.155"| FP1
    FP1 -->|"6 — HTTP to\nbackend-svc:9090"| BSVC
    BSVC -->|"7 — kube-proxy\npicks a backend pod"| BP2
    BP2 -->|"8 — 'Hello from Backend'"| FP1
    FP1 -->|"9 — assembled response\nback to user"| USER

    TP -->|"debug: curl backend-svc:9090"| BSVC
    TP -->|"debug: curl frontend-svc:80"| FSVC

    style KIND_HOST fill:transparent,stroke:#BA7517,stroke-dasharray:3 2
    style CLUSTER fill:transparent,stroke:#888780,stroke-dasharray:4 3
    style FE_LAYER fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style BE_LAYER fill:transparent,stroke:#534AB7,stroke-dasharray:3 2
    style FSVC fill:#F0997B,color:#4A1B0C
    style BSVC fill:#AFA9EC,color:#26215C
    style DNS fill:#FAC775,color:#412402
    style TP fill:#D3D1C7,color:#2C2C2A
```

### Sequence Diagram — Full Request Lifecycle

```mermaid
sequenceDiagram
    participant U as User (browser)
    participant NP as NodePort :31000
    participant FP as Frontend Pod (nginx)
    participant DNS as CoreDNS
    participant BS as backend-svc ClusterIP
    participant BP as Backend Pod (http-echo)

    U->>NP: GET http://localhost:31000
    NP->>FP: kube-proxy forwards to nginx pod
    FP->>DNS: Resolve "backend-svc"
    DNS-->>FP: 10.96.26.155 (ClusterIP)
    FP->>BS: GET http://backend-svc:9090
    BS->>BP: kube-proxy forwards to http-echo pod
    BP-->>BS: "Hello from Backend"
    BS-->>FP: Response body
    FP-->>U: Rendered HTML with backend data
```

---

## 6. Labels — What Senior Engineers Know

Labels are the glue of Kubernetes. Services find pods using labels. Understanding the full label chain is critical.

### Label Selector Chain

```mermaid
flowchart LR
    SVC["Service\nselector:\n  app: backend"]
    DEP["Deployment\nselector.matchLabels:\n  app: backend"]
    RS["ReplicaSet\n(auto-created by Deployment)\nselector.matchLabels:\n  app: backend"]
    POD["Pod\nlabels:\n  app: backend\n  pod-template-hash: xxx"]

    SVC -->|"finds pods via\nlabel selector"| POD
    DEP -->|"manages via\nmatchLabels"| RS
    RS -->|"creates pods\nwith these labels"| POD

    style SVC fill:#AFA9EC,color:#26215C
    style DEP fill:#9FE1CB,color:#04342C
    style RS fill:#FAC775,color:#412402
    style POD fill:#B5D4F4,color:#042C53
```

### What You Can and Cannot Do with Labels Imperatively

```mermaid
flowchart LR
    subgraph CAN["✅ Can do imperatively"]
        C1["kubectl run nginx-pod\n--labels='app=nginx,env=prod'\nLabels on standalone pod"]
        C2["kubectl label pod my-pod\nenv=production\nAdd label to existing pod"]
        C3["kubectl get pods\n-l app=backend\nFilter by label"]
    end

    subgraph CANNOT["❌ Cannot do imperatively"]
        N1["kubectl create deployment ...\n--pod-labels='env=prod'\nPod template labels\nin a deployment"]
        N2["Must edit YAML manually\nunder template.metadata.labels"]
    end

    style CAN fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style CANNOT fill:transparent,stroke:#993C1D,stroke-dasharray:3 2
```

### Creating Pods with Multiple Labels

```bash
# Create a pod with multiple labels imperatively
kubectl run nginx-pod \
  --image=nginx \
  --restart=Never \
  --labels="app=nginx,env=production,tier=frontend"

# Verify labels
kubectl get pods --show-labels
# NAME        READY   STATUS    LABELS
# nginx-pod   1/1     Running   app=nginx,env=production,tier=frontend

# Filter by a single label
kubectl get pods -l env=production

# Filter by multiple labels (AND logic)
kubectl get pods -l app=nginx,env=production

# Filter using set-based selector
kubectl get pods -l 'env in (production, staging)'

# Show which pods a service is targeting
kubectl get endpoints backend-svc
# NAME          ENDPOINTS
# backend-svc   10.244.1.5:5678,10.244.2.8:5678,10.244.3.3:5678
# ↑ If this is empty, your service selector doesn't match any pod labels
```

> **Senior debug tip:** If a service has no Endpoints, the label selector doesn't match. Always check `kubectl get endpoints <svc-name>` before chasing networking issues.

---

## 7. Test Pods for Troubleshooting

### When and Why to Use Test Pods

```mermaid
flowchart LR
    INCIDENT["🚨 Production incident\nApp can't reach DB"]

    INCIDENT --> Q1["Is it DNS?\nnslookup backend-svc"]
    INCIDENT --> Q2["Is it the service?\ncurl backend-svc:9090"]
    INCIDENT --> Q3["Is it the pod?\ncurl 10.244.x.x:5678 directly"]
    INCIDENT --> Q4["Is it network policy?\ntest from different namespace"]

    Q1 & Q2 & Q3 & Q4 --> TP["🧪 Spin up test-pod\nin the right namespace\nwith the right image\nto answer each question"]

    style INCIDENT fill:#F7C1C1,color:#501313
    style TP fill:#9FE1CB,color:#04342C
```

### Generate Test Pod YAML Quickly

```bash
# Generate test pod YAML (useful for CKA)
kubectl run my-nginx \
  --image=nginx \
  --restart=Never \
  --labels=app=test-pod \
  --dry-run=client -o yaml > test-pod.yaml

# Review it
cat test-pod.yaml
```

Output:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: test-pod
  name: my-nginx
spec:
  containers:
  - image: nginx
    name: my-nginx
  restartPolicy: Never
```

### Temporary Interactive Pod — Most Used in Production

```bash
# Spin up busybox — lightweight, has wget, nslookup, ping
kubectl run test-pod \
  --rm -it \
  --restart=Never \
  --image=busybox \
  -- /bin/sh

# Inside the pod — run your checks:
nslookup backend-svc              # DNS resolution
wget -qO- http://backend-svc:9090 # HTTP check
wget -qO- http://frontend-svc:80  # Frontend check
ping 10.244.1.5                   # Direct pod IP ping
exit  # ← pod auto-deleted because of --rm
```

```bash
# Use nginx instead if you need curl (busybox wget vs curl)
kubectl run test-pod \
  --rm -it \
  --restart=Never \
  --image=nginx \
  -- /bin/bash

# Inside:
curl http://backend-svc:9090
curl http://backend-svc.default.svc.cluster.local:9090  # FQDN
exit
```

### Test Pod Communication Flow

```mermaid
sequenceDiagram
    participant TP as test-pod (busybox)
    participant DNS as CoreDNS
    participant SVC as backend-svc
    participant BP as Backend Pod

    Note over TP: kubectl run test-pod --rm -it --image=busybox

    TP->>DNS: nslookup backend-svc
    DNS-->>TP: 10.96.26.155

    TP->>SVC: wget -qO- http://backend-svc:9090
    SVC->>BP: forward to :5678
    BP-->>SVC: "Hello from Backend"
    SVC-->>TP: Response body

    Note over TP: exit → --rm deletes pod automatically
```

---

## 8. Real-World Troubleshooting Workflows

### Workflow 1 — Service Not Reachable

```mermaid
flowchart TD
    START["🚨 curl backend-svc:9090 fails"]

    START --> S1["kubectl get svc backend-svc\nDoes service exist?"]
    S1 -->|"No"| FIX1["kubectl apply -f backend-deploy.yaml\nRecreate service"]
    S1 -->|"Yes"| S2["kubectl get endpoints backend-svc\nAny endpoints listed?"]

    S2 -->|"Empty / none"| S3["Label mismatch!\nkubectl get pods --show-labels\nCompare with service selector"]
    S3 --> FIX2["Fix label in deployment\nor fix selector in service"]

    S2 -->|"Has endpoints"| S4["kubectl exec -it test-pod\ncurl <endpoint-ip>:5678 directly\nDoes direct pod IP work?"]

    S4 -->|"Yes"| FIX3["kube-proxy issue\nRestart kube-proxy pod\nor check iptables rules"]
    S4 -->|"No"| FIX4["Container issue\nkubectl logs <backend-pod>\nkubectl describe pod <backend-pod>"]

    style START fill:#F7C1C1,color:#501313
    style FIX1 fill:#9FE1CB,color:#04342C
    style FIX2 fill:#9FE1CB,color:#04342C
    style FIX3 fill:#9FE1CB,color:#04342C
    style FIX4 fill:#9FE1CB,color:#04342C
```

### Workflow 2 — NodePort Not Accessible from Outside

```mermaid
flowchart TD
    START["🚨 curl localhost:31000 fails"]

    START --> C1["kubectl get svc frontend-svc\nCheck nodePort field is 31000"]
    C1 -->|"nodePort missing"| F1["Edit YAML\nadd nodePort: 31000\nkubectl apply -f frontend-deploy.yaml"]
    C1 -->|"nodePort set"| C2["KIND cluster?\nCheck kind-cluster.yaml\nextraPortMappings set?"]

    C2 -->|"Missing hostPort mapping"| F2["Recreate KIND cluster\nwith extraPortMappings:\n- containerPort: 31000\n  hostPort: 31000"]
    C2 -->|"Port mapping exists"| C3["kubectl get pods -l app=frontend\nAre frontend pods Running?"]

    C3 -->|"Pods not Running"| F3["kubectl describe pod <name>\nCheck image pull / OOMKilled / CrashLoop"]
    C3 -->|"Pods Running"| C4["kubectl get endpoints frontend-svc\nAre endpoints populated?"]
    C4 -->|"Empty"| F4["Label selector mismatch\nFix selector in service YAML"]
    C4 -->|"Has endpoints"| F5["curl <node-ip>:31000 directly\nFirewall / security group blocking?"]

    style START fill:#F7C1C1,color:#501313
    style F1 fill:#9FE1CB,color:#04342C
    style F2 fill:#9FE1CB,color:#04342C
    style F3 fill:#9FE1CB,color:#04342C
    style F4 fill:#9FE1CB,color:#04342C
    style F5 fill:#FAC775,color:#412402
```

### Cheat Sheet — Senior Debug Commands

```bash
# ── SERVICE DEBUGGING ─────────────────────────────────────
# Does the service exist and what type?
kubectl get svc -A

# Are pods actually being targeted?
kubectl get endpoints backend-svc
# Empty = label mismatch between service selector and pod labels

# What labels do pods have?
kubectl get pods --show-labels -l app=backend

# Does the service selector match?
kubectl describe svc backend-svc | grep -A3 Selector

# ── POD DEBUGGING ─────────────────────────────────────────
# Check pod status
kubectl get pods -o wide

# Check events (crash reasons, image pull failures)
kubectl describe pod <pod-name> | tail -20

# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous   # previous container (after crash)

# ── NETWORK DEBUGGING ─────────────────────────────────────
# Exec into a running pod
kubectl exec -it <pod-name> -- /bin/sh

# One-liner network test without entering pod
kubectl exec -it <frontend-pod> -- curl http://backend-svc:9090

# DNS check
kubectl exec -it <any-pod> -- nslookup backend-svc

# Direct pod IP bypass (skips service layer)
kubectl exec -it <any-pod> -- curl http://10.244.1.5:5678
```

---

## 9. Imperative vs Declarative — When to Use What

### The Recommended Senior Workflow

```mermaid
flowchart LR
    subgraph FAST["⚡ Fast Generation (Imperative)"]
        I1["kubectl create deployment ...\n--dry-run=client -o yaml > deploy.yaml"]
        I2["kubectl expose deployment ...\n--dry-run=client -o yaml >> deploy.yaml"]
    end

    subgraph EDIT["✏️ Manual Polish"]
        E1["Add --- separator\nAdd args / env vars\nAdd nodePort\nAdd resource limits\nClean up nulls"]
    end

    subgraph COMMIT["📦 Declarative Apply"]
        C1["git add deploy.yaml\ngit commit -m 'add backend deployment'\ngit push"]
        C2["kubectl apply -f deploy.yaml\n(or ArgoCD syncs it)"]
    end

    FAST --> EDIT --> COMMIT

    style FAST fill:transparent,stroke:#1D9E75,stroke-dasharray:3 2
    style EDIT fill:transparent,stroke:#BA7517,stroke-dasharray:3 2
    style COMMIT fill:transparent,stroke:#534AB7,stroke-dasharray:3 2
```

### Things You Always Need to Add Manually After `--dry-run`

| What | Why kubectl can't do it |
|---|---|
| `---` YAML separator | `>>` appends without separator |
| `args:` in container spec | `kubectl create deployment` has no `--args` flag |
| `nodePort: 31000` | `kubectl expose` has no `--node-port` flag |
| `resources.requests/limits` | Not exposed as a flag in create/expose |
| `env:` variables | `kubectl create deployment` has `--env` but limited support |
| `namespace: app1-ns` in metadata | Must be specified with `-n` flag or added manually |
| `imagePullPolicy` | Not a flag — manual edit |
| `livenessProbe / readinessProbe` | Not a flag — manual edit |

---

## 10. Key Takeaways

```mermaid
flowchart TB
    KT["Day 13 Key Takeaways"]

    KT --> A["🔑 --dry-run=client\nValidates locally, never hits API\nBest for YAML generation"]
    KT --> B["🔑 --dry-run=server\nFull API validation incl webhooks\nBest pre-deploy check"]
    KT --> C["🔑 >> appends\n> overwrites\nKnow when to use each"]
    KT --> D["🔑 nodePort cannot be set\nimperatively\nAlways manual edit"]
    KT --> E["🔑 Empty Endpoints = label mismatch\nFirst check in any\nservice connectivity issue"]
    KT --> F["🔑 --rm on test pods\nAuto-cleanup\nNo orphaned debug pods"]
    KT --> G["🔑 Generate fast\nedit carefully\napply declaratively\nBest of both worlds"]

    style KT fill:#AFA9EC,color:#26215C
    style A fill:#9FE1CB,color:#04342C
    style B fill:#9FE1CB,color:#04342C
    style C fill:#9FE1CB,color:#04342C
    style D fill:#FAC775,color:#412402
    style E fill:#F7C1C1,color:#501313
    style F fill:#9FE1CB,color:#04342C
    style G fill:#B5D4F4,color:#042C53
```

---

## Appendix — Uploaded YAML Files Reference

### `backend-deploy.yaml`
```yaml
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
        - image: hashicorp/http-echo
          name: http-echo
          ports:
            - containerPort: 5678
          args:
            - "-text=Hello from Backend"

---

apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  ports:
  - port: 9090
    protocol: TCP
    targetPort: 5678
  selector:
    app: backend
  type: ClusterIP
```

### `frontend-deploy.yaml`
```yaml
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
        - image: nginx
          name: nginx
          ports:
            - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 31000
  selector:
    app: frontend
  type: NodePort
```

---

## References
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [kubectl create deployment](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-deployment-em-)
- [kubectl expose](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#expose)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
