# SECTION 9 — Kubernetes and AKS Foundations (6h)

**Deliverable:** an AKS cluster provisioned by Terraform, running the previously containerized monolith as a Deployment + Service, with dev/test namespace isolation and a Helm-installed component.

---

## Lab 9.1 — Architecture, modernization strategy & sizing (45 min, no terminal)

### Objectives
- Describe the control plane / data plane split and say which half Microsoft operates on AKS.
- Map five VM-era concepts onto Kubernetes objects and explain declarative reconciliation.
- Apply a scoring rubric to decide whether a workload belongs on AKS — and name five that do not.
- Size a cluster and design its node pools from workload requests.

### Concept

Kubernetes is a control system, not a place to run containers. You submit *desired state*; controllers close the gap between it and observed state, continuously. In VM operations a change was an imperative event that could silently drift afterwards. Here a change is a persistent intent that keeps being true.

| Traditional                 | Kubernetes                          | What changed                                          |
| --------------------------- | ----------------------------------- | ----------------------------------------------------- |
| Virtual machine             | **Pod**                             | Disposable, seconds to create, no persistent identity |
| Golden image                | **Container image**                 | Immutable, versioned, built in CI                     |
| LB VIP + pool members       | **Service**                         | Membership computed from labels, not edited by hand   |
| `systemctl enable app`      | **Deployment** `replicas: 3`        | The controller restarts it, not you                   |
| `/etc/myapp/app.conf`       | **ConfigMap**                       | Versioned, namespaced, injected at runtime            |
| `.env` with the DB password | **Secret** (→ Key Vault)            | Controlled by RBAC, not file permissions              |
| dev/test/prod VLANs         | **Namespaces** + policy             | Logical isolation inside one cluster                  |
| Change window + runbook     | `kubectl apply` of Git-tracked YAML | Reviewable, auditable, revertable                     |

```mermaid
flowchart TB
    USER["Engineer<br/>kubectl / Terraform / CI"]
    subgraph MSFT["Managed control plane — Microsoft operates"]
        API["kube-apiserver<br/>the only door in"]
        ETCD[("etcd")]
        SCHED["kube-scheduler"]
        KCM["controller-manager<br/>Deployment / ReplicaSet / Node"]
        CCM["cloud-controller-manager"]
    end
    subgraph CUST["Data plane — your subscription, your cost"]
        SYS["System node pool<br/>coredns, metrics-server, CSI"]
        USR["User node pool<br/>kubelet + containerd + kube-proxy"]
    end
    AZURE["Azure: VNet, Load Balancer, Disks, ACR, Entra ID"]
    USER -->|"HTTPS 443, Entra ID"| API
    API <--> ETCD
    SCHED --> API
    KCM --> API
    CCM --> API
    CCM -->|"provision LB / disks"| AZURE
    SYS -->|"kubelet registers"| API
    USR -->|"kubelet registers"| API
    USR --> AZURE
```

**AKS specifics to state now, so later labs make sense**

- **Managed control plane** — no SSH to the API server, no `etcd` backup on you. You choose the tier: 
  - Free (no SLA)
  - Standard (99.95% with zones)
  - Premium (long-term version support)
- **Node pools** — one *system* pool for add-ons, one or more *user* pools for workloads. The split stops a runaway app from starving CoreDNS. Enforced with the `CriticalAddonsOnly` taint (Lab 9.5).
- **Upgrades are two-phase** — control plane first, then node pools (cordon → drain → surge-replace).
- **If the control plane is down for ten minutes**, running Pods keep serving. You lose the ability to *change*, *reschedule* and *scale*. 
### Modernization decision framework

```mermaid
flowchart TD
    START["Workload under review"] --> USED{"Still used?<br/>telemetry, not opinions"}
    USED -->|no| RETIRE["RETIRE"]
    USED -->|yes| CONT{"Can it be containerized?"}
    CONT -->|"no: dongle, kernel module,<br/>unsupported OS"| VM["REHOST on VM / VMSS"]
    CONT -->|yes| STATE{"Local disk or<br/>in-process session?"}
    STATE -->|"heavy, unavoidable"| PAAS["Move state to PaaS first,<br/>then re-evaluate"]
    STATE -->|"none or externalisable"| NEED{"Many services, frequent deploys,<br/>elastic scale, or portability?"}
    NEED -->|no| APPSVC["App Service / Container Apps<br/>Lower TCO. Stop here."]
    NEED -->|yes| TEAM{"Is there a platform team to own<br/>upgrades, security, observability?"}
    TEAM -->|no| CA["Container Apps now.<br/>Revisit AKS when the team exists."]
    TEAM -->|yes| AKS["AKS: replatform as<br/>Deployment + Service"]
    AKS --> DEC{"Does one component's scale or<br/>release cadence conflict<br/>with the rest?"}
    DEC -->|no| DONE["Stop. A monolith on AKS<br/>is a legitimate end state."]
    DEC -->|yes| REF["REFACTOR that component only,<br/>strangler-fig"]
```

| Strategy   | Changes                         | Effort   | Right when                                  |
| ---------- | ------------------------------- | -------- | ------------------------------------------- |
| Retire     | It goes away                    | Very low | Nobody can name a user                      |
| Rehost     | Hosting only                    | Low      | Datacentre exit deadline, no capacity       |
| Replatform | Packaging + externalised config | Medium   | Deployment pain is the bottleneck           |
| Refactor   | Architecture                    | High     | One component's cadence genuinely conflicts |
| Replace    | Buy instead of build            | Medium   | A commodity you happened to build           |

**Sequencing rule that saves projects: containerize before you decompose.** Teams that attempt platform adoption and microservice decomposition at once usually deliver neither.

**When NOT to use Kubernetes** — say all six out loud:

1. A single app deployed monthly by one team. The platform costs more than it returns.
2. Anything that can't be containerized like dongles, kernel modules, MAC-bound licence servers.
3. Stateful engines you can buy as a service. You *can* run PostgreSQL or Kafka with operators; you then own storage performance, backup verification and failover testing.
4. Latency-critical or specialised-hardware workloads (real-time kernels, SR-IOV).
5. Windows apps with heavy GUI/COM+/MSMQ dependencies.
6. **When there is no platform team.**

### Sizing & node pool design

AKS reserves resources on every node before your Pods see any. 

- Memory: 
  - 25% of the first 4 GB, 
  - 20% of 4–8, 
  - 10% of 8–16, 
  - 6% of 16–128, 
  - 2% above. 
- CPU: 
  - 6% of core 1, 
  - 1% of core 2, 
  - 0.5% each for cores 3–4, 
  - 0.25% thereafter. 
 
A `Standard_D2s_v5` (2 vCPU / 8 GB) yields roughly 1.9 vCPU and ~5.4 GB allocatable. This is nearly a third of the memory gone before you deploy anything. Every cost estimate built on raw capacity is wrong.


| Pool role          | Size                         | Notes                                                         |
| ------------------ | ---------------------------- | ------------------------------------------------------------- |
| System             | `D2s_v5`/`D4s_v5`, 2–3 nodes | Never 1 node. Spread across zones. Tainted.                   |
| General apps       | `D4s_v5`/`D8s_v5`            | 4–8 vCPU is the value sweet spot                              |
| Memory-heavy (JVM) | `E4s_v5`/`E8s_v5`            | 8 GB per vCPU                                                 |
| Batch / CI         | `D8s_v5` Spot                | Taint `kubernetes.azure.com/scalesetpriority=spot:NoSchedule` |

IP planning, which you cannot change later: 
- **kubenet** 110 pods/node, non-routable Pod IPs. 
- **Classic Azure CNI** : every Pod takes a VNet IP, so the subnet needs `(nodes + surge) × (max_pods + 1)`; 
  - 100 nodes × 110 pods = 11,211 addresses, a /18. 
- **Azure CNI Overlay** : nodes get VNet IPs, Pods get overlay IPs from a private CIDR, so a /24 supports a large cluster. Node subnets cannot be resized after cluster creation. Default to Overlay.

Rules for every pool: 
- taint special-purpose pools and require an explicit toleration; 
- label by *intent* (`workload=batch`) not implementation (`vm=D8sv5`); - keep the system pool boring and multi-zone; 

---

## Lab 9.2 — Cluster access, `kubectl` workflow & contexts

### Objectives
- Authenticate to the shared AKS cluster with Entra ID + `kubelogin` and explain why `az aks get-credentials` alone is not enough.
- Manage contexts and pin a default namespace.
- Read the cluster's physical topology and your own permissions.

### Concept

`kubectl` reads `~/.kube/config`, which holds three independent lists:
- **clusters** (URL + CA), 
- **users** (how to authenticate), 
- **contexts** (cluster + user + default namespace). 

On an Entra-integrated cluster the user entry holds no token. it holds an *exec plugin* stanza that runs `kubelogin` to fetch a fresh OIDC token. 
- `az aks get-credentials` writes the removed legacy provider; 
- `kubelogin convert-kubeconfig` rewrites it to the supported exec format.

```mermaid
sequenceDiagram
    autonumber
    participant K as kubectl
    participant KL as kubelogin
    participant AAD as Entra ID
    participant API as AKS apiserver
    participant R as RBAC
    K->>KL: exec get-token
    KL->>AAD: request token (azurecli)
    AAD-->>KL: JWT with group claims
    KL-->>K: token (cached ~/.kube/cache/kubelogin)
    K->>API: GET /api/v1/namespaces/student-01/pods
    API->>R: is this group allowed?
    alt RoleBinding exists
        R-->>API: allow
        API-->>K: 200 PodList
    else
        R-->>API: deny
        API-->>K: 403 Forbidden
    end
```

### Setup

```bash
cd ~/k8s-labs/09-foundations
```
```bash
export STUDENT="student-01"          # CHANGE to your assigned ID
export NAMESPACE="$STUDENT"
export NS_DEV="${STUDENT}-dev"
export NS_TEST="${STUDENT}-test"
export RG="rg-k8s-training"
export AKS_NAME="aks-training-shared"
export ACR_NAME="acrtrainingshared"
export LOCATION="centralindia"
```
```bash
cat <<EOF >> ~/.bashrc
export STUDENT="$STUDENT" NAMESPACE="$NAMESPACE" NS_DEV="$NS_DEV" NS_TEST="$NS_TEST"
export RG="$RG" AKS_NAME="$AKS_NAME" ACR_NAME="$ACR_NAME" LOCATION="$LOCATION"
EOF
```

### CLI walkthrough

```bash
az login --use-device-code
az account show -o table
```
```bash
# If multiple subscriptions: 
az account set --subscription "<ID-or-NAME>"
```

```bash
az aks get-credentials -g "$RG" -n "$AKS_NAME" --overwrite-existing
```
```bash
kubelogin convert-kubeconfig -l azurecli
```
```bash
kubectl cluster-info
```

`--overwrite-existing` prevents `aks-training-shared-1`, `-2` accumulating.

Login modes: 
- `-l azurecli` (workstation, our default), 
- `-l devicecode` (headless), 
- `-l workloadidentity` (Pods and federated CI), 
- `-l spn` (legacy CI).

```bash
# Contexts: pin the namespace and give the context a name you can type.
kubectl config get-contexts
```
```bash
kubectl config rename-context "$AKS_NAME" "aks-training"
```
```bash
kubectl config use-context "aks-training"
```
```bash
kubectl config set-context --current --namespace="$NAMESPACE"
```
```bash
kubectl config view --minify -o jsonpath='{..namespace}{"\n"}'
```

```bash
# Physical topology
kubectl get nodes -o wide
```
```bash
kubectl get nodes -L agentpool -L kubernetes.azure.com/mode \
  -L topology.kubernetes.io/zone -L node.kubernetes.io/instance-type
```
```bash
kubectl describe node "$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')" \
  | grep -A 12 -E "Taints:|Allocatable:|Allocated resources"
```
```bash
kubectl top nodes
```

```bash
# Your three self-service reference tools
kubectl api-resources | head -20
```
```bash
kubectl api-resources --namespaced=false | head -10     # why a ClusterRole can't live in your namespace
```
```bash
kubectl explain deployment.spec.strategy
```
```bash
kubectl auth can-i create deployments -n "$NAMESPACE"   # yes
```
```bash
kubectl auth can-i create deployments -n kube-system    # no
```
```bash
kubectl auth can-i --list -n "$NAMESPACE" | head -15
```

### WebUI equivalent

| CLI                              | Azure Portal                                              |
| -------------------------------- | --------------------------------------------------------- |
| `az aks show`                    | **Kubernetes services → cluster → Overview**              |
| `kubectl get nodes -o wide`      | **→ Node pools → pool → Nodes**                           |
| `kubectl get nodes -L agentpool` | **→ Node pools** (Mode column: System / User)             |
| `kubectl top nodes`              | **→ Monitoring → Insights → Nodes**                       |
| `kubectl get pods -A`            | **→ Kubernetes resources → Workloads** (namespace filter) |
| `kubectl auth can-i --list`      | **→ Access control (IAM) → Check access**                 |

1. Portal → **Kubernetes services → `aks-training-shared` → Node pools**. Exactly one pool is Mode `System`.
2. **Kubernetes resources → Workloads**, set **Namespace** to yours. Empty for now.
3. **Monitoring → Insights → Cluster** for node CPU/memory over the last 6 hours.

The Portal's Kubernetes resources blade is a real API client authenticating as *you*. "You do not have access" there is the same `403` the CLI returns, not a Portal bug.

### Verify & troubleshoot

```bash
kubectl config current-context  # aks-training
```
```bash
kubectl config view --minify -o jsonpath='{..namespace}{"\n"}'  # student-01
```
```bash
kubectl get nodes  # all Ready
```
```bash
kubectl auth can-i create deployments -n "$NAMESPACE"  # yes
```
```bash
kubectl get pods -n "$NAMESPACE" # "No resources found" = PASS
```

**Scenario : deliberate 403.** 

Run 
```bash
kubectl get pods -n kube-system
```
then 
```bash
kubectl auth can-i list pods -n kube-system
```

- **`Forbidden` means authentication succeeded and authorization failed;** 
- **`Unauthorized` means the token itself was rejected.** 

---

## Lab 9.3 — Namespaces & dev/test isolation

### Objectives
- Explain precisely what a namespace does and does not isolate.
- Enforce capacity with `ResourceQuota` and set container defaults with `LimitRange`.
- Show that DNS is namespace-aware and that namespaces are *not* a network boundary.

### Concept

A namespace is a **name scope plus a policy attachment point**.

It gives you: 
- name uniqueness, 
- an RBAC attachment point, 
- a quota attachment point, 
- a DNS domain (`<svc>.<ns>.svc.cluster.local`), 
- a cheap blast radius (`delete namespace` removes everything inside).

It does **not** give you:
- network isolation (every Pod can reach every Pod cluster-wide until a NetworkPolicy says otherwise),
- node isolation, 
- kernel isolation.

#### The enterprise answer to "namespaces or clusters for dev/test/prod": 
- dev and test as namespaces in one cluster is normal and cost-effective; 
- **production in its own cluster is the default posture**, 
- because upgrades, CRDs and admission controllers are cluster-wide and hit every namespace at once.

```mermaid
flowchart LR
    POD["New Pod submitted"] --> LR{"LimitRange in namespace?"}
    LR -->|"yes, Pod omits requests"| INJ["Inject default requests + limits"]
    LR -->|no| SKIP["Leave spec as written"]
    INJ --> RQ{"Would ResourceQuota<br/>be exceeded?"}
    SKIP --> RQ
    RQ -->|yes| REJ["REJECTED at admission<br/>'exceeded quota'"]
    RQ -->|no| SCHED["Admitted -> scheduler"]
    SCHED --> RUN["Scheduled onto a node<br/>with enough allocatable capacity"]
```

### Setup

```bash
cd ~/k8s-labs/09-foundations
```
```bash
kubectl config use-context aks-training
```
```bash
kubectl config set-context --current --namespace="$NAMESPACE"
```
```bash
kubectl get ns "$NAMESPACE" "$NS_DEV" "$NS_TEST"
```

### CLI walkthrough

```bash
kubectl label namespace "$NS_DEV"  environment=dev  owner="$STUDENT" cost-center=training --overwrite
```
```bash
kubectl label namespace "$NS_TEST" environment=test owner="$STUDENT" cost-center=training --overwrite
```
```bash
kubectl get namespaces -L environment,owner,cost-center
```
```bash
kubectl get namespaces -l "owner=$STUDENT"
```

```bash
cat <<EOF > 01-governance.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: $NS_DEV
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "8Gi"
    pods: "10"
    services: "5"
    services.loadbalancers: "0"
    persistentvolumeclaims: "4"
    count/deployments.apps: "5"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: $NS_DEV
spec:
  limits:
    - type: Container
      default:
        cpu: "300m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      min:
        cpu: "10m"
        memory: "32Mi"
      max:
        cpu: "1"
        memory: "1Gi"
    - type: Pod
      max:
        cpu: "2"
        memory: "2Gi"
    - type: PersistentVolumeClaim
      min:
        storage: "1Gi"
      max:
        storage: "10Gi"
EOF

kubectl apply -f 01-governance.yaml
kubectl describe resourcequota dev-quota -n "$NS_DEV"
kubectl describe limitrange dev-limits -n "$NS_DEV"
```

`services.loadbalancers: "0"` is a real cost control. every `type: LoadBalancer` Service provisions a public IP.

**Critical rule:** 
- once a quota sets `requests.cpu` or `requests.memory`, 
- **every** container in that namespace must specify requests and limits or be rejected at admission. 
- The `LimitRange` is what keeps existing manifests working.

```bash
cat <<'EOF' > 02-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "192Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: web
  labels:
    app: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: http
EOF
```
```bash
kubectl apply -f 02-web.yaml -n "$NS_DEV"
```
```bash
kubectl apply -f 02-web.yaml -n "$NS_TEST"
```
```bash
kubectl rollout status deployment/web -n "$NS_DEV"  --timeout=180s
```
```bash
kubectl rollout status deployment/web -n "$NS_TEST" --timeout=180s
```
```bash
kubectl describe resourcequota dev-quota -n "$NS_DEV" | grep -E "pods|requests"
```

Two Deployments named `web`, two Services named `web`, zero conflicts. That is the name-scope property.

```bash
# DNS is namespace-aware
kubectl run netshoot --image=nicolaka/netshoot:latest --restart=Never -n "$NS_DEV" --command -- sleep 3600
```
```bash
kubectl wait --for=condition=Ready pod/netshoot -n "$NS_DEV" --timeout=120s
```
```bash
kubectl exec -it netshoot -n "$NS_DEV" -- cat /etc/resolv.conf
```
```bash
kubectl exec -it netshoot -n "$NS_DEV" -- nslookup web
```
```bash
kubectl exec -it netshoot -n "$NS_DEV" -- nslookup "web.${NS_TEST}.svc.cluster.local"
```
```bash
kubectl exec -it netshoot -n "$NS_DEV" -- curl -s -o /dev/null -w "dev  -> %{http_code}\n" http://web
```
```bash
kubectl exec -it netshoot -n "$NS_DEV" -- curl -s -o /dev/null -w "test -> %{http_code}\n" "http://web.${NS_TEST}.svc.cluster.local"
```

The `search` line in `/etc/resolv.conf` is the whole magic behind short names. **Both curls return 200**

### WebUI equivalent

| CLI                                   | Portal                                                        |
| ------------------------------------- | ------------------------------------------------------------- |
| `kubectl get ns`                      | **Kubernetes resources → Namespaces**                         |
| `kubectl label namespace`             | Namespace → **YAML** → edit `metadata.labels` → Review + save |
| `kubectl apply -f 01-governance.yaml` | **Kubernetes resources → + Create → Add with YAML**           |
| `kubectl get deploy -n $NS_DEV`       | **Workloads → Deployments**, Namespace filter                 |


### Verify & troubleshoot

```bash
kubectl get ns -l "owner=$STUDENT"
kubectl get resourcequota,limitrange -n "$NS_DEV"
kubectl get deploy web -n "$NS_DEV"  -o jsonpath='{.status.readyReplicas}{"\n"}'   # 2
kubectl get deploy web -n "$NS_TEST" -o jsonpath='{.status.readyReplicas}{"\n"}'   # 2
```

**Scenario 1 — exceed the quota (asynchronous failure).**

```bash
kubectl scale deployment/web --replicas=12 -n "$NS_DEV"
```
```bash
kubectl get deploy web -n "$NS_DEV"                 # 2/12
```
```bash
kubectl get pods -n "$NS_DEV" -l app=web            # only 2 -- the rest never existed
```
```bash
RS="$(kubectl get rs -n "$NS_DEV" -l app=web -o jsonpath='{.items[0].metadata.name}')"
kubectl describe rs "$RS" -n "$NS_DEV" | sed -n '/Events:/,$p'
```
```bash
kubectl scale deployment/web --replicas=2 -n "$NS_DEV"
```

Expected: `pods "web-xxxxx" is forbidden: exceeded quota: dev-quota, requested: pods=1, used: pods=10, limited: pods=10`.


**Scenario 2 — violate the LimitRange (synchronous failure).**

```bash
kubectl run too-big --image=nginx:1.27-alpine -n "$NS_DEV" --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"hog","image":"nginx:1.27-alpine","resources":{"requests":{"cpu":"100m","memory":"128Mi"},"limits":{"cpu":"3","memory":"4Gi"}}}]}}'
```

Expected: `Error from server (Forbidden): ... maximum cpu usage per Container is 1, but limit is 3`. 

This failed at **admission**, before anything was stored, it appears in your terminal. 

Scenario 1's object *was* stored and failed asynchronously, visible only in `describe` and events. 

Engineers who read only stdout miss half of Kubernetes.

| Symptom                                                       | Cause                                   | Fix                                       |
| ------------------------------------------------------------- | --------------------------------------- | ----------------------------------------- |
| `must specify limits.cpu` on a manifest that worked yesterday | Quota added without a LimitRange        | Add a LimitRange or set requests/limits   |
| Deployment says `0/3`, no Pods exist                          | Quota/admission blocked creation        | `kubectl describe rs` → `FailedCreate`    |
| Namespace stuck `Terminating`                                 | Finalizer, or an unreachable APIService | `kubectl get apiservices \| grep -v True` |
| Short-name DNS fails cross-namespace                          | Expected behaviour                      | Use the FQDN                              |

### Cleanup

```bash
kubectl delete pod netshoot -n "$NS_DEV" --ignore-not-found
```
```bash
kubectl delete -f 02-web.yaml -n "$NS_TEST" --ignore-not-found
```
```bash
kubectl delete -f 02-web.yaml -n "$NS_DEV"  --ignore-not-found
```
```bash
kubectl delete -f 01-governance.yaml --ignore-not-found      # governance goes LAST
```bash
kubectl get all -n "$NS_DEV"; kubectl get all -n "$NS_TEST"
```

Leave the namespaces — later labs use them, and you cannot recreate them on the shared cluster.

---

## Lab 9.4 — Core objects and the monolith on AKS

### Objectives
- Show that a bare Pod is not self-healing, that a ReplicaSet adds *count*, and that a Deployment adds *change management*.
- Perform a rolling update and a rollback; explain the label-selector → endpoints → `kube-proxy` chain.
- Deploy the containerized monolith with externalised config, probes and a `PodDisruptionBudget`.
- Diagnose `ImagePullBackOff`, `CrashLoopBackOff` and an empty-endpoint Service.

### Concept

Three layers, each adding exactly one capability. 
**Pod** is the atom; 
  - one or more containers sharing a network namespace and volumes;
  - mortal, 
  - nothing brings it back. 
**ReplicaSet** adds count; 
  - it creates or deletes Pods until observed equals `spec.replicas`; 
  - it knows nothing about versions. 
**Deployment** adds change management; 
  - it manages one ReplicaSet per Pod-template revision and shifts replicas between them.

**Service** solves a different problem: 
  - Pod IPs change constantly, 
  - so a stable VIP and DNS name have their backends computed continuously from a **label selector**. 
  - A selector that matches nothing fails *silently*
  - the Service exists, DNS resolves, every connection is refused.

**Configuration**: one image, many environments, injected at runtime.
  - `envFrom` values are frozen at container start and need a Pod restart; 
  - mounted files refresh within ~60 s but only matter if the app watches them. 
  - Neither restarts your Pods

```mermaid
flowchart LR
    CLIENT["Client Pod"] -->|"1. resolve"| DNS["CoreDNS<br/>web.student-01.svc.cluster.local"]
    DNS -->|"2. ClusterIP"| CLIENT
    CLIENT -->|"3. connect ClusterIP:80"| KP["kube-proxy<br/>iptables / IPVS"]
    KP -->|"4. DNAT"| P1["Pod app=web READY"]
    SVC["Service web<br/>selector app=web"] -->|"selector match"| EPS["EndpointSlice"]
    EPS --> P1
    EPS --> P2["Pod app=web READY"]
    P3["Pod app=web NOT READY"] -.->|"excluded:<br/>readinessProbe failing"| EPS
    EPS -->|"programmed into"| KP
```

```mermaid
sequenceDiagram
    autonumber
    participant U as Engineer
    participant D as Deployment controller
    participant O as Old ReplicaSet (3 pods)
    participant N as New ReplicaSet (0 pods)
    participant S as Service endpoints
    U->>D: kubectl set image ... app:v2
    D->>N: create RS v2, scale to 1 (maxSurge=1)
    N-->>S: readiness passes -> endpoint ADDED
    D->>O: scale to 2 (maxUnavailable=0 -> only now)
    O-->>S: endpoint REMOVED, pod terminated
    Note over O,N: repeat until N=3, O=0
    D-->>U: successfully rolled out
    Note over O: Old RS KEPT at 0 replicas -> instant rollback
```

### Setup

```bash
cd ~/k8s-labs/09-foundations
```
```bash
kubectl config set-context --current --namespace="$NAMESPACE"
```
```bash
export ACR_LOGIN_SERVER="$(az acr show --name "$ACR_NAME" --query loginServer -o tsv 2>/dev/null || echo "${ACR_NAME}.azurecr.io")"
```
```bash
export APP_IMAGE="${ACR_LOGIN_SERVER}/${STUDENT}/monolith:1.0.0"
```
```bash
az acr repository show-tags --name "$ACR_NAME" --repository "${STUDENT}/monolith" -o table 2>/dev/null \
  || { export APP_IMAGE="mcr.microsoft.com/azuredocs/aks-helloworld:v1"; echo "fallback image: $APP_IMAGE"; }
```
```bash
echo "APP_IMAGE=$APP_IMAGE"
```

### Part A — Pod, ReplicaSet, Deployment

```bash
# A bare Pod, killed, is gone. That is the whole lesson.
kubectl run standalone --image=mcr.microsoft.com/azuredocs/aks-helloworld:v1 \
  -n "$NAMESPACE" --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"web","image":"mcr.microsoft.com/azuredocs/aks-helloworld:v1","resources":{"requests":{"cpu":"50m","memory":"64Mi"},"limits":{"cpu":"200m","memory":"192Mi"}}}]}}'
```
```bash
kubectl wait --for=condition=Ready pod/standalone -n "$NAMESPACE" --timeout=180s
```
```bash
kubectl delete pod standalone -n "$NAMESPACE"
sleep 5
```
```bash
kubectl get pods -n "$NAMESPACE"        # nothing. A bare Pod is like a VM you forgot to put in a scale set.
```

```bash
cat <<'EOF' > 03-replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
  labels:
    app: web
    managed-by: replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      managed-by: replicaset
  template:
    metadata:
      labels:
        app: web
        managed-by: replicaset
    spec:
      containers:
        - name: web
          image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "192Mi"
EOF
```
```bash
kubectl apply -f 03-replicaset.yaml -n "$NAMESPACE"
```
```bash
kubectl rollout status deployment/web-rs -n "$NAMESPACE" 2>/dev/null || kubectl get rs,pods -n "$NAMESPACE"
```

# Self-healing: delete one, get three back.
VICTIM="$(kubectl get pods -n "$NAMESPACE" -l managed-by=replicaset -o jsonpath='{.items[0].metadata.name}')"
kubectl delete pod "$VICTIM" -n "$NAMESPACE"
sleep 15
kubectl get pods -n "$NAMESPACE" -l managed-by=replicaset

# The selector -- not the owner reference -- is what it watches.
ADOPTEE="$(kubectl get pods -n "$NAMESPACE" -l managed-by=replicaset -o jsonpath='{.items[0].metadata.name}')"
kubectl label pod "$ADOPTEE" -n "$NAMESPACE" managed-by=orphan --overwrite
kubectl get pods -n "$NAMESPACE" --show-labels          # FOUR pods now
kubectl delete pod "$ADOPTEE" -n "$NAMESPACE"
```

Relabelling to orphan a Pod is exactly how you quarantine a misbehaving Pod for a live post-mortem: the controller replaces it immediately while your broken copy keeps running for inspection.

```bash
# The limitation that motivates Deployments:
kubectl set image rs/web-rs web=mcr.microsoft.com/azuredocs/aks-helloworld:v2 -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image'
# Every running Pod is still v1. A ReplicaSet applies its template only when CREATING a Pod.
kubectl delete -f 03-replicaset.yaml -n "$NAMESPACE"
```

### Part B — The monolith as a Deployment + Service + ConfigMap + Secret (40 min)

```bash
cat <<'EOF' > 04-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: monolith-config
  labels:
    app: monolith
data:
  APP_ENV: "dev"
  LOG_LEVEL: "debug"
  TITLE: "Containerized Monolith on AKS"
  DB_HOST: "monolith-db.database.svc.cluster.local"
  DB_PORT: "5432"
  app.properties: |
    server.port=80
    server.shutdown=graceful
    management.endpoints.web.exposure.include=health,info,metrics
    logging.level.root=INFO
---
apiVersion: v1
kind: Secret
metadata:
  name: monolith-secrets
  labels:
    app: monolith
type: Opaque
stringData:
  DB_USER: "monolith_app"
  DB_PASSWORD: "Tr41n1ng-L4b-N0t-Real!"
  API_KEY: "lab-only-8f3c1d0a4b7e9265"
EOF

kubectl apply -f 04-config.yaml -n "$NAMESPACE"
echo "04-config.yaml" >> .gitignore

# base64 is ENCODING, not encryption:
kubectl get secret monolith-secrets -n "$NAMESPACE" -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
```

Say it to the room: anyone with `get secret` in this namespace just read the database password in one command. The control is RBAC + encryption-at-rest + not committing it to Git — not the word "Secret". **Lab 12.2 replaces this with Key Vault and Workload Identity.**

```bash
cat <<EOF > 05-monolith.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monolith
  labels:
    app: monolith
    tier: application
  annotations:
    kubernetes.io/change-cause: "Initial deployment of monolith 1.0.0"
spec:
  replicas: 2
  revisionHistoryLimit: 5
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: monolith
      tier: application
  template:
    metadata:
      labels:
        app: monolith
        tier: application
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: monolith
          image: $APP_IMAGE
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
          envFrom:
            - configMapRef:
                name: monolith-config
            - secretRef:
                name: monolith-secrets
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: app-config
              mountPath: /etc/monolith
              readOnly: true
          startupProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 5
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 20
            timeoutSeconds: 3
            failureThreshold: 3
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
      volumes:
        - name: app-config
          configMap:
            name: monolith-config
            items:
              - key: app.properties
                path: app.properties
            defaultMode: 0444
      terminationGracePeriodSeconds: 45
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: monolith
---
apiVersion: v1
kind: Service
metadata:
  name: monolith
  labels:
    app: monolith
spec:
  type: ClusterIP
  selector:
    app: monolith
    tier: application
  ports:
    - name: http
      port: 80
      targetPort: http
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: monolith-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: monolith
      tier: application
EOF

kubectl apply -f 05-monolith.yaml -n "$NAMESPACE"
kubectl rollout status deployment/monolith -n "$NAMESPACE" --timeout=300s
kubectl get deploy,rs,pods,svc,endpoints,pdb -n "$NAMESPACE" -o wide
```

Narrate four design decisions while it rolls:

- **`startupProbe` 30×5s** gives a slow JVM/.NET monolith 150 s to boot while liveness stays aggressive afterwards. Without it you'd need `livenessProbe.initialDelaySeconds: 150`, which also delays detection of a real hang by 150 s.
- **`preStop: sleep 5` + `terminationGracePeriodSeconds: 45`** — endpoint removal and `SIGTERM` happen concurrently, so a short pause closes the race where a Pod receives requests after it starts shutting down.
- **`maxSurge: 1` / `maxUnavailable: 0`** — capacity never dips below 100%.
- **`minAvailable: 1` PDB on 2 replicas** gives `ALLOWED DISRUPTIONS: 1`. **The trap:** `minAvailable: 1` on a *single*-replica Deployment gives `0` and blocks node drains — and therefore AKS upgrades — forever, with no obvious error.

```bash
# Verify config actually reached the container
POD="$(kubectl get pods -n "$NAMESPACE" -l app=monolith -o jsonpath='{.items[0].metadata.name}')"
kubectl exec -it "$POD" -n "$NAMESPACE" -- sh -c 'env | grep -E "^(APP_ENV|LOG_LEVEL|DB_HOST|DB_USER|POD_NAME|NODE_NAME)=" | sort'
kubectl exec -it "$POD" -n "$NAMESPACE" -- ls -la /etc/monolith/
kubectl exec -it "$POD" -n "$NAMESPACE" -- cat /etc/monolith/app.properties
```

The `..data` symlink pointing at a timestamped directory is how the kubelet updates config atomically — an app never reads a half-written file.

```bash
# Env vars are frozen at container start.
kubectl patch configmap monolith-config -n "$NAMESPACE" --type=merge -p '{"data":{"LOG_LEVEL":"warn"}}'
sleep 10
kubectl exec -it "$POD" -n "$NAMESPACE" -- sh -c 'echo "still: LOG_LEVEL=$LOG_LEVEL"'
kubectl rollout restart deployment/monolith -n "$NAMESPACE"
kubectl rollout status deployment/monolith -n "$NAMESPACE" --timeout=300s
NEW_POD="$(kubectl get pods -n "$NAMESPACE" -l app=monolith -o jsonpath='{.items[0].metadata.name}')"
kubectl exec -it "$NEW_POD" -n "$NAMESPACE" -- sh -c 'echo "now: LOG_LEVEL=$LOG_LEVEL"'
```

`rollout restart` adds a `restartedAt` annotation, changing the template hash and triggering a normal rolling update. **Never `kubectl delete pod` to "restart" production** — that skips the surge and drops capacity.

```bash
# Same image, second environment, different config -- the point of the whole exercise.
kubectl apply -f 04-config.yaml   -n "$NS_DEV"
kubectl apply -f 05-monolith.yaml -n "$NS_DEV"
kubectl patch configmap monolith-config -n "$NS_DEV" --type=merge -p '{"data":{"APP_ENV":"dev","LOG_LEVEL":"trace"}}'
kubectl rollout restart deployment/monolith -n "$NS_DEV"
kubectl rollout status deployment/monolith -n "$NS_DEV" --timeout=300s

kubectl get deploy monolith -n "$NAMESPACE" -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get deploy monolith -n "$NS_DEV"    -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get cm monolith-config -n "$NAMESPACE" -o jsonpath='{.data.APP_ENV}{"\n"}'
kubectl get cm monolith-config -n "$NS_DEV"    -o jsonpath='{.data.APP_ENV}{"\n"}'
```

### Part C — Rolling update, rollback, and the three failure modes (20 min)

```bash
kubectl set image deployment/monolith monolith=mcr.microsoft.com/azuredocs/aks-helloworld:v2 -n "$NAMESPACE"
kubectl annotate deployment/monolith -n "$NAMESPACE" kubernetes.io/change-cause="Upgrade to v2" --overwrite
kubectl rollout status deployment/monolith -n "$NAMESPACE" --timeout=300s
kubectl get rs -n "$NAMESPACE" -l app=monolith        # old RS kept at 0 -> instant rollback
kubectl rollout history deployment/monolith -n "$NAMESPACE"
kubectl rollout undo deployment/monolith -n "$NAMESPACE"
kubectl rollout status deployment/monolith -n "$NAMESPACE" --timeout=300s
```

**Scenario 1 — `ImagePullBackOff`.**

```bash
kubectl set image deployment/monolith monolith=mcr.microsoft.com/azuredocs/aks-helloworld:v99-nope -n "$NAMESPACE"
sleep 25
kubectl get pods -n "$NAMESPACE" -l app=monolith
BAD="$(kubectl get pods -n "$NAMESPACE" -l app=monolith --field-selector=status.phase=Pending -o jsonpath='{.items[0].metadata.name}')"
kubectl describe pod "$BAD" -n "$NAMESPACE" | sed -n '/Events:/,$p'
kubectl rollout undo deployment/monolith -n "$NAMESPACE"
kubectl rollout status deployment/monolith -n "$NAMESPACE" --timeout=300s
```

**Availability never dropped.** `maxUnavailable: 0` meant a healthy old Pod was never removed until a new one was Ready — and none ever was. A bad image became a *stalled deployment* instead of an *outage*. Real-world causes in order: wrong tag; ACR not attached (`az aks update -g $RG -n $AKS_NAME --attach-acr $ACR_NAME`); missing `imagePullSecrets`; wrong CPU architecture; registry firewall.

**Scenario 2 — `CrashLoopBackOff` and `--previous`.**

```bash
kubectl create deployment crasher --image=busybox:1.36 -n "$NAMESPACE" -- \
  /bin/sh -c "echo starting; sleep 5; echo 'FATAL: cannot reach db:5432' >&2; exit 1"
sleep 45
CP="$(kubectl get pods -n "$NAMESPACE" -l app=crasher -o jsonpath='{.items[0].metadata.name}')"
kubectl logs "$CP" -n "$NAMESPACE"                 # the current attempt -- usually not the failure
kubectl logs "$CP" -n "$NAMESPACE" --previous      # the attempt that actually died
kubectl get pod "$CP" -n "$NAMESPACE" -o jsonpath='restarts={.status.containerStatuses[0].restartCount} exit={.status.containerStatuses[0].lastState.terminated.exitCode} reason={.status.containerStatuses[0].lastState.terminated.reason}{"\n"}'
kubectl delete deployment crasher -n "$NAMESPACE"
```

`--previous` separates people who can debug Kubernetes from people who cannot. Exit **1** = the app quit; **137** = SIGKILL, almost always OOM (confirm `reason=OOMKilled`); **143** = SIGTERM, normal shutdown.

**Scenario 3 — the silent killer: a Service selecting nothing.**

```bash
kubectl create service clusterip web-broken --tcp=80:8080 -n "$NAMESPACE"
kubectl patch svc web-broken -n "$NAMESPACE" --type=merge -p '{"spec":{"selector":{"app":"monolith","tier":"backend"}}}'
kubectl get endpoints web-broken -n "$NAMESPACE"      # <none>
kubectl get svc web-broken -n "$NAMESPACE" -o jsonpath='{.spec.selector}{"\n"}'
kubectl get pods -n "$NAMESPACE" -l app=monolith --show-labels
kubectl delete svc web-broken -n "$NAMESPACE"
```

**The Service debugging ladder — muscle memory:**

```bash
kubectl get svc <svc> -n $NAMESPACE -o jsonpath='{.spec.selector}{"\n"}'   # 1. what does it select?
kubectl get pods -n $NAMESPACE --show-labels                               # 2. what do Pods have?
kubectl get endpoints <svc> -n $NAMESPACE                                  # 3. did they match?
kubectl get pods -n $NAMESPACE -o wide                                     # 4. are they Ready?
kubectl exec -it <client> -n $NAMESPACE -- nslookup <svc>                  # 5. does DNS resolve?
kubectl exec -it <client> -n $NAMESPACE -- curl -v http://<svc>:<port>     # 6. does the port match?
```

Ninety percent of "the Service doesn't work" tickets die at step 3.

### WebUI equivalent

| CLI                                 | Portal                                                       |
| ----------------------------------- | ------------------------------------------------------------ |
| `kubectl apply -f 05-monolith.yaml` | **Kubernetes resources → + Create → Add with YAML**          |
| `kubectl get deploy/rs/pods`        | **Workloads → Deployments / Replica sets / Pods**            |
| `kubectl describe pod`              | Pod → **Overview** + **Events**                              |
| `kubectl logs -f`                   | Pod → **Live logs**                                          |
| `kubectl scale` / `set image`       | Deployment → **YAML** tab → edit → Review + save             |
| `kubectl get svc / endpoints`       | **Services and ingresses → Services** → selector + endpoints |
| `kubectl rollout undo`              | **No UI equivalent — CLI/GitOps only**                       |
| `kubectl exec`                      | Not in the Portal; use `kubectl` or Lens                     |

Point at the gap in that table. The Portal is excellent for *observing* and acceptable for a one-off edit. Lifecycle operations — rollout, rollback, pause, restart — belong to `kubectl` and, in the target state, to Git.

### Verify

```bash
kubectl get deploy monolith -n "$NAMESPACE" -o jsonpath='{.status.readyReplicas}/{.spec.replicas}{"\n"}'   # 2/2
kubectl get endpoints monolith -n "$NAMESPACE"                                    # 2 IPs
kubectl get pdb monolith-pdb -n "$NAMESPACE"                                      # ALLOWED DISRUPTIONS 1
kubectl rollout history deployment/monolith -n "$NAMESPACE"                       # >= 3 revisions
kubectl port-forward svc/monolith 8082:80 -n "$NAMESPACE" & sleep 3
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8082; kill %1
```

| Symptom                           | Cause                             | Command that proves it                          |
| --------------------------------- | --------------------------------- | ----------------------------------------------- |
| `Pending` forever                 | No allocatable capacity, or quota | `describe pod` → `FailedScheduling`             |
| `Running` but `0/1 READY`         | Readiness failing                 | `describe pod` → `Readiness probe failed`       |
| `CreateContainerConfigError`      | Missing ConfigMap/Secret **key**  | `describe pod`; `kubectl logs` returns nothing  |
| Restarts every ~60 s, app healthy | Liveness probe wrong port/path    | `get events --field-selector reason=Unhealthy`  |
| Exit 137 / `OOMKilled`            | Memory limit too low              | `-o jsonpath='{..lastState.terminated.reason}'` |
| Drain / upgrade hangs             | PDB `disruptionsAllowed: 0`       | `kubectl get pdb -A`                            |

### Cleanup

Keep the monolith running in `$NAMESPACE` — Sections 10, 11 and 12 all build on it. Remove only the dev copy and the debris:

```bash
kubectl delete -f 05-monolith.yaml -n "$NS_DEV" --ignore-not-found
kubectl delete -f 04-config.yaml   -n "$NS_DEV" --ignore-not-found
kubectl delete pod --field-selector=status.phase=Succeeded -n "$NAMESPACE" --ignore-not-found
kubectl get all -n "$NAMESPACE"
```

---

## Lab 9.5 — Provision AKS with Terraform (75 min)

### Objectives
- Build a complete AKS cluster in code: RG, VNet, subnet, Log Analytics, cluster, user node pool.
- Enforce the system/user pool split with the `CriticalAddonsOnly` taint and prove it with a scheduling test.
- Read a `terraform plan` as a change record, and destroy cleanly.

### Concept

Everything so far assumed a cluster existed. Three design decisions dominate an AKS module: **node pool topology** (tainted system pool + user pools), **network plugin** (Azure CNI Overlay — nodes on VNet IPs, Pods on an overlay CIDR, near-CNI performance with kubenet-like IP economy), and **identity** (system-assigned managed identity, `AcrPull` for the kubelet identity, OIDC + Workload Identity so apps need no stored secret).

**Pacing note for the instructor:** start `terraform apply` at the beginning of the hour, then teach the sizing material from Lab 9.1 while the 8–10 minute provision runs.

**State warning before anyone runs `apply`:** state contains secrets and is the source of truth for destruction. Local state is fine for a lab and unacceptable anywhere else.

```mermaid
flowchart LR
    A["terraform init<br/>+ fmt + validate"] --> B["terraform plan -out=tfplan<br/>what WILL change"]
    B --> C{"Reviewed<br/>by a human?"}
    C -->|no| B
    C -->|yes| D["terraform apply tfplan<br/>~8 min"]
    D --> E["az aks get-credentials<br/>kubelogin convert-kubeconfig"]
    E --> F["kubectl apply the monolith"]
    F --> G["terraform destroy<br/>end of lab"]
    subgraph POOLS["What gets built"]
        SYS["system pool: 2x D2s_v5<br/>taint CriticalAddonsOnly"]
        USR["user pool: 1-3x D4s_v5<br/>autoscaled, untainted"]
    end
    D --> POOLS
```

### Setup

```bash
mkdir -p ~/k8s-labs/09-foundations/terraform && cd $_
export ARM_SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
export TENANT_ID="$(az account show --query tenantId -o tsv)"

az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.OperationalInsights
az aks get-versions --location "$LOCATION" -o table     # pin a version that actually exists
```

### CLI walkthrough

```bash
cat <<'EOF' > providers.tf
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
  # Lab uses local state. Production MUST use a remote backend:
  # backend "azurerm" {
  #   resource_group_name  = "rg-tfstate"
  #   storage_account_name = "sttfstateshared01"
  #   container_name       = "tfstate"
  #   key                  = "aks/student-01.tfstate"
  #   use_azuread_auth     = true
  # }
}

provider "azurerm" {
  subscription_id = var.subscription_id
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}

provider "random" {}
EOF
```

```bash
cat <<'EOF' > variables.tf
variable "subscription_id" {
  type = string
}

variable "tenant_id" {
  type = string
}

variable "student_id" {
  type = string
  validation {
    condition     = can(regex("^[a-z0-9-]{3,20}$", var.student_id))
    error_message = "student_id must be 3-20 lowercase alphanumeric or hyphen characters."
  }
}

variable "location" {
  type    = string
  default = "centralindia"
}

variable "kubernetes_version" {
  type    = string
  default = "1.31"
}

variable "sku_tier" {
  type    = string
  default = "Free"
}

variable "vnet_address_space" {
  type    = list(string)
  default = ["10.10.0.0/16"]
}

variable "aks_subnet_prefix" {
  type    = list(string)
  default = ["10.10.1.0/24"]
}

variable "pod_cidr" {
  type    = string
  default = "192.168.0.0/16"
}

variable "service_cidr" {
  type    = string
  default = "172.16.0.0/16"
}

variable "dns_service_ip" {
  type    = string
  default = "172.16.0.10"
}

variable "system_node_vm_size" {
  type    = string
  default = "Standard_D2s_v5"
}

variable "system_node_count" {
  type    = number
  default = 2
}

variable "user_node_vm_size" {
  type    = string
  default = "Standard_D4s_v5"
}

variable "user_node_min_count" {
  type    = number
  default = 1
}

variable "user_node_max_count" {
  type    = number
  default = 3
}

variable "max_pods_per_node" {
  type    = number
  default = 110
}

variable "availability_zones" {
  type    = list(string)
  default = ["1", "2", "3"]
}

variable "admin_group_object_ids" {
  type    = list(string)
  default = []
}

variable "acr_id" {
  description = "Resource ID of an existing ACR to attach. Empty string skips the role assignment."
  type        = string
  default     = ""
}

variable "tags" {
  type = map(string)
  default = {
    environment = "training"
    managed-by  = "terraform"
  }
}
EOF
```

```bash
cat <<'EOF' > main.tf
locals {
  name_prefix = var.student_id
  common_tags = merge(var.tags, { owner = var.student_id })
}

resource "random_string" "suffix" {
  length  = 5
  special = false
  upper   = false
}

resource "azurerm_resource_group" "aks" {
  name     = "rg-aks-${local.name_prefix}"
  location = var.location
  tags     = local.common_tags
}

# With Azure CNI Overlay the subnet only holds node IPs plus surge nodes,
# so a /24 is generous even for a large cluster.
resource "azurerm_virtual_network" "aks" {
  name                = "vnet-aks-${local.name_prefix}"
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
  address_space       = var.vnet_address_space
  tags                = local.common_tags
}

resource "azurerm_subnet" "aks_nodes" {
  name                 = "snet-aks-nodes"
  resource_group_name  = azurerm_resource_group.aks.name
  virtual_network_name = azurerm_virtual_network.aks.name
  address_prefixes     = var.aks_subnet_prefix
}

resource "azurerm_log_analytics_workspace" "aks" {
  name                = "law-aks-${local.name_prefix}-${random_string.suffix.result}"
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
  tags                = local.common_tags
}

resource "azurerm_kubernetes_cluster" "this" {
  name                = "aks-${local.name_prefix}"
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
  dns_prefix          = "aks-${local.name_prefix}"
  kubernetes_version  = var.kubernetes_version
  sku_tier            = var.sku_tier
  node_resource_group = "rg-aks-${local.name_prefix}-nodes"

  automatic_upgrade_channel = "patch"
  node_os_upgrade_channel   = "NodeImage"

  oidc_issuer_enabled       = true
  workload_identity_enabled = true

  role_based_access_control_enabled = true

  default_node_pool {
    name                        = "system"
    vm_size                     = var.system_node_vm_size
    node_count                  = var.system_node_count
    vnet_subnet_id              = azurerm_subnet.aks_nodes.id
    zones                       = var.availability_zones
    max_pods                    = var.max_pods_per_node
    os_disk_size_gb             = 128
    os_sku                      = "Ubuntu"
    temporary_name_for_rotation = "systemtmp"

    # Taints this pool CriticalAddonsOnly=true:NoSchedule so only
    # tolerating add-ons (CoreDNS, metrics-server, CSI) land here.
    only_critical_addons_enabled = true

    node_labels = {
      "nodepool-type" = "system"
    }

    upgrade_settings {
      max_surge = "33%"
    }
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"
    network_policy      = "calico"
    load_balancer_sku   = "standard"
    pod_cidr            = var.pod_cidr
    service_cidr        = var.service_cidr
    dns_service_ip      = var.dns_service_ip
  }

  azure_active_directory_role_based_access_control {
    tenant_id              = var.tenant_id
    admin_group_object_ids = var.admin_group_object_ids
    azure_rbac_enabled     = true
  }

  oms_agent {
    log_analytics_workspace_id      = azurerm_log_analytics_workspace.aks.id
    msi_auth_for_monitoring_enabled = true
  }

  auto_scaler_profile {
    balance_similar_node_groups = true
    scale_down_delay_after_add  = "10m"
    scale_down_unneeded         = "10m"
  }

  tags = local.common_tags

  lifecycle {
    # The autoscaler owns the live node count; don't fight it on every plan.
    ignore_changes = [default_node_pool[0].node_count]
  }
}

resource "azurerm_kubernetes_cluster_node_pool" "user" {
  name                  = "user"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.this.id
  vm_size               = var.user_node_vm_size
  mode                  = "User"
  zones                 = var.availability_zones
  vnet_subnet_id        = azurerm_subnet.aks_nodes.id
  max_pods              = var.max_pods_per_node
  os_disk_size_gb       = 128
  os_type               = "Linux"
  os_sku                = "Ubuntu"

  auto_scaling_enabled = true
  min_count            = var.user_node_min_count
  max_count            = var.user_node_max_count

  node_labels = {
    "nodepool-type" = "user"
    "workload"      = "applications"
  }

  upgrade_settings {
    max_surge = "33%"
  }

  tags = local.common_tags

  lifecycle {
    ignore_changes = [node_count]
  }
}

resource "azurerm_role_assignment" "acr_pull" {
  count                            = var.acr_id == "" ? 0 : 1
  scope                            = var.acr_id
  role_definition_name             = "AcrPull"
  principal_id                     = azurerm_kubernetes_cluster.this.kubelet_identity[0].object_id
  skip_service_principal_aad_check = true
}
EOF
```

```bash
cat <<'EOF' > outputs.tf
output "resource_group_name" {
  value = azurerm_resource_group.aks.name
}

output "cluster_name" {
  value = azurerm_kubernetes_cluster.this.name
}

output "cluster_fqdn" {
  value = azurerm_kubernetes_cluster.this.fqdn
}

output "node_resource_group" {
  description = "Azure-managed RG holding the VMSS, disks and load balancer. Never edit it by hand."
  value       = azurerm_kubernetes_cluster.this.node_resource_group
}

output "oidc_issuer_url" {
  value = azurerm_kubernetes_cluster.this.oidc_issuer_url
}

output "kubelet_identity_object_id" {
  value = azurerm_kubernetes_cluster.this.kubelet_identity[0].object_id
}

output "get_credentials_command" {
  value = format(
    "az aks get-credentials --resource-group %s --name %s --overwrite-existing && kubelogin convert-kubeconfig -l azurecli",
    azurerm_resource_group.aks.name,
    azurerm_kubernetes_cluster.this.name
  )
}

output "kube_config_raw" {
  value     = azurerm_kubernetes_cluster.this.kube_config_raw
  sensitive = true
}
EOF
```

```bash
cat <<EOF > terraform.tfvars
subscription_id    = "$ARM_SUBSCRIPTION_ID"
tenant_id          = "$TENANT_ID"
student_id         = "$STUDENT"
location           = "$LOCATION"
kubernetes_version = "1.31"
sku_tier           = "Free"
acr_id             = ""
EOF

cat <<'EOF' > .gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
tfplan
terraform.tfvars
EOF
```

`terraform.tfvars` holds subscription and tenant IDs; `*.tfstate` holds the cluster's kubeconfig and certificates in cleartext. Both stay out of Git — in the real pipeline they become pipeline variables and a remote backend.

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan -out=tfplan
terraform show -no-color tfplan | grep -E "^  # |Plan:"
```

Expect ~7 resources to create. **Read the plan out loud with the class** — in a regulated environment, this output attached to a change ticket is worth more than any screenshot.

> If `validate` rejects an argument, you are on a different provider major version. AzureRM 4.x renamed several (`enable_auto_scaling` → `auto_scaling_enabled`, `automatic_channel_upgrade` → `automatic_upgrade_channel`). `terraform validate` names the offending argument — read the error rather than searching for a copy-paste fix. This is the job.

```bash
time terraform apply tfplan          # 6-10 min. Teach the 9.1 sizing material while it runs.
terraform output
export TF_RG="$(terraform output -raw resource_group_name)"
export TF_AKS="$(terraform output -raw cluster_name)"
az aks get-credentials -g "$TF_RG" -n "$TF_AKS" --overwrite-existing
kubelogin convert-kubeconfig -l azurecli
kubectl get nodes -L nodepool-type,topology.kubernetes.io/zone
```

**Prove the node pool design works:**

```bash
kubectl get nodes -l nodepool-type=system \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints[*].key}{"\n"}{end}'

kubectl create namespace app-demo
kubectl create deployment sched-test --image=nginx:1.27-alpine --replicas=4 -n app-demo
kubectl rollout status deployment/sched-test -n app-demo --timeout=180s
kubectl get pods -n app-demo -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName'
kubectl get nodes -L nodepool-type
```

Every Pod is on `user`, zero on `system`. That output is the deliverable of this step.

**Deploy the monolith onto your own cluster — the Section 9 deliverable:**

```bash
kubectl create namespace production
kubectl apply -f ~/k8s-labs/09-foundations/04-config.yaml   -n production
kubectl apply -f ~/k8s-labs/09-foundations/05-monolith.yaml -n production
kubectl rollout status deployment/monolith -n production --timeout=300s
kubectl get all -n production
kubectl port-forward svc/monolith 8083:80 -n production & sleep 3
curl -s -o /dev/null -w "monolith on my own AKS cluster -> HTTP %{http_code}\n" http://localhost:8083; kill %1
```

**Change through Terraform, not the Portal:**

```bash
sed -i 's/^acr_id             = ""/user_node_max_count = 5/' terraform.tfvars 2>/dev/null
echo 'user_node_max_count = 5' >> terraform.tfvars
terraform plan -out=tfplan2 | grep -E "max_count|Plan:"
terraform apply tfplan2
az aks nodepool show -g "$TF_RG" --cluster-name "$TF_AKS" -n user --query "{min:minCount,max:maxCount}" -o table
```

An in-place update. Contrast with changing `default_node_pool.name` or `network_profile.network_plugin`, which print `# forces replacement` — the one line you must never skim past.

### WebUI equivalent

Walk the **Create → Containers → AKS** wizard *after* Terraform finishes, so students map fields to HCL:

| Terraform                                           | Portal field                                   |
| --------------------------------------------------- | ---------------------------------------------- |
| `default_node_pool.vm_size` / `node_count`          | **Basics → Node size / Node count**            |
| `azurerm_kubernetes_cluster_node_pool.user`         | **Node pools → + Add node pool** (Mode = User) |
| `only_critical_addons_enabled`                      | **Node pools → pool → Taints**                 |
| `auto_scaling_enabled` / `min` / `max`              | **Node pools → Scale method: Autoscale**       |
| `network_plugin_mode = "overlay"`                   | **Networking → Azure CNI Overlay**             |
| `network_policy = "calico"`                         | **Networking → Network policy: Calico**        |
| `oidc_issuer_enabled` / `workload_identity_enabled` | **Security → OIDC issuer / Workload Identity** |
| `oms_agent`                                         | **Integrations → Container Insights**          |
| `acr_id` role assignment                            | **Integrations → Container registry**          |

Then two things to show in the Portal:

1. **Resource groups → `rg-aks-student-01-nodes`** — the *node resource group*: VMSS, disks, load balancer, NSG. Azure owns it. **Never modify anything in it by hand**; AKS reconciles it and your change vanishes, or breaks the cluster.
2. **The drift demo.** Change the `user` pool's max count to 9 in the Portal → Apply. Then run `terraform plan`. Terraform reports drift and proposes to revert. The Portal edit was faster and is now invisible to every reviewer, every audit and every colleague. **That demo is the strongest argument for GitOps you will make all day.**

```bash
terraform apply -auto-approve      # restores declared state
```

### Verify & troubleshoot

```bash
az aks show -g "$TF_RG" -n "$TF_AKS" \
  --query "{name:name,version:currentKubernetesVersion,plugin:networkProfile.networkPlugin,mode:networkProfile.networkPluginMode,policy:networkProfile.networkPolicy,state:provisioningState}" -o table
az aks nodepool list -g "$TF_RG" --cluster-name "$TF_AKS" -o table
kubectl get pods -n kube-system -o wide | head
```

**Scenario — a Pod that targets the tainted system pool.**

```bash
kubectl run wrong-pool --image=nginx:1.27-alpine -n app-demo \
  --overrides='{"spec":{"nodeSelector":{"nodepool-type":"system"},"containers":[{"name":"app","image":"nginx:1.27-alpine"}]}}'
sleep 15
kubectl describe pod wrong-pool -n app-demo | sed -n '/Events:/,$p'
kubectl delete pod wrong-pool -n app-demo
```

Expected: `node(s) had untolerated taint {CriticalAddonsOnly: true}`. **The fix is not to remove the taint** — it is to schedule on the user pool, or, for a genuine add-on, add the toleration deliberately.

| Symptom                                            | Cause                              | Fix                                                |
| -------------------------------------------------- | ---------------------------------- | -------------------------------------------------- |
| `subscription ID could not be determined`          | AzureRM 4.x needs it explicitly    | Set `subscription_id` or `ARM_SUBSCRIPTION_ID`     |
| `Unsupported argument`                             | Provider major-version rename      | Read the error; check docs for the pinned version  |
| `QuotaExceeded` / insufficient vCPU                | Subscription core quota            | Reduce node counts or request an increase          |
| `ServiceCidrOverlapExistingSubnetsCidr`            | `service_cidr` overlaps the VNet   | VNet, pod CIDR and service CIDR must be disjoint   |
| Apply hangs then fails                             | No capacity for that SKU in a zone | Try another SKU or drop `availability_zones`       |
| `creating Role Assignment ... AuthorizationFailed` | No `roleAssignments/write`         | Attach ACR later with `az aks update --attach-acr` |
| `destroy` fails on the RG                          | Objects created outside Terraform  | Delete LoadBalancer Services and PVCs first        |

### Cleanup

**Order matters more here than anywhere else in the course.** Kubernetes objects that created Azure resources go first.

```bash
kubectl delete svc -A --field-selector spec.type=LoadBalancer --ignore-not-found   # each owns a public IP
kubectl delete namespace production app-demo --ignore-not-found                     # removes PVCs / disks
kubectl get pv                                                                      # must be empty

cd ~/k8s-labs/09-foundations/terraform
terraform plan -destroy -out=tfdestroy
terraform apply tfdestroy

az group list --query "[?starts_with(name, 'rg-aks-${STUDENT}')].name" -o tsv       # must be empty
kubectl config delete-context "aks-${STUDENT}" 2>/dev/null || true
kubectl config use-context aks-training
```

If destroy fails partway, re-run `terraform apply tfdestroy` — it is idempotent. If it fails repeatedly on one resource, delete it in the Portal, `terraform state rm <address>`, destroy again.

> **Instructor cost sweep, next morning:**
> `for g in $(az group list --query "[?starts_with(name,'rg-aks-student-')].name" -o tsv); do az group delete -n "$g" --yes --no-wait; done`
> Twenty student clusters left running for a week is a four-figure invoice.

---

## Lab 9.6 — Helm (40 min)

### Objectives
- Explain chart, release and values, and the precedence order between them.
- Install, upgrade, roll back and uninstall a release in your namespace.
- Render with `helm template` before installing, and use the checksum-annotation idiom so a config change rolls the Pods automatically.

### Concept

Helm is a template engine plus a release ledger. A **chart** is templated manifests + `values.yaml`. A **release** is one installation of a chart into one namespace under a name. **Values precedence**, lowest to highest: chart defaults → `-f values.yaml` (later files win) → `--set`. Each revision's rendered manifests are stored in a Secret named `sh.helm.release.v1.<release>.v<n>` — that ledger is what makes rollback possible.

What Helm is **not**: a deployment controller. `helm upgrade` applies and waits; if someone edits the objects afterwards, Helm has no idea until the next upgrade. That gap is the argument for ArgoCD or Flux.

```mermaid
sequenceDiagram
    autonumber
    participant U as Engineer
    participant H as helm
    participant A as kube-apiserver
    participant S as Release Secret
    U->>H: helm install web ./chart -n student-01 -f values.yaml
    H->>H: merge values, render templates
    H->>A: create objects
    H->>S: store revision 1
    U->>H: helm upgrade --set replicaCount=2
    H->>S: read revision 1
    H->>A: 3-way merge patch, changed objects only
    H->>S: store revision 2
    U->>H: helm rollback web 1
    H->>S: read revision 1 manifests
    H->>A: apply revision 1 state
    H->>S: store revision 3 (copy of 1) -- ledger is append-only
```

### Setup

```bash
cd ~/k8s-labs/09-foundations
kubectl config use-context aks-training
kubectl config set-context --current --namespace="$NAMESPACE"
helm version
```

### CLI walkthrough

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm search repo ingress-nginx --versions | head -5
helm show values ingress-nginx/ingress-nginx | head -30
```

> **Chart sourcing, for the enterprise audience.** Public chart repositories change ownership, licensing and image locations — Bitnami's public catalog was restructured during 2025 and broke a large number of pinned references overnight. Mirror the charts and images you depend on into your own ACR, and pin chart versions and image digests. Never let a production rollout depend on an anonymous pull from a repository you do not control.

**Render before you install — always:**

```bash
helm template demo ingress-nginx/ingress-nginx -n "$NAMESPACE" > preview.yaml
grep -E "^kind:" preview.yaml | sort | uniq -c
```

`ClusterRole`, `ValidatingWebhookConfiguration`, `IngressClass` — all cluster-scoped, all things you cannot create as a namespace-scoped student. **`helm template` told you the install would fail before you tried it.** (The instructor has already installed this chart cluster-wide for Section 10.)

**Scaffold your own chart — how every real chart in your organisation starts:**

```bash
helm create monolith-chart
rm -f monolith-chart/templates/hpa.yaml monolith-chart/templates/ingress.yaml

cat <<'EOF' > monolith-chart/values.yaml
replicaCount: 2

image:
  repository: mcr.microsoft.com/azuredocs/aks-helloworld
  pullPolicy: IfNotPresent
  tag: "v1"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: "monolith-helm"

serviceAccount:
  create: true
  automount: false
  annotations: {}
  name: ""

config:
  appEnv: "dev"
  logLevel: "info"
  title: "Monolith deployed by Helm"

podAnnotations: {}
podLabels: {}

podSecurityContext:
  seccompProfile:
    type: RuntimeDefault

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

livenessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 15
  periodSeconds: 20

readinessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 5
  periodSeconds: 10

autoscaling:
  enabled: false

volumes: []
volumeMounts: []
nodeSelector: {}
tolerations: []
affinity: {}
EOF

cat <<'EOF' > monolith-chart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "monolith-chart.fullname" . }}-config
  labels:
    {{- include "monolith-chart.labels" . | nindent 4 }}
data:
  APP_ENV: {{ .Values.config.appEnv | quote }}
  LOG_LEVEL: {{ .Values.config.logLevel | quote }}
  TITLE: {{ .Values.config.title | quote }}
EOF
```

Wire the ConfigMap into the Deployment, with the checksum annotation — this is *the* Helm idiom:

```bash
python3 - <<'PYEOF'
import pathlib
p = pathlib.Path("monolith-chart/templates/deployment.yaml")
s = p.read_text()

anno = ('      annotations:\n'
        '        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}\n')

if "checksum/config" not in s:
    if "      annotations:\n        {{- toYaml . | nindent 8 }}" in s:
        s = s.replace("      annotations:\n",
                      anno.replace("      annotations:\n", "      annotations:\n"), 1)
        s = s.replace('      annotations:\n        {{- toYaml . | nindent 8 }}',
                      anno + '        {{- toYaml . | nindent 8 }}', 1)
    else:
        s = s.replace("    spec:\n", anno + "    spec:\n", 1)

s = s.replace(
    "          resources:\n            {{- toYaml .Values.resources | nindent 12 }}",
    '          envFrom:\n            - configMapRef:\n                name: {{ include "monolith-chart.fullname" . }}-config\n'
    "          resources:\n            {{- toYaml .Values.resources | nindent 12 }}",
    1,
)
p.write_text(s)
PYEOF

grep -n -B1 -A2 "checksum/config" monolith-chart/templates/deployment.yaml
grep -n -A2 "envFrom" monolith-chart/templates/deployment.yaml
```

```bash
helm lint ./monolith-chart
helm template monolith-helm ./monolith-chart -n "$NAMESPACE" | grep -E "^kind:|checksum/config"

helm install monolith-helm ./monolith-chart -n "$NAMESPACE" --wait --timeout 5m
helm list -n "$NAMESPACE"
helm status monolith-helm -n "$NAMESPACE" | head -8
kubectl get secret -n "$NAMESPACE" -l "owner=helm"     # the release ledger, visible
```

**Upgrade with a values file, then with `--set`:**

```bash
cat <<'EOF' > values-dev.yaml
replicaCount: 3
config:
  appEnv: "dev"
  logLevel: "debug"
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 192Mi
EOF

helm upgrade monolith-helm ./monolith-chart -n "$NAMESPACE" -f values-dev.yaml --wait --timeout 5m
helm upgrade monolith-helm ./monolith-chart -n "$NAMESPACE" -f values-dev.yaml --set replicaCount=1 --wait
kubectl get deploy -n "$NAMESPACE" -l "app.kubernetes.io/instance=monolith-helm" \
  -o custom-columns='NAME:.metadata.name,REPLICAS:.spec.replicas'
helm history monolith-helm -n "$NAMESPACE"
```

`--set replicaCount=1` beat the file's `3`. Command line always wins.

**The checksum idiom, proved:**

```bash
kubectl get pods -n "$NAMESPACE" -l "app.kubernetes.io/instance=monolith-helm" \
  -o custom-columns='NAME:.metadata.name,AGE:.metadata.creationTimestamp'
helm upgrade monolith-helm ./monolith-chart -n "$NAMESPACE" -f values-dev.yaml --set config.logLevel=warn --wait
kubectl get pods -n "$NAMESPACE" -l "app.kubernetes.io/instance=monolith-helm" \
  -o custom-columns='NAME:.metadata.name,AGE:.metadata.creationTimestamp'
```

New Pod names. A ConfigMap-only change produced a rolling restart automatically — no `rollout restart` needed. That is the whole reason charts beat a folder of static YAML.

**Rollback and test:**

```bash
helm history monolith-helm -n "$NAMESPACE"
helm rollback monolith-helm 2 -n "$NAMESPACE" --wait --timeout 5m
helm history monolith-helm -n "$NAMESPACE"      # rollback is a NEW revision -- append-only ledger
helm test monolith-helm -n "$NAMESPACE" --logs  # the generated test-connection Pod; most teams delete it, a mistake
```

**Diff plugin — install on day one of any real project:**

```bash
helm plugin install https://github.com/databus23/helm-diff 2>/dev/null || echo "already installed"
helm diff upgrade monolith-helm ./monolith-chart -n "$NAMESPACE" -f values-dev.yaml --set replicaCount=4
```

Red/green field-level output before anything happens. In a change-controlled enterprise that output *is* your change record.

**Package to ACR — the enterprise pattern (charts as OCI artifacts beside your images):**

```bash
helm package ./monolith-chart --destination ./dist
# az acr login --name "$ACR_NAME"
# helm push ./dist/monolith-chart-0.1.0.tgz "oci://${ACR_LOGIN_SERVER}/helm"
# helm install monolith-helm "oci://${ACR_LOGIN_SERVER}/helm/monolith-chart" --version 0.1.0 -n "$NAMESPACE"
```

### WebUI equivalent

Helm has no first-class Portal UI. **Workloads → Deployments →** open a release object → **YAML** → find `app.kubernetes.io/managed-by: Helm`. That label is the only clue in the UI. **Configuration → Secrets** shows `sh.helm.release.v1.monolith-helm.v1..v5` — one per revision.

**Drift demo:** scale `monolith-helm` to 5 in the Portal, then `helm upgrade monolith-helm ./monolith-chart -n "$NAMESPACE" -f values-dev.yaml --wait`. It snaps back. Editing a Helm-managed object through the Portal is a drift event that the next upgrade silently reverts.

### Verify & troubleshoot

```bash
helm list -n "$NAMESPACE"                                    # STATUS deployed
helm history monolith-helm -n "$NAMESPACE"                   # >= 4 revisions
kubectl get all -n "$NAMESPACE" -l "app.kubernetes.io/managed-by=Helm"
```

**Scenario — a failed upgrade, and `--atomic`.**

```bash
helm upgrade monolith-helm ./monolith-chart -n "$NAMESPACE" \
  -f values-dev.yaml --set image.tag=not-a-real-tag --wait --timeout 90s
helm list -n "$NAMESPACE"                                    # STATUS: failed
kubectl describe pod -n "$NAMESPACE" -l "app.kubernetes.io/instance=monolith-helm" | sed -n '/Events:/,$p' | tail -10
helm rollback monolith-helm -n "$NAMESPACE" --wait --timeout 5m

# Better: let Helm roll itself back.
helm upgrade monolith-helm ./monolith-chart -n "$NAMESPACE" \
  -f values-dev.yaml --set image.tag=still-not-real --atomic --timeout 90s || echo "auto-rolled-back"
helm list -n "$NAMESPACE"                                    # STATUS: deployed
```

`--wait --timeout` is what turns a silent half-broken rollout into a failed command your pipeline can detect. Always use it in CI; prefer `--atomic`.

| Symptom                                        | Cause                                      | Fix                                        |
| ---------------------------------------------- | ------------------------------------------ | ------------------------------------------ |
| `cannot re-use a name that is still in use`    | Release exists                             | `helm upgrade`, or uninstall first         |
| Stuck in `pending-upgrade`                     | Interrupted upgrade (Ctrl-C, CI timeout)   | `helm rollback <rel> <last-good>`          |
| `field is immutable`                           | Changed a Deployment `selector`            | Uninstall and reinstall                    |
| Objects exist, `helm list` empty               | Wrong namespace, or created with `kubectl` | `helm list -A`; check `managed-by`         |
| `INSTALLATION FAILED: ... is forbidden`        | Chart creates cluster-scoped objects       | `helm template` first; instructor installs |
| Rollback reverts config but Pods don't restart | No checksum annotation                     | Add it as above                            |
| Values seem ignored                            | Precedence, or a typo in a nested key      | `helm get values <rel> -n <ns> --all`      |

### Cleanup

```bash
helm uninstall monolith-helm -n "$NAMESPACE"
helm list -n "$NAMESPACE" --all
kubectl get all -n "$NAMESPACE" -l "app.kubernetes.io/managed-by=Helm"
rm -rf ./dist preview.yaml
kubectl get all -n "$NAMESPACE"                  # the hand-written monolith from 9.4 remains -- keep it
```

`helm uninstall` deletes the history Secrets too, so rollback is no longer possible. Use `--keep-history` when you want the release marked `uninstalled` but still rollback-able.

