## Vault on Kubernetes — Troubleshooting Guide
### 1. Pod Pending / Not Scheduling

**Symptom:**

Vault pod stays in Pending status.
```sh
kubectl get pods -n vault
NAME                     READY   STATUS    RESTARTS   AGE
vault-xxxxx              0/1     Pending   0          2m
```

**Steps to troubleshoot:**

#### Check nodeSelector / affinity:

kubectl describe pod vault-xxxxx -n vault


#### Look for events like:

0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector


#### Verify node labels:
```sh
kubectl get nodes -o custom-columns=NAME:.metadata.name,TYPE:.metadata.labels.type
```

#### Fix Deployment nodeSelector:

Ensure your nodeSelector matches your node label:
```sh
nodeSelector:
  type: dependent_services
```

Or add a label to the node:
```sh
kubectl label node <node-name> type=dependent_services --overwrite
```
---
### 2. Pod CrashLoopBackOff

**Symptom:**

Vault pod starts and immediately crashes:
```sh
kubectl get pods -n vault
vault-xxxxx 0/1 CrashLoopBackOff
```

**Steps to troubleshoot:**

#### Check logs:
```sh
kubectl logs vault-xxxxx -n vault
```

**Common errors:**

- A storage backend must be specified → Missing storage backend in non-dev mode.
- You cannot specify a custom root token ID outside of "dev" mode → Using VAULT_DEV_ROOT_TOKEN_ID without -dev.
- Couldn't start vault with IPC_LOCK → In dev mode, can usually ignore.

#### Check pod description for events:
```sh
kubectl describe pod vault-xxxxx -n vault
```

#### Fix Deployment YAML for dev mode:
```sh
args:
  - "server"
  - "-dev"
  - "-dev-root-token-id=root"
```

#### Set environment variables inside pod / CLI:
```sh
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'
```
---
### 3. Vault CLI “server gave HTTP response to HTTPS client”

**Symptom:**
```sh
http: server gave HTTP response to HTTPS client
```

**Cause:**
Vault dev server runs over HTTP, but Vault CLI defaults to HTTPS.

Fix:
```sh
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'
vault status
```
---
### 4. Cannot exec into Vault pod

**Symptom:**

```sh
kubectl exec -it deploy/vault -n vault -- /bin/sh
error: Internal error occurred: unable to upgrade connection: container not found ("vault")
```

**Cause:**
Trying to exec into Deployment instead of Pod, or pod is not running.

**Fix:**

#### Get pod name:
```sh
kubectl get pods -n vault
```

#### Exec into pod:
```sh
kubectl exec -it <pod-name> -n vault -- /bin/sh
```

#### If pod is in CrashLoopBackOff, exec won’t work — check logs instead:
```sh
kubectl logs <pod-name> -n vault
```
---
### 5. Vault Secrets Not Saving / Updating

**Symptom:**
Secrets do not appear or return HTTP errors.

**Steps to troubleshoot:**

#### Verify Vault dev mode is running:
```sh
vault status
```

Must show Initialized: true and Sealed: false.

#### Verify VAULT_ADDR and VAULT_TOKEN are set:
```sh
echo $VAULT_ADDR
echo $VAULT_TOKEN
```

#### Use correct KV version paths:
```sh
KV v2 stores data under secret/data/...

vault kv put secret/student-api/app FLASK_ENV="production" ...
vault kv get secret/student-api/app
```
---
### 6. Pod Restart / Secrets Lost in Dev Mode

**Cause:**
Vault dev mode uses in-memory storage; pod restart deletes all secrets.

**Fix / Recommendation:**

#### Use a persistent storage backend for production:
```sh
storage "file" {
  path = "/vault/data"
}
```
- Or use ExternalSecrets / Kubernetes Secrets to persist secrets outside the Vault pod.
---
### 7. General Commands for Debugging

```sh
# Check pods & status
kubectl get pods -n vault
kubectl describe pod <pod-name> -n vault

# Check node labels
kubectl get nodes --show-labels
kubectl get nodes -o custom-columns=NAME:.metadata.name,TYPE:.metadata.labels.type

# Logs
kubectl logs <pod-name> -n vault
kubectl logs -f <pod-name> -n vault

# Exec into pod
kubectl exec -it <pod-name> -n vault -- /bin/sh

# Restart pods
kubectl delete pod -n vault <pod-name>
kubectl scale deployment vault -n vault --replicas=0
kubectl scale deployment vault -n vault --replicas=1
```

## Vault + External Secrets Operator (ESO) Troubleshooting

### 1. Initial Setup & Issue

- `Action`: Applied external-secrets.yaml.
- `Observation`: kubectl get secrets -n student-api showed no secrets, even after applying the ExternalSecret.
- `Issue`: ESO did not create the target secret (student-api-secret).
- `Error in ESO logs`:
  - failed to get API group resources: unable to retrieve the complete list of server APIs: external-secrets.io/v1beta1: the server could not find the requested resource
- `Analysis`:
  - ESO pod was using an old image (v0.9.11) that referenced v1beta1 instead of v1.
  - ServiceAccount permissions were insufficient for cluster-wide operations.

---

### 2. ESO Permissions Issue

- `Observation`: ESO logs showed:
- `secrets is forbidden`: User "system:serviceaccount:external-secrets:default" cannot list resource "secrets" in API group "" at the cluster scope

- `Analysis`: ESO ServiceAccount lacked the necessary ClusterRole and ClusterRoleBinding for accessing secrets across namespaces.

- `Solution`:  
  - Created a ServiceAccount, ClusterRole, and ClusterRoleBinding for ESO in external-secrets.yaml.
  - Updated Deployment to use serviceAccountName: external-secrets-sa.
  - Restarted the ESO Deployment.

---
### 3. ESO Image Version Mismatch

- `Observation`: Current Deployment image: ghcr.io/external-secrets/external-secrets:v0.9.11
- `Issue`: This version uses old API v1beta1, causing cache & CRD errors.
- `Solution`: Upgraded to v0.19.2 using Helm:

```sh
helm uninstall external-secrets -n external-secrets
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --version 0.19.2
```
  - ESO pods came up correctly with v0.19.2.

---

### 4. CRD Verification

- `Command`:

```sh
kubectl get crds | grep external-secrets
```
- `Observation`: CRDs for externalsecrets.external-secrets.io and clustersecretstores.external-secrets.io present.
- `Conclusion`: ESO CRDs were correctly installed.

### 5. ExternalSecret & Secret Verification

- Applied external-secrets.yaml after permissions fix and ESO upgrade.

- `Result`:

```sh
kubectl get externalsecrets -n student-api
NAME                  STORETYPE            STORE           REFRESH INTERVAL   STATUS         READY
student-api-secrets   ClusterSecretStore   vault-backend   1h                 SecretSynced   True

kubectl get secrets -n student-api
NAME                 TYPE     DATA   AGE
student-api-secret   Opaque   12     50s
```
- Secret contains all Vault values in base64-encoded form.

### 7. Key Troubleshooting Steps Recap
- Issue	Observation / Error	Root Cause	Fix
- Secrets not created	No resources found in student-api namespace	ESO lacked cluster permissions	Created SA, ClusterRole, ClusterRoleBinding
- ESO logs v1beta1 not found	failed to get API group resources ... v1beta1	ESO image too old	Upgraded ESO Helm chart to v0.19.2
- Vault command failing	/bin/sh: /: Permission denied	Extra characters in shell	Corrected multi-line command
- Deployment update fails	spec.selector field is immutable	Trying to kubectl apply on existing deployment	Reinstalled ESO via Helm
- Secrets not reflecting	SecretSynced = False	ESO not restarted, permissions missing	Restarted deployment after fixes

### Milestone Status

- Vault running and storing secrets.
- ESO installed and correctly configured.
- ExternalSecret successfully syncs Vault secrets to student-api-secret.
- Secrets available for application pods.
