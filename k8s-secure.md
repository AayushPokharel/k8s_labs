# SECURE AKS OPERATIONS

Production-oriented, lab-first workshop for developers and DevOps engineers working with Azure Kubernetes Service (AKS).

## Workshop overview

The core workflow is:

```text
Human identity → AKS authentication → Kubernetes authorization

Workload identity → Azure resource authorization

External secret store → CSI volume → application

Pod security → workload hardening

Network policy → explicit application connectivity
```


## Lab sequence

1. **Lab 1 — Replace a Kubernetes Secret with Azure Key Vault**
2. **Lab 2 — Replace static Azure credentials with Workload Identity**
3. **Lab 3 — Enforce the Restricted Pod Security Standard**
4. **Lab 4 — Build a default-deny application network model**

## Learning objectives

By the end of the workshop, participants should be able to:

1. Apply least-privilege access patterns inside and outside AKS.
2. Distinguish cluster-wide administrative access from namespace-scoped access.
3. Replace static Azure credentials with Microsoft Entra Workload ID.
4. Explain the security limitations of native Kubernetes Secrets.
5. Mount Azure Key Vault secrets through the Secrets Store CSI Driver.
6. Apply Pod Security Standards to harden workloads.
7. Implement default-deny network policies with explicit application and DNS exceptions.
8.  Troubleshoot common identity, secret-store, pod-security, and network-policy failures.

---

# Lab 1 — Replace Kubernetes Secrets with Azure Key Vault

### Student inputs

the student must provide with:

```text
LAB1_KEYVAULT_NAME
LAB1_KEYVAULT_SECRET
LAB1_CLIENT_ID
LAB1_TENANT_ID
```

The `LAB1_CLIENT_ID` is the client ID of the pre-provisioned user-assigned managed identity used by the Secrets Store CSI Driver.

## Student objective

You will:

1. Create an isolated namespace.
2. Create a simple Kubernetes Secret.
3. Mount and read it from a Pod.
4. Replace it with an Azure Key Vault secret.
5. Mount the Key Vault secret using the Secrets Store CSI Driver.
6. Remove the Pod's dependency on the Kubernetes Secret.

## Expected result

The final Pod reads the secret from `/mnt/secrets-store/app-secret`, while the application has no dependency on a Kubernetes Secret object for the value.

---

## Step 1 — Set lab variables

> **STUDENT**

Set the student identifier assigned by the instructor.

```bash
# Set the unique student identifier supplied for this workshop.
export STUDENT="student01"
```

Build the namespace name used by every manifest in this lab.

```bash
# Build the namespace name used by all student resources.
export LAB_NAMESPACE="secure-${STUDENT}"
```

Set the pre-provisioned Key Vault inputs supplied by the instructor.

```bash
# Point the lab at this student's dedicated Key Vault and federated identity.
export LAB1_KEYVAULT_NAME="REPLACE_WITH_INSTRUCTOR_VALUE"
```

Set the Key Vault secret object name supplied by the instructor.

```bash
# Set the exact Key Vault secret object name prepared for this student.
export LAB1_KEYVAULT_SECRET="REPLACE_WITH_INSTRUCTOR_VALUE"
```

Set the user-assigned managed identity client ID supplied by the instructor.

```bash
# Set the client ID used by the SecretProviderClass to authenticate to Key Vault.
export LAB1_CLIENT_ID="REPLACE_WITH_INSTRUCTOR_VALUE"
```

Set the Azure tenant that owns the Key Vault.

```bash
# Read the tenant ID from the current Azure CLI session unless the instructor supplied a different tenant.
export LAB1_TENANT_ID="$(az account show --query tenantId --output tsv)"
```

## Step 2 — Create the isolated namespace

> **STUDENT**

Create only your own namespace.

```bash
# Create the student's isolated namespace.
kubectl create namespace "$LAB_NAMESPACE"
```

Verify that it exists.

```bash
# Confirm the namespace was created successfully.
kubectl get namespace "$LAB_NAMESPACE"
```

---

## Step 3 — Create a simple Kubernetes Secret

This first phase demonstrates the static Kubernetes Secret pattern before we replace it.

Apply the starter Secret manifest.

```bash
# Create the temporary Kubernetes Secret used by the first version of the application.
envsubst < manifests/keyvault/static-secret.yaml | kubectl apply -f -
```

Verify the Secret exists.

```bash
# Confirm the temporary Secret exists in the student namespace.
kubectl get secret demo-app-secret --namespace "$LAB_NAMESPACE"
```

### Manifest — `manifests/keyvault/static-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-app-secret
  namespace: ${LAB_NAMESPACE}
type: Opaque
stringData:
  app-secret: workshop-static-secret
```

> **Production note:** Base64 in a Kubernetes Secret manifest is not encryption. Kubernetes recommends encryption at rest, least-privilege access, and considering an external secret store.
> [Kubernetes — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) and [Kubernetes — Good practices for Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/).

---

## Step 4 — Deploy the application using the Kubernetes Secret

Apply the starter Pod.

```bash
# Start a tiny BusyBox workload that mounts the Kubernetes Secret as a file.
envsubst < manifests/keyvault/static-secret-pod.yaml | kubectl apply -f -
```

### Manifest — `manifests/keyvault/static-secret-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
  namespace: ${LAB_NAMESPACE}
  labels:
    app: secret-demo
spec:
  restartPolicy: Always
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          while true; do
            echo "secret file contents:";
            cat /etc/app-secret/app-secret;
            sleep 30;
          done
      volumeMounts:
        - name: app-secret
          mountPath: /etc/app-secret
          readOnly: true
  volumes:
    - name: app-secret
      secret:
        secretName: demo-app-secret
```

Wait for the Pod to start.

```bash
# Wait until the demonstration Pod is running.
kubectl wait --for=condition=Ready pod/secret-demo --namespace "$LAB_NAMESPACE" --timeout=120s
```

Read the mounted Secret from inside the Pod.

```bash
# Demonstrate that the application can read the Kubernetes Secret through the mounted file.
kubectl exec --namespace "$LAB_NAMESPACE" pod/secret-demo -- cat /etc/app-secret/app-secret
```

Expected result:

```text
workshop-static-secret
```

### Teaching pause

Ask the class:

> Where is the secret now?

> Answer: in the Kubernetes API as a Secret object, and in the Pod filesystem after the kubelet mounts it.

---

## Step 5 — Inspect the Secret without revealing its value in normal output

List the Secret metadata.

```bash
# Inspect the Secret object without printing its data field.
kubectl get secret demo-app-secret --namespace "$LAB_NAMESPACE" --output yaml --show-managed-fields=false
```

---

## Step 6 — Verify Key Vault identity preparation

Show the Key Vault identity client ID currently configured for the lab.

```bash
# Display the client ID that the SecretProviderClass will use.
echo "$LAB1_CLIENT_ID"
```

Verify the Key Vault itself is reachable through Azure control-plane metadata.

```bash
# Confirm the Key Vault resource exists and show its resource ID.
az keyvault show --name "$LAB1_KEYVAULT_NAME" --query '{name:name,id:id}' --output yaml
```

---

## Step 7 — Create the SecretProviderClass

The following manifest uses the pre-provisioned federated identity and reads a single secret from the student's Key Vault.

### Manifest — `manifests/keyvault/secretproviderclass.yaml`

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: student-keyvault
  namespace: ${LAB_NAMESPACE}
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "${LAB1_CLIENT_ID}"
    keyvaultName: "${LAB1_KEYVAULT_NAME}"
    tenantId: "${LAB1_TENANT_ID}"
    cloudName: ""
    objects: |
      array:
        - |
          objectName: ${LAB1_KEYVAULT_SECRET}
          objectType: secret
          objectVersion: ""
```

Render and apply the `SecretProviderClass` to the student's namespace.

```bash
# Substitute the student-specific values and create the namespace-scoped SecretProviderClass.
envsubst < manifests/keyvault/secretproviderclass.yaml | kubectl apply -f -
```

Verify the resource exists.

```bash
# Confirm the SecretProviderClass exists in the student namespace.
kubectl get secretproviderclass student-keyvault --namespace "$LAB_NAMESPACE"
```

---

## Step 8 — Deploy the application from Key Vault

Create the ServiceAccount referenced by the Key Vault Pod before scheduling the Pod.

```bash
# Create the namespace-scoped ServiceAccount trusted by the instructor-prepared federated identity.
envsubst < manifests/keyvault/serviceaccount.yaml | kubectl apply -f -
```

### Manifest — `manifests/keyvault/keyvault-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: keyvault-demo
  namespace: ${LAB_NAMESPACE}
  labels:
    app: keyvault-demo
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: keyvault-reader
  restartPolicy: Always
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          while true; do
            echo "Key Vault mounted value:";
            cat /mnt/secrets-store/app-secret;
            sleep 30;
          done
      volumeMounts:
        - name: secrets-store
          mountPath: /mnt/secrets-store
          readOnly: true
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: student-keyvault
```

### Manifest — `manifests/keyvault/serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: keyvault-reader
  namespace: ${LAB_NAMESPACE}
  annotations:
    azure.workload.identity/client-id: "${LAB1_CLIENT_ID}"
```

Apply the final Pod manifest.

```bash
# Start the test Pod with the Key Vault secret mounted by the Secrets Store CSI Driver.
envsubst < manifests/keyvault/keyvault-pod.yaml | kubectl apply -f -
```

Wait for it to be ready.

```bash
# Wait for the Key Vault test Pod to become ready.
kubectl wait --for=condition=Ready pod/keyvault-demo --namespace "$LAB_NAMESPACE" --timeout=120s
```

Read the mounted Key Vault secret.

```bash
# Prove the secret was mounted from Azure Key Vault into the Pod filesystem.
kubectl exec --namespace "$LAB_NAMESPACE" pod/keyvault-demo -- cat /mnt/secrets-store/app-secret
```

Expected result is the instructor-provisioned Key Vault secret value.

---

## Step 9 — Remove the Kubernetes Secret dependency

Delete the temporary Pod that used the Kubernetes Secret.

```bash
# Remove the first version of the application that depended on the Kubernetes Secret.
kubectl delete pod secret-demo --namespace "$LAB_NAMESPACE" --ignore-not-found
```

Delete the temporary Kubernetes Secret.

```bash
# Remove the static Secret so the final workload no longer depends on it.
kubectl delete secret demo-app-secret --namespace "$LAB_NAMESPACE" --ignore-not-found
```

Verify only the Key Vault path remains.

```bash
# Confirm the temporary Kubernetes Secret is gone.
kubectl get secret demo-app-secret --namespace "$LAB_NAMESPACE"
```

Expected result: `NotFound`.

---

## Validation checklist

> **STUDENT**

Verify the final workload, identity reference, and external secret mount.

```bash
# List the final Pod and ServiceAccount in the student namespace.
kubectl get pod,serviceaccount --namespace "$LAB_NAMESPACE"
```

```bash
# Show the final Pod's mounted volumes and scheduling events for troubleshooting.
kubectl describe pod keyvault-demo --namespace "$LAB_NAMESPACE"
```

```bash
# Confirm the mounted secret file exists without exposing the full value.
kubectl exec --namespace "$LAB_NAMESPACE" pod/keyvault-demo -- test -s /mnt/secrets-store/app-secret && echo "Key Vault secret mounted"
```
## Teaching point

The final workload is not made magically secure just because the secret came from Key Vault. Security depends on **which identity is trusted, how narrowly it is scoped, and who can create or mutate the workload that uses it**.

---

# Lab 2 — Azure Workload Identity

## Student objective

Use a Kubernetes ServiceAccount to obtain an Azure token and read Storage Account metadata without:

- a client secret;
- a storage account key;
- a Kubernetes Secret containing Azure credentials.

## Expected result

A test Pod successfully authenticates to Azure Storage with the projected federation token injected by the Workload Identity webhook.

---

## Step 1 — Set lab variables

> **STUDENT**

Set the student identifier.

```bash
# Set the unique student identifier used for the namespace and federated credential.
export STUDENT="student01"
```

Set the namespace.

```bash
# Build the isolated namespace name for this student.
export LAB_NAMESPACE="secure-${STUDENT}"
```

Set the Storage Account name supplied by the instructor.

```bash
# Identify the Azure Storage Account used for the read-only lab.
export LAB2_STORAGE_ACCOUNT="REPLACE_WITH_INSTRUCTOR_VALUE"
```

Set the pre-created blob container name supplied by the instructor.

```bash
# Identify the pre-created read-only blob container used by the lab.
export LAB2_STORAGE_CONTAINER="REPLACE_WITH_INSTRUCTOR_VALUE"
```

Set the managed identity client ID supplied by the instructor.

```bash
# Set the client ID of this student's user-assigned managed identity.
export LAB2_CLIENT_ID="REPLACE_WITH_INSTRUCTOR_VALUE"
```

Get the current Azure tenant ID.

```bash
# Read the Microsoft Entra tenant ID from the current Cloud Shell session.
export LAB2_TENANT_ID="$(az account show --query tenantId --output tsv)"
```

---

## Step 2 — Verify the namespace exists

> **STUDENT**

Check the namespace from Lab 1.

```bash
# Confirm the same isolated namespace is available for the Workload Identity lab.
kubectl get namespace "$LAB_NAMESPACE"
```

---

## Step 3 — Create the ServiceAccount

The ServiceAccount annotation identifies the Azure managed identity client ID.

### Manifest — `manifests/workload-identity/serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: azure-storage-reader
  namespace: ${LAB_NAMESPACE}
  annotations:
    azure.workload.identity/client-id: "${LAB2_CLIENT_ID}"
```

Render and apply the ServiceAccount.

```bash
# Create the namespace-scoped ServiceAccount used by the federated identity.
envsubst < manifests/workload-identity/serviceaccount.yaml | kubectl apply -f -
```

Inspect the annotation.

```bash
# Confirm the ServiceAccount points at the expected managed identity client ID.
kubectl get serviceaccount azure-storage-reader --namespace "$LAB_NAMESPACE" --output yaml
```

---

## Step 4 — Deploy the test Pod

The Pod label `azure.workload.identity/use: "true"` is required for the Workload Identity mutating webhook to inject the projected token and environment variables. [Microsoft Learn — Microsoft Entra Workload ID](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview).

### Manifest — `manifests/workload-identity/azure-cli-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: workload-identity-demo
  namespace: ${LAB_NAMESPACE}
  labels:
    app: workload-identity-demo
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: azure-storage-reader
  restartPolicy: Never
  containers:
    - name: azure-cli
      image: mcr.microsoft.com/azure-cli:latest
      command:
        - sh
        - -c
        - |
          echo "Workload Identity environment:";
          env | grep '^AZURE_' | sort;
          echo "Federated token file:";
          ls -l "$AZURE_FEDERATED_TOKEN_FILE";
          echo "Logging into Microsoft Entra ID with the projected federation token...";
          az login --service-principal \
            --username "$AZURE_CLIENT_ID" \
            --tenant "$AZURE_TENANT_ID" \
            --federated-token "$(cat "$AZURE_FEDERATED_TOKEN_FILE")" \
            --output none;
          echo "Reading Blob container metadata with the federated identity...";
          az storage container list \
            --account-name "${LAB2_STORAGE_ACCOUNT}" \
            --auth-mode login \
            --query '[].{name:name}' \
            --output table
```

Apply the Pod manifest.

```bash
# Start the test Pod that authenticates using the projected federation token.
envsubst < manifests/workload-identity/azure-cli-pod.yaml | kubectl apply -f -
```

Wait for completion.

```bash
# Wait for the one-shot Azure CLI Pod to finish its authentication test.
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/workload-identity-demo --namespace "$LAB_NAMESPACE" --timeout=180s
```

Read the test output.

```bash
# Show the Workload Identity authentication and Storage Account lookup results.
kubectl logs pod/workload-identity-demo --namespace "$LAB_NAMESPACE"
```

Expected output includes an Azure account login followed by a Storage Account row. Exact output varies by Azure CLI version.

---

## Step 5 — Prove there is no static Azure credential in the Pod spec

Inspect the Pod YAML for the absence of a client secret or storage key.

```bash
# Search the effective Pod definition for common static credential fields.
kubectl get pod workload-identity-demo --namespace "$LAB_NAMESPACE" --output yaml | grep -E 'client-secret|clientSecret|storage-key|account-key' || true
```

Inspect the injected environment and volume configuration.

```bash
# Show the Workload Identity-related environment variables and token volume without printing the token value.
kubectl describe pod workload-identity-demo --namespace "$LAB_NAMESPACE" | grep -A20 -E 'Environment:|Mounts:'
```

### Teaching pause

Question:

> What changed from a client-secret design?

Answer: the application now relies on an identity federation trust relationship instead of storing a long-lived credential.

---

## Step 6 — Inspect the federated ServiceAccount token path

Show the ServiceAccount configuration.

```bash
# Confirm the client ID annotation and namespace of the federated ServiceAccount.
kubectl describe serviceaccount azure-storage-reader --namespace "$LAB_NAMESPACE"
```

Inspect the Pod events if the token injection did not happen.

```bash
# Review admission and scheduling events when Workload Identity mutation is missing.
kubectl describe pod workload-identity-demo --namespace "$LAB_NAMESPACE"
```

## Teaching point

The security boundary is the **federation relationship**: the exact ServiceAccount identity, namespace, issuer, audience, and Azure role assignment jointly determine what the Pod can do.

---

# Lab 3 — Pod Security Standards

## Student objective

You will:

1. Enforce the `restricted` Pod Security Standard on your namespace.
2. Attempt to deploy an intentionally insecure Pod.
3. Observe admission rejection.
4. Deploy a compliant Pod.

Kubernetes defines Restricted as the strongly hardened PSS profile, including requirements around non-root execution, privilege escalation, capabilities, and seccomp. [Kubernetes — Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/).

## Expected result

The insecure Pod is rejected by admission, while the hardened Pod is admitted and reaches `Running`.

---

## Step 1 — Set variables

> **STUDENT**

Set the namespace used by this workshop.

```bash
# Set the existing isolated student namespace.
export STUDENT="student01"
```

```bash
# Build the namespace used by the Pod Security exercise.
export LAB_NAMESPACE="secure-${STUDENT}"
```

## Step 2 — Apply namespace PSS labels

Apply Restricted enforcement plus audit and warning labels so students can see the policy behavior clearly.

```bash
# Enforce Restricted and also enable audit/warn visibility for the same profile.
kubectl label namespace "$LAB_NAMESPACE" pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/audit=restricted pod-security.kubernetes.io/warn=restricted --overwrite
```

Verify the labels.

```bash
# Confirm the namespace now carries the expected Pod Security labels.
kubectl get namespace "$LAB_NAMESPACE" --show-labels
```

---

## Step 3 — Deploy an intentionally insecure Pod

This Pod deliberately requests multiple privileges blocked by the Restricted profile.

### Manifest — `manifests/pod-security/insecure-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
  namespace: ${LAB_NAMESPACE}
spec:
  restartPolicy: Never
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "id; sleep 3600"]
      securityContext:
        privileged: true
        runAsUser: 0
        allowPrivilegeEscalation: true
```

Attempt to create the Pod.

```bash
# Submit the deliberately insecure Pod and observe the admission response.
envsubst < manifests/pod-security/insecure-pod.yaml | kubectl apply -f -
```

Expected result: the API server rejects the Pod with Pod Security admission errors or warnings indicating forbidden Restricted controls.

---

## Step 4 — Inspect the namespace policy

Check the labels again.

```bash
# Verify that Restricted remains the enforced profile for the namespace.
kubectl get namespace "$LAB_NAMESPACE" -o jsonpath='{.metadata.labels}' && echo
```

Inspect the rejected Pod if your cluster created an object record before admission completed.

```bash
# Check whether the rejected Pod object exists; NotFound is also an acceptable outcome.
kubectl get pod insecure-pod --namespace "$LAB_NAMESPACE" --ignore-not-found
```

---

## Step 5 — Deploy the compliant Pod

The corrected Pod explicitly satisfies the important Restricted controls.

### Manifest — `manifests/pod-security/restricted-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  namespace: ${LAB_NAMESPACE}
  labels:
    app: restricted-demo
spec:
  restartPolicy: Always
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
    fsGroup: 1000
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo 'restricted pod is running'; sleep 3600"]
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

Apply the compliant Pod.

```bash
# Deploy the hardened Pod into the Restricted namespace.
envsubst < manifests/pod-security/restricted-pod.yaml | kubectl apply -f -
```

Wait for it to become ready.

```bash
# Confirm the Restricted-compliant Pod reaches Ready.
kubectl wait --for=condition=Ready pod/restricted-pod --namespace "$LAB_NAMESPACE" --timeout=120s
```

Inspect the running Pod.

```bash
# Display the effective security context and recent events for the hardened Pod.
kubectl describe pod restricted-pod --namespace "$LAB_NAMESPACE"
```

Read its output.

```bash
# Verify the test container started under the restricted security profile.
kubectl logs pod/restricted-pod --namespace "$LAB_NAMESPACE"
```

Expected output:

```text
restricted pod is running
```

## Validation checklist

> **STUDENT**

Confirm the insecure Pod is not running.

```bash
# Verify only the compliant Pod is running in the namespace.
kubectl get pods --namespace "$LAB_NAMESPACE"
```

Confirm the Pod is non-root from inside the container.

```bash
# Show the container's effective UID; it should not be 0.
kubectl exec --namespace "$LAB_NAMESPACE" pod/restricted-pod -- id
```

Expected output includes:

```text
uid=1000
```

## Teaching point

Restricted PSS is a **policy boundary**, not an application rewrite. The application should still be designed to run as a non-root process with explicit privileges.

---

# Lab 4 — Default-Deny Network Policy

## Student objective

Build this namespace-level trust model:

```text
frontend  --ALLOW--> backend  --ALLOW--> database
    |                    |                  |
    +------ DNS ---------+------------------+

Everything else = DENY
```

## Expected result

- Before policies: frontend can reach backend, backend can reach database.
- After default deny: application traffic is blocked.
- After explicit allow rules: only frontend→backend and backend→database are restored, plus required DNS egress.

---

## Step 1 — Set variables

> **STUDENT**

Set the namespace.

```bash
# Reuse the unique student namespace for the network-policy exercise.
export STUDENT="student01"
```

```bash
# Build the namespace name used by every network-policy resource in this lab.
export LAB_NAMESPACE="secure-${STUDENT}"
```

## Step 2 — Verify a network-policy engine is present

Use Azure metadata to inspect the configured AKS network policy engine.

```bash
# Read the AKS network plugin, network policy engine, and Cilium data plane settings without changing them.
az aks show --name "$AKS_CLUSTER_NAME" --resource-group "$AKS_RESOURCE_GROUP" --query '{networkPlugin:networkProfile.networkPlugin,networkPolicy:networkProfile.networkPolicy,networkDataplane:networkProfile.networkDataplane}' --output yaml
```
---

## Step 3 — Deploy frontend, backend, and database test workloads

The demo uses small non-root BusyBox containers and ClusterIP Services. The database is represented by an HTTP listener purely so the network path can be tested without adding a real database dependency. This keeps the demo compatible with the Restricted Pod Security namespace from Lab 3.

### Manifest — `manifests/network-policy/app-stack.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: ${LAB_NAMESPACE}
spec:
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: ${LAB_NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: ${LAB_NAMESPACE}
spec:
  selector:
    app: database
  ports:
    - port: 8080
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: ${LAB_NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
        - name: database
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: ${LAB_NAMESPACE}
  labels:
    app: frontend
spec:
  containers:
    - name: frontend
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Apply the test stack.

```bash
# Deploy the frontend, backend, and database test workloads plus their Services.
envsubst < manifests/network-policy/app-stack.yaml | kubectl apply -f -
```

Wait for the Deployments and Pod.

```bash
# Wait for the backend Deployment to become available.
kubectl wait --for=condition=Available deployment/backend --namespace "$LAB_NAMESPACE" --timeout=120s
```

```bash
# Wait for the database Deployment to become available.
kubectl wait --for=condition=Available deployment/database --namespace "$LAB_NAMESPACE" --timeout=120s
```

```bash
# Wait for the frontend Pod to become Ready.
kubectl wait --for=condition=Ready pod/frontend --namespace "$LAB_NAMESPACE" --timeout=120s
```

---

## Step 4 — Demonstrate connectivity before policy

Test frontend → backend.

```bash
# Confirm frontend can resolve and reach the backend Service before a policy is applied.
kubectl exec --namespace "$LAB_NAMESPACE" pod/frontend -- wget -qO- --timeout=5 http://backend:8080 | head
```

Expected result: HTML returned by nginx.

Test backend → database by executing from the backend Pod.

```bash
# Confirm backend can reach the database test Service before a policy is applied.
kubectl exec --namespace "$LAB_NAMESPACE" deployment/backend -- wget -qO- --timeout=5 http://database:8080 | head
```

Expected result: HTML returned by nginx.

---

## Step 5 — Apply default deny

The default-deny policy isolates both ingress and egress inside this namespace.

### Manifest — `manifests/network-policy/default-deny.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: ${LAB_NAMESPACE}
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

Apply the default-deny policy.

```bash
# Block all ingress and egress for Pods in the student namespace until an allow policy permits it.
envsubst < manifests/network-policy/default-deny.yaml | kubectl apply -f -
```

List the active policy.

```bash
# Confirm the namespace now contains the default-deny policy.
kubectl get networkpolicy --namespace "$LAB_NAMESPACE"
```

Retry frontend → backend.

```bash
# Demonstrate that the previously working frontend-to-backend path is now blocked.
kubectl exec --namespace "$LAB_NAMESPACE" pod/frontend -- wget -qO- --timeout=5 http://backend:8080
```

Expected result: the request times out or fails because the network path is no longer allowed.

---

## Step 6 — Allow DNS

Without DNS egress, Service-name resolution can fail even when the final application traffic is otherwise permitted.

### Manifest — `manifests/network-policy/allow-dns.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: ${LAB_NAMESPACE}
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Apply the DNS policy.

```bash
# Restore only the DNS dependency required by the application Pods.
envsubst < manifests/network-policy/allow-dns.yaml | kubectl apply -f -
```

Test DNS resolution.

```bash
# Confirm the frontend can resolve the backend Service name again.
kubectl exec --namespace "$LAB_NAMESPACE" pod/frontend -- nslookup backend
```

> **Production note:** The exact DNS labels can vary by cluster implementation. The standard AKS/CoreDNS label is commonly `k8s-app=kube-dns`, but inspect the cluster if DNS is not reachable.

---

## Step 7 — Allow frontend → backend

We explicitly allow the frontend Pod to reach the backend Service on TCP/8080.

### Manifest — `manifests/network-policy/allow-frontend-backend.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-backend
  namespace: ${LAB_NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-frontend
  namespace: ${LAB_NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

Apply the frontend/backend policy pair.

```bash
# Restore only frontend-to-backend traffic on the backend Service port.
envsubst < manifests/network-policy/allow-frontend-backend.yaml | kubectl apply -f -
```

Test the path.

```bash
# Confirm frontend-to-backend connectivity works again and nothing broader was restored.
kubectl exec --namespace "$LAB_NAMESPACE" pod/frontend -- wget -qO- --timeout=5 http://backend:8080 | head
```

---

## Step 8 — Allow backend → database

Now permit only the backend-to-database path.

### Manifest — `manifests/network-policy/allow-backend-database.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-database
  namespace: ${LAB_NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-from-backend
  namespace: ${LAB_NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 8080
```

Apply the backend/database policy pair.

```bash
# Restore only backend-to-database communication on TCP/8080.
envsubst < manifests/network-policy/allow-backend-database.yaml | kubectl apply -f -
```

Test backend → database again.

```bash
# Confirm the backend can reach the database test service.
kubectl exec --namespace "$LAB_NAMESPACE" deployment/backend -- wget -qO- --timeout=5 http://database:8080 | head
```

---

## Step 9 — Validate the zero-trust pattern

List every NetworkPolicy in the namespace.

```bash
# Show the complete namespace-scoped policy set that now defines the allowed paths.
kubectl get networkpolicy --namespace "$LAB_NAMESPACE"
```

Describe the frontend policy.

```bash
# Inspect the exact frontend egress rule and its destination selector.
kubectl describe networkpolicy allow-frontend-backend --namespace "$LAB_NAMESPACE"
```

Describe the database policy.

```bash
# Inspect the exact backend-to-database ingress rule.
kubectl describe networkpolicy allow-database-from-backend --namespace "$LAB_NAMESPACE"
```

Test frontend → database directly. This path should remain blocked because there is no direct allow rule.

```bash
# Prove that the frontend does not have a direct path to the database.
kubectl exec --namespace "$LAB_NAMESPACE" pod/frontend -- wget -qO- --timeout=5 http://database:8080
```

Expected result: timeout/failure.

## Teaching point

A useful NetworkPolicy set is not about writing the largest possible policy. It is about being able to explain every permitted network path in one sentence.
