# Kubernetes Production Deployment Guide: Flask REST API Stack with Vault & External Secrets

This README provides a comprehensive guide for deploying your REST API stack on Kubernetes with secure secrets management via HashiCorp Vault and External Secrets Operator (ESO).

---

## 1. Project Kubernetes Overview for Production Deployment

You are deploying a microservices stack on Kubernetes for production. This setup includes:

- **Namespaces** for logical isolation (`student-api` for app & DB, `vault` for secret storage, `external-secrets` for ESO).
- **ConfigMaps** for non-sensitive configuration (DB host, port, etc).
- **External Secrets Operator (ESO)** to inject secrets from Vault into K8s.
- **HashiCorp Vault** as the secrets backend.
- **Init Containers** to run DB migrations before app startup.
- **Multiple Replicas** for the Flask API for high availability.
- **NodeSelector** to place workloads on dedicated nodes.
- **Services** to expose the API and DB within/outside the cluster.

### Why this Architecture?
- **Security:** Secrets never touch disk, only injected at runtime.
- **Scalability:** K8s manages scaling, restarts, health, and upgrades.
- **Reliability:** Automated deployments, rolling updates, and healthchecks.
- **Maintainability:** Declarative manifests (YAML) for all resources.

---

## 2. Manifest Breakdown: Vault, ESO, Application, Database

### **A. Vault Manifests (`vault.yaml`)**

```yaml
# Namespace for Vault
apiVersion: v1
kind: Namespace
metadata:
  name: vault
---
# Deployment for Vault
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault
  namespace: vault
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vault
  template:
    metadata:
      labels:
        app: vault
    spec:
      nodeSelector:
        type: dependent_services
      containers:
        - name: vault
          image: hashicorp/vault:1.17.3
          args:
            - "server"
            - "-dev"
          ports:
            - containerPort: 8200
          env:
            - name: VAULT_DEV_ROOT_TOKEN_ID
              value: "root"
            - name: VAULT_DEV_LISTEN_ADDRESS
              value: "0.0.0.0:8200"
          readinessProbe:
            httpGet:
              path: /v1/sys/health
              port: 8200
            initialDelaySeconds: 5
            periodSeconds: 10
---
# Service for Vault
apiVersion: v1
kind: Service
metadata:
  name: vault
  namespace: vault
spec:
  selector:
    app: vault
  ports:
    - name: http
      port: 8200
      targetPort: 8200
  type: ClusterIP
```
- **Namespace:** `vault`
- **Deployment:** Runs Vault in dev mode (`hashicorp/vault:1.17.3`), exposes port 8200, readiness probe checks health.
- **Service:** ClusterIP for Vault, only reachable inside cluster.

#### Implementation Steps 

```sh
ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get pods -n vault
NAME                    READY   STATUS    RESTARTS      AGE
vault-7df59997d-xhzkg   1/1     Running   1 (29h ago)   2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl apply -f vault.yaml 
namespace/vault created
deployment.apps/vault created
service/vault created

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl get pods -n vault
NAME                    READY   STATUS    RESTARTS   AGE
vault-7df59997d-qw949   1/1     Running   0          28s

	1. Use Vault CLI inside the Vault pod (recommended for your dev setup)

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl exec -n vault -it vault-7df59997d-qw949 -- /bin/sh

/ # export VAULT_ADDR='http://127.0.0.1:8200'
/ # export VAULT_TOKEN='root'

/ # vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.17.3
Build Date      2024-08-06T14:28:45Z
Storage Type    inmem
Cluster Name    vault-cluster-892a5efd
Cluster ID      6566fca3-710b-12c5-8b6a-2dcf922a4596
HA Enabled      false

/ # vault secrets enable -path=secret kv-v2
Error enabling: Error making API request.

URL: POST http://127.0.0.1:8200/v1/sys/mounts/secret
Code: 400. Errors:

* path is already in use at secret/

/ # vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_98c7c7d8    per-token private secret storage
identity/     identity     identity_b79c80e0     identity store
secret/       kv           kv_2706ef0c           key/value secret storage
sys/          system       system_18eeb68c       system endpoints used for control, policy and debugging

/ # vault kv put secret/studentdb POSTGRES_USER=postgres POSTGRES_PASSWORD=postgres123

==== Secret Path ====
secret/data/studentdb

======= Metadata =======
Key                Value
---                -----
created_time       2025-09-17T06:51:38.585606479Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
/ # vault kv get secret/studentdb
==== Secret Path ====
secret/data/studentdb

======= Metadata =======
Key                Value
---                -----
created_time       2025-09-17T06:51:38.585606479Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

========== Data ==========
Key                  Value
---                  -----
POSTGRES_PASSWORD    postgres123
POSTGRES_USER        postgres

/ # vault kv get -format=json secret/studentdb
{
  "request_id": "310740ec-1bf4-09cd-882d-5779b296f34e",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "data": {
      "POSTGRES_PASSWORD": "postgres123",
      "POSTGRES_USER": "postgres"
    },
    "metadata": {
      "created_time": "2025-09-17T06:51:38.585606479Z",
      "custom_metadata": null,
      "deletion_time": "",
      "destroyed": false,
      "version": 1
    }
  },
  "warnings": null,
  "mount_type": "kv"
}

/ # 
cat > /tmp/eso-policy.hcl <<'EOF'
# Allow read of the DB secret stored at secret/data/studentdb (KV v2)
path "secret/data/studentdb" {
  capabilities = ["read"]
}
EOF

/ # vault policy write eso-policy /tmp/eso-policy.hcl
Success! Uploaded policy: eso-policy

/ # vault policy list
default
eso-policy
root

/ # vault token create -policy=eso-policy -period=24h -orphan -format=json
{
  "request_id": "7c4110b3-76bd-1710-4d4e-450afa062bba",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": null,
  "warnings": null,
  "auth": {
    "client_token": "hvs.CAESIHGbl96lzeJ0GdqYQEWYaZ4hLNS0_fyQRhXfXr3_2GAjGh4KHGh2cy5nRVk2SWZDRTdHNFdqVWxXZUZwSEM0TXE",
    "accessor": "dYANXAaiDDGPz2gmNirc2XQ2",
    "policies": [
      "default",
      "eso-policy"
    ],
    "token_policies": [
      "default",
      "eso-policy"
    ],
    "identity_policies": null,
    "metadata": null,
    "orphan": true,
    "entity_id": "",
    "lease_duration": 86400,
    "renewable": true,
    "mfa_requirement": null
  },
  "mount_type": "token"
}

/ # exit

	1. Create a Kubernetes Secret for ESO with that token
	
ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl create ns external-secrets

	2. create a K8s secret with the Vault token (replace <VAULT_TOKEN> with the token you copied):
	
ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl create secret generic vault-token --from-literal=token=hvs.CAESIHGbl96lzeJ0GdqYQEWYaZ4hLNS0_fyQRhXfXr3_2GAjGh4KHG
h2cy5nRVk2SWZDRTdHNFdqVWxXZUZwSEM0TXE -n external-secrets
secret/vault-token created
ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ 

Port-forward + curl (verify from host)

You can also verify the secret via HTTP from your host (useful for quick checks):
	1. port-forward Vault service to localhost (run in separate terminal):


ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl port-forward -n vault svc/vault 8200:8200

Forwarding from 127.0.0.1:8200 -> 8200
Forwarding from [::1]:8200 -> 8200
Handling connection for 8200

	2. Read the secret (KV v2 read path is /v1/secret/data/<name>):
	
ubuntu@ip-10-0-6-246:~/Flask-REST-API$ curl --header "X-Vault-Token: root" \
  http://127.0.0.1:8200/v1/secret/data/studentdb | jq .

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 100   380  100   380    0     0   1163      0 --:--:-- --:--:-- --:--:--  1165
{
  "request_id": "edbba646-a2b7-05b2-00fb-eca510bb4748",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "data": {
      "POSTGRES_PASSWORD": "postgres123",
      "POSTGRES_USER": "postgres"
    },
    "metadata": {
      "created_time": "2025-09-17T06:51:38.585606479Z",
      "custom_metadata": null,
      "deletion_time": "",
      "destroyed": false,
      "version": 1
    }
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null,
  "mount_type": "kv"
}

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ 
```

### **B. External Secrets Operator (`external-secrets.yaml`)**

```yaml
# Namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: external-secrets
---
apiVersion: v1
kind: Namespace
metadata:
  name: student-api
---
# ServiceAccount for ESO
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets
  namespace: external-secrets
---
# ClusterRole for ESO
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-secrets-cluster-role
rules:
  - apiGroups: [""]
    resources: ["secrets", "namespaces"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["external-secrets.io"]
    resources: ["secretstores", "clustersecretstores", "externalsecrets", "clusterexternalsecrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
# ClusterRoleBinding for ESO
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-secrets-cluster-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-secrets-cluster-role
subjects:
  - kind: ServiceAccount
    name: external-secrets
    namespace: external-secrets
---
# Vault token secret (in external-secrets namespace)
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
  namespace: external-secrets
type: Opaque
stringData:
  token: "root"
---
# ESO deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-secrets-operator
  namespace: external-secrets
spec:
  replicas: 1
  selector:
    matchLabels:
      app: external-secrets
  template:
    metadata:
      labels:
        app: external-secrets
    spec:
      serviceAccountName: external-secrets
      nodeSelector:
        type: dependent_services
      containers:
        - name: external-secrets-operator
          image: ghcr.io/external-secrets/external-secrets:v0.9.9
          imagePullPolicy: IfNotPresent
---
# ClusterSecretStore pointing to Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-secretstore
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        tokenSecretRef:
          name: vault-token
          key: token
          namespace: external-secrets  
---
# ExternalSecret pulling Postgres credentials
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: postgres-secret
  namespace: student-api
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: vault-secretstore
    kind: ClusterSecretStore
  target:
    name: postgres-secret
    creationPolicy: Owner
  data:
    - secretKey: POSTGRES_USER
      remoteRef:
        key: studentdb
        property: POSTGRES_USER
    - secretKey: POSTGRES_PASSWORD
      remoteRef:
        key: studentdb
        property: POSTGRES_PASSWORD
```
- **Namespace:** `external-secrets`
- **ServiceAccount, ClusterRole, ClusterRoleBinding:** Grants ESO controller permissions to read/write secrets, CRDs.
- **Vault Token Secret:** Stores Vault token for ESO.
- **ESO Deployment:** Runs ESO controller, scheduled on dependent_services node.
- **ClusterSecretStore:** Tells ESO how to connect to Vault (URL, token, path).
- **ExternalSecret:** Specifies which secrets to sync (POSTGRES_USER, POSTGRES_PASSWORD) and where to create them (`student-api` namespace).

#### Implementation Steps 
```sh
# Install official ESO CRDs + controller (recommended way):

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl apply -f https://github.com/external-secrets/external-secrets/releases/download/v0.9.9/external-secrets.yaml

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl apply -f external-secrets.yaml
namespace/external-secrets created
deployment.apps/external-secrets-operator created
serviceaccount/external-secrets created
secret/vault-token configured
secretstore.external-secrets.io/vault-secretstore created

# Now confirm ESO is running properly:

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl get pods -n external-secrets
NAME                                         READY   STATUS    RESTARTS      AGE
external-secrets-operator-6d9df787d6-8ng48   1/1     Running   2 (29s ago)   4m36s

# Also check CRDs are installed:

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl get crd | grep external-secrets
acraccesstokens.generators.external-secrets.io          2025-09-15T19:37:00Z
clusterexternalsecrets.external-secrets.io              2025-09-15T19:37:00Z
clustergenerators.generators.external-secrets.io        2025-09-15T19:37:00Z
clusterpushsecrets.external-secrets.io                  2025-09-15T19:37:00Z
clustersecretstores.external-secrets.io                 2025-09-17T08:25:50Z
ecrauthorizationtokens.generators.external-secrets.io   2025-09-15T19:37:00Z
externalsecrets.external-secrets.io                     2025-09-17T08:25:50Z
fakes.generators.external-secrets.io                    2025-09-15T19:37:00Z
gcraccesstokens.generators.external-secrets.io          2025-09-15T19:37:00Z
generatorstates.generators.external-secrets.io          2025-09-15T19:37:00Z
githubaccesstokens.generators.external-secrets.io       2025-09-15T19:37:00Z
grafanas.generators.external-secrets.io                 2025-09-15T19:37:00Z
mfas.generators.external-secrets.io                     2025-09-15T19:37:00Z
passwords.generators.external-secrets.io                2025-09-15T19:37:00Z
pushsecrets.external-secrets.io                         2025-09-15T19:37:00Z
quayaccesstokens.generators.external-secrets.io         2025-09-15T19:37:00Z
secretstores.external-secrets.io                        2025-09-17T08:25:50Z
sshkeys.generators.external-secrets.io                  2025-09-15T19:37:00Z
stssessiontokens.generators.external-secrets.io         2025-09-15T19:37:00Z
uuids.generators.external-secrets.io                    2025-09-15T19:37:00Z
vaultdynamicsecrets.generators.external-secrets.io      2025-09-15T19:37:00Z
webhooks.generators.external-secrets.io                 2025-09-15T19:37:00Z


ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl apply -f external-secrets.yaml
namespace/external-secrets unchanged
deployment.apps/external-secrets-operator unchanged
serviceaccount/external-secrets unchanged
secret/vault-token configured
secretstore.external-secrets.io/vault-secretstore created

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl get secretstore -n external-secrets
NAME                AGE   STATUS   CAPABILITIES   READY
vault-secretstore   11s   Valid    ReadWrite      True

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl describe secretstore vault-secretstore -n external-secrets
Name:         vault-secretstore
Namespace:    external-secrets
Labels:       <none>
Annotations:  <none>
API Version:  external-secrets.io/v1beta1
Kind:         SecretStore
Metadata:
  Creation Timestamp:  2025-09-17T08:32:40Z
  Generation:          1
  Resource Version:    361751
  UID:                 d065e847-7f25-4d82-b5f7-d775fd274c5c
Spec:
  Provider:
    Vault:
      Auth:
        Token Secret Ref:
          Key:   token
          Name:  vault-token
      Path:      secret
      Server:    http://vault.vault.svc.cluster.local:8200
      Version:   v2
Status:
  Capabilities:  ReadWrite
  Conditions:
    Last Transition Time:  2025-09-17T08:32:40Z
    Message:               store validated
    Reason:                Valid
    Status:                True
    Type:                  Ready
Events:
  Type    Reason  Age                From          Message
  ----    ------  ----               ----          -------
  Normal  Valid   21s (x2 over 21s)  secret-store  store validated

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl get secrets -n student-api
NAME              TYPE     DATA   AGE
postgres-secret   Opaque   2      2m6s

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl get secrets -n vault
No resources found in vault namespace.

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl get pods -n vault
NAME                    READY   STATUS    RESTARTS   AGE
vault-7df59997d-m79r8   1/1     Running   0          59m

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl get secret postgres-secret -n student-api -o jsonpath='{.data.POSTGRES_USER}' | base64 -d; echo
kubectl get secret postgres-secret -n student-api -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d; echo
postgres
postgres123

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ kubectl get secretstore -n external-secrets
NAME                AGE   STATUS   CAPABILITIES   READY
vault-secretstore   48m   Valid    ReadWrite      True

ubuntu@ip-10-0-6-246:~/Flask-REST-API/k8s$ 
```

### **C. Application Manifest (`application.yaml`)**

```yaml
# Namespace for application
apiVersion: v1
kind: Namespace
metadata:
  name: student-api
---
# ConfigMap for Flask API
apiVersion: v1
kind: ConfigMap
metadata:
  name: flask-config
  namespace: student-api
data:
  POSTGRES_HOST: "postgres"
  POSTGRES_PORT: "5432"
  POSTGRES_DB: "studentdb"
---
# Deployment for Flask REST API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-api
  namespace: student-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-api
  template:
    metadata:
      labels:
        app: flask-api
    spec:
      nodeSelector:
        type: application
      initContainers:
        - name: db-migrations
          image: akhilthyadi/flask-app:7.0.0
          workingDir: /api/app
          env:
            - name: FLASK_APP
              value: "wsgi.py"
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_HOST
              valueFrom:
                configMapKeyRef:
                  name: flask-config
                  key: POSTGRES_HOST
            - name: POSTGRES_PORT
              valueFrom:
                configMapKeyRef:
                  name: flask-config
                  key: POSTGRES_PORT
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: flask-config
                  key: POSTGRES_DB
          command:
            - sh
            - -c
            - |
              echo "Waiting for Postgres at $POSTGRES_HOST:$POSTGRES_PORT..."
              until pg_isready -h $POSTGRES_HOST -p $POSTGRES_PORT -U $POSTGRES_USER; do
                sleep 2
              done
              echo "Postgres is ready, running migrations..."
              flask db upgrade
      containers:
        - name: flask-api
          image: akhilthyadi/flask-app:7.0.0
          ports:
            - containerPort: 5000
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_HOST
              valueFrom:
                configMapKeyRef:
                  name: flask-config
                  key: POSTGRES_HOST
            - name: POSTGRES_PORT
              valueFrom:
                configMapKeyRef:
                  name: flask-config
                  key: POSTGRES_PORT
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: flask-config
                  key: POSTGRES_DB
---
# Service to expose Flask API
apiVersion: v1
kind: Service
metadata:
  name: flask-api-service
  namespace: student-api
spec:
  selector:
    app: flask-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: NodePort
```
- **Namespace:** `student-api`
- **ConfigMap:** Non-sensitive DB connection info.
- **Deployment:**
  - 2 replicas of Flask API (high availability).
  - **InitContainer:** Waits for DB, runs migrations (with Flask-Migrate).
  - **Secrets & ConfigMap:** Injected as environment variables.
  - **NodeSelector:** Schedules API pods on application nodes.
- **Service:** NodePort service exposes API to outside traffic.

#### Implementation Steps 
```sh
ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get pods -n student-api
NAME                         READY   STATUS    RESTARTS      AGE
flask-api-58b96b78fb-6nhh2   1/1     Running   2 (26h ago)   2d1h
flask-api-58b96b78fb-thr4f   1/1     Running   2 (26h ago)   2d1h
postgres-7cbd777dfd-kd9n8    1/1     Running   2 (26h ago)   2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get deployment -n student-api
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
flask-api   2/2     2            2           2d1h
postgres    1/1     1            1           2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl rollout status deployment/flask-api -n student-api
deployment "flask-api" successfully rolled out

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get svc -n student-api
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
flask-api-service   NodePort    10.107.202.160   <none>        80:31954/TCP   2d1h
postgres            ClusterIP   10.100.201.79    <none>        5432/TCP       2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get configmap -n student-api
NAME               DATA   AGE
flask-config       3      2d1h
kube-root-ca.crt   1      2d1h
postgres-config    3      2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get secret -n student-api
NAME              TYPE     DATA   AGE
postgres-secret   Opaque   2      2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl logs -n student-api flask-api-58b96b78fb-6nhh2
Defaulted container "flask-api" out of: flask-api, db-migrations (init)
[2025-10-08 17:19:09 +0000] [1] [INFO] Starting gunicorn 23.0.0
[2025-10-08 17:19:09 +0000] [1] [INFO] Listening at: http://0.0.0.0:5000 (1)
[2025-10-08 17:19:09 +0000] [1] [INFO] Using worker: sync
[2025-10-08 17:19:09 +0000] [8] [INFO] Booting worker with pid: 8
[2025-10-08 17:19:09 +0000] [9] [INFO] Booting worker with pid: 9

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl exec -n student-api -it flask-api-58b96b78fb-6nhh2 -- /bin/sh
Defaulted container "flask-api" out of: flask-api, db-migrations (init)
/api # echo $DB_HOST $DB_PORT $DB_USER $DB_PASSWORD

/api # exit

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl port-forward -n student-api svc/flask-api-service 5000:80
Forwarding from 127.0.0.1:5000 -> 5000
Forwarding from [::1]:5000 -> 5000
Handling connection for 5000
Handling connection for 5000


ubuntu@ip-10-0-6-246:~/Flask-REST-API$ curl http://localhost:5000/health
{"status":"ok"}

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ curl http://localhost:5000/students
[]

ubuntu@ip-10-0-6-246:~/Flask-REST-API$
```

### **D. Database Manifest (`database.yaml`)**

```yaml
# Namespace for database
apiVersion: v1
kind: Namespace
metadata:
  name: student-api
---
# ConfigMap for database host, port, and DB name
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: student-api
data:
  POSTGRES_HOST: postgres
  POSTGRES_PORT: "5432"
  POSTGRES_DB: studentdb
---
# PersistentVolumeClaim for Postgres storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: student-api
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
# Deployment for Postgres
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: student-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      nodeSelector:
        type: database
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            # Secrets for username & password
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            # ConfigMap for DB name, host, port
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: postgres-config
                  key: POSTGRES_DB
            - name: POSTGRES_HOST
              valueFrom:
                configMapKeyRef:
                  name: postgres-config
                  key: POSTGRES_HOST
            - name: POSTGRES_PORT
              valueFrom:
                configMapKeyRef:
                  name: postgres-config
                  key: POSTGRES_PORT
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
---
# Service to expose Postgres
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: student-api
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
  type: ClusterIP
```
- **Namespace:** `student-api`
- **ConfigMap:** DB host, port, name.
- **PersistentVolumeClaim:** For durable Postgres storage.
- **Deployment:** One replica of Postgres, secrets injected, scheduled on database nodes.
- **Service:** ClusterIP for Postgres, only accessible inside cluster.

#### Implementation Steps 
```sh
ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get pods -n vault
NAME                    READY   STATUS    RESTARTS      AGE
vault-7df59997d-xhzkg   1/1     Running   1 (29h ago)   2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get ns
NAME               STATUS   AGE
argocd             Active   27h
default            Active   15d
external-secrets   Active   2d1h
kube-node-lease    Active   15d
kube-public        Active   15d
kube-system        Active   15d
student-api        Active   2d1h
vault              Active   2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get configmap postgres-config -n student-api -o yaml
apiVersion: v1
data:
  POSTGRES_DB: studentdb
  POSTGRES_HOST: postgres
  POSTGRES_PORT: "5432"
kind: ConfigMap
metadata:
  annotations:
    argocd.argoproj.io/tracking-id: student-db:/ConfigMap:student-api/postgres-config
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"POSTGRES_DB":"studentdb","POSTGRES_HOST":"postgres","POSTGRES_PORT":"5432"},"kind":"ConfigMap","metadata":{"annotations":{"argocd.argoproj.io/tracking-id":"student-db:/ConfigMap:student-api/postgres-config"},"name":"postgres-config","namespace":"student-api"}}
  creationTimestamp: "2025-10-06T15:42:11Z"
  name: postgres-config
  namespace: student-api
  resourceVersion: "202094"
  uid: fda47856-d8e5-45ee-87b4-1b80a2d597c9

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get pvc -n student-api
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
postgres-pvc   Bound    pvc-4705cca6-3aa4-4357-b648-3cc6cca74956   1Gi        RWO            standard       <unset>                 2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get deploy -n student-api
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
flask-api   2/2     2            2           2d1h
postgres    1/1     1            1           2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get pods -n student-api
NAME                         READY   STATUS    RESTARTS      AGE
flask-api-58b96b78fb-6nhh2   1/1     Running   2 (26h ago)   2d1h
flask-api-58b96b78fb-thr4f   1/1     Running   2 (26h ago)   2d1h
postgres-7cbd777dfd-kd9n8    1/1     Running   2 (26h ago)   2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl exec -it postgres-7cbd777dfd-kd9n8 -n student-api -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/postgresql/15/bin
HOSTNAME=postgres-7cbd777dfd-kd9n8
TERM=xterm
POSTGRES_PASSWORD=postgres123
POSTGRES_DB=studentdb
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_SERVICE_PORT=5432
POSTGRES_PORT_5432_TCP=tcp://10.100.201.79:5432
POSTGRES_PORT_5432_TCP_PORT=5432
KUBERNETES_PORT_443_TCP_PORT=443
FLASK_API_SERVICE_SERVICE_HOST=10.107.202.160
FLASK_API_SERVICE_SERVICE_PORT=80
FLASK_API_SERVICE_PORT_80_TCP=tcp://10.107.202.160:80
POSTGRES_PORT_5432_TCP_ADDR=10.100.201.79
KUBERNETES_PORT=tcp://10.96.0.1:443
FLASK_API_SERVICE_PORT_80_TCP_ADDR=10.107.202.160
POSTGRES_PORT_5432_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
FLASK_API_SERVICE_PORT=tcp://10.107.202.160:80
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
FLASK_API_SERVICE_PORT_80_TCP_PROTO=tcp
FLASK_API_SERVICE_PORT_80_TCP_PORT=80
POSTGRES_SERVICE_HOST=10.100.201.79
GOSU_VERSION=1.18
LANG=en_US.utf8
PG_MAJOR=15
PG_VERSION=15.14-1.pgdg13+1
PGDATA=/var/lib/postgresql/data
HOME=/root

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get svc -n student-api
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
flask-api-service   NodePort    10.107.202.160   <none>        80:31954/TCP   2d1h
postgres            ClusterIP   10.100.201.79    <none>        5432/TCP       2d1h

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl exec -it flask-api-58b96b78fb-6nhh2 -n student-api -- pg_isready -h postgres -p 5432
Defaulted container "flask-api" out of: flask-api, db-migrations (init)
postgres:5432 - accepting connections
ubuntu@ip-10-0-6-246:~/Flask-REST-API$
```
---

### Complete setup check

```sh
ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get all --all-namespaces
NAMESPACE          NAME                                                    READY   STATUS             RESTARTS       AGE
argocd             pod/argocd-application-controller-0                     1/1     Running            1 (26h ago)    27h
argocd             pod/argocd-applicationset-controller-56b6c785db-6wqwv   1/1     Running            1 (26h ago)    27h
argocd             pod/argocd-dex-server-5ff788dd5b-2t664                  1/1     Running            1 (26h ago)    27h
argocd             pod/argocd-notifications-controller-5c45c8bdbc-tkf7w    1/1     Running            1 (26h ago)    27h
argocd             pod/argocd-redis-84db6f668f-zsgdc                       1/1     Running            1 (26h ago)    27h
argocd             pod/argocd-repo-server-56d55b9bf4-c4lkx                 1/1     Running            1 (26h ago)    27h
argocd             pod/argocd-server-7554484b9d-ptf5w                      1/1     Running            3 (24m ago)    27h
default            pod/external-secrets-7c4d6d64dd-fwmvt                   1/1     Running            13 (26h ago)   15d
default            pod/external-secrets-cert-controller-7798b5cb8d-pcpsh   1/1     Running            13 (26h ago)   15d
default            pod/external-secrets-webhook-98f9dbb87-sknjd            1/1     Running            13 (26h ago)   15d
external-secrets   pod/external-secrets-operator-79bf774bd6-9zq8t          0/1     Running            13 (32s ago)   26h
kube-system        pod/coredns-66bc5c9577-gb62v                            1/1     Running            15 (26h ago)   15d
kube-system        pod/etcd-minikube                                       1/1     Running            13 (26h ago)   15d
kube-system        pod/kindnet-gq422                                       1/1     Running            13 (26h ago)   15d
kube-system        pod/kindnet-kp2g5                                       1/1     Running            13 (26h ago)   15d
kube-system        pod/kindnet-nt8bj                                       1/1     Running            13 (26h ago)   15d
kube-system        pod/kube-apiserver-minikube                             1/1     Running            16 (26h ago)   15d
kube-system        pod/kube-controller-manager-minikube                    1/1     Running            13 (26h ago)   15d
kube-system        pod/kube-proxy-4bcb6                                    1/1     Running            13 (26h ago)   15d
kube-system        pod/kube-proxy-9796f                                    1/1     Running            13 (26h ago)   15d
kube-system        pod/kube-proxy-n5vld                                    1/1     Running            43 (25m ago)   15d
kube-system        pod/kube-scheduler-minikube                             1/1     Running            13 (26h ago)   15d
kube-system        pod/storage-provisioner                                 1/1     Running            42 (25m ago)   15d
student-api        pod/flask-api-58b96b78fb-6nhh2                          1/1     Running            2 (26h ago)    2d2h
student-api        pod/flask-api-58b96b78fb-thr4f                          1/1     Running            2 (26h ago)    2d2h
student-api        pod/postgres-7cbd777dfd-kd9n8                           1/1     Running            2 (26h ago)    2d2h
vault              pod/vault-7df59997d-xhzkg                               1/1     Running            2 (26h ago)    2d2h

NAMESPACE     NAME                                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
argocd        service/argocd-applicationset-controller   ClusterIP   10.108.253.118   <none>        7000/TCP                 27h
argocd        service/argocd-dex-server                  ClusterIP   10.111.206.7     <none>        5556/TCP,5557/TCP        27h
argocd        service/argocd-redis                       ClusterIP   10.106.225.252   <none>        6379/TCP                 27h
argocd        service/argocd-repo-server                 ClusterIP   10.107.107.251   <none>        8081/TCP                 27h
argocd        service/argocd-server                      ClusterIP   10.100.179.201   <none>        80/TCP,443/TCP           27h
default       service/external-secrets-webhook           ClusterIP   10.100.25.207    <none>        443/TCP                  15d
default       service/kubernetes                         ClusterIP   10.96.0.1        <none>        443/TCP                  15d
kube-system   service/kube-dns                           ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   15d
student-api   service/flask-api-service                  NodePort    10.107.202.160   <none>        80:31954/TCP             2d2h
student-api   service/postgres                           ClusterIP   10.100.201.79    <none>        5432/TCP                 2d2h
vault         service/vault                              ClusterIP   10.100.89.198    <none>        8200/TCP                 2d2h

NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/kindnet      3         3         3       3            3           <none>                   15d
kube-system   daemonset.apps/kube-proxy   3         3         3       3            3           kubernetes.io/os=linux   15d

NAMESPACE          NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
argocd             deployment.apps/argocd-applicationset-controller   1/1     1            1           27h
argocd             deployment.apps/argocd-dex-server                  1/1     1            1           27h
argocd             deployment.apps/argocd-notifications-controller    1/1     1            1           27h
argocd             deployment.apps/argocd-redis                       1/1     1            1           27h
argocd             deployment.apps/argocd-repo-server                 1/1     1            1           27h
argocd             deployment.apps/argocd-server                      1/1     1            1           27h
default            deployment.apps/external-secrets                   1/1     1            1           15d
default            deployment.apps/external-secrets-cert-controller   1/1     1            1           15d
default            deployment.apps/external-secrets-webhook           1/1     1            1           15d
external-secrets   deployment.apps/external-secrets-operator          0/1     1            0           2d2h
kube-system        deployment.apps/coredns                            1/1     1            1           15d
student-api        deployment.apps/flask-api                          2/2     2            2           2d2h
student-api        deployment.apps/postgres                           1/1     1            1           2d2h
vault              deployment.apps/vault                              1/1     1            1           2d2h

NAMESPACE          NAME                                                          DESIRED   CURRENT   READY   AGE
argocd             replicaset.apps/argocd-applicationset-controller-56b6c785db   1         1         1       27h
argocd             replicaset.apps/argocd-dex-server-5ff788dd5b                  1         1         1       27h
argocd             replicaset.apps/argocd-notifications-controller-5c45c8bdbc    1         1         1       27h
argocd             replicaset.apps/argocd-redis-84db6f668f                       1         1         1       27h
argocd             replicaset.apps/argocd-repo-server-56d55b9bf4                 1         1         1       27h
argocd             replicaset.apps/argocd-server-7554484b9d                      1         1         1       27h
default            replicaset.apps/external-secrets-7c4d6d64dd                   1         1         1       15d
default            replicaset.apps/external-secrets-cert-controller-7798b5cb8d   1         1         1       15d
default            replicaset.apps/external-secrets-webhook-98f9dbb87            1         1         1       15d
external-secrets   replicaset.apps/external-secrets-operator-79bf774bd6          1         1         0       2d2h
kube-system        replicaset.apps/coredns-66bc5c9577                            1         1         1       15d
student-api        replicaset.apps/flask-api-58b96b78fb                          2         2         2       2d2h
student-api        replicaset.apps/postgres-7cbd777dfd                           1         1         1       2d2h
vault              replicaset.apps/vault-7df59997d                               1         1         1       2d2h

NAMESPACE   NAME                                             READY   AGE
argocd      statefulset.apps/argocd-application-controller   1/1     27h
ubuntu@ip-10-0-6-246:~/Flask-REST-API$
```
---

## 3. Service Flow & Importance

### **Provisioning Flow:**
1. **Vault starts** in its own namespace, storing secrets.
2. **ESO starts** in `external-secrets`, connects to Vault via token.
3. **ClusterSecretStore** configures how ESO reads secrets from Vault.
4. **ExternalSecret** object in `student-api` namespace tells ESO to sync secrets (`POSTGRES_USER`, `POSTGRES_PASSWORD`) from Vault into a K8s Secret.
5. **Database Deployment** uses these secrets for secure DB startup.
6. **Application Deployment:**
   - **InitContainer** waits for DB, applies migrations.
   - Main container starts Flask API, reading credentials from K8s Secret and config from ConfigMap.
7. **Services** allow API and DB to be reached from cluster or externally.

### **Why this matters:**
- **Secret Rotation:** Change secrets in Vault, ESO automatically updates them in K8s.
- **Zero Trust:** App never sees secrets until runtime, not hardcoded or stored in images.
- **Auditing:** Vault logs secret access.
- **Separation of Concerns:** Each manifest manages only its component, easier to maintain.

---

## 4. All Kubernetes Commands Used

### **Apply & Verify Resources**
```sh
kubectl apply -f vault.yaml
kubectl apply -f external-secrets.yaml
kubectl apply -f database.yaml
kubectl apply -f application.yaml
```

### **Namespace Operations**
```sh
kubectl get ns
kubectl describe ns student-api
```

### **ConfigMap & Secret Checks**
```sh
kubectl get configmap -n student-api
kubectl describe configmap flask-config -n student-api

kubectl get secret -n student-api
kubectl describe secret postgres-secret -n student-api
kubectl get secret vault-token -n external-secrets
```

### **Pod & Deployment Monitoring**
```sh
kubectl get pods -n student-api -o wide
kubectl describe pod <flask-api-pod-name> -n student-api
kubectl get deploy -n student-api
kubectl get pods -n external-secrets
kubectl get pods -n vault
```

### **Logs (App, DB, InitContainer)**
```sh
kubectl logs <flask-api-pod-name> -n student-api -c db-migrations
kubectl logs <flask-api-pod-name> -n student-api -c flask-api
kubectl logs <postgres-pod-name> -n student-api
```

### **Service Discovery**
```sh
kubectl get svc -n student-api
kubectl describe svc flask-api-service -n student-api
kubectl port-forward svc/flask-api-service -n student-api 5000:80
curl http://localhost:5000/health
```

### **NodePort Access**
```sh
NODE_IP=$(kubectl get nodes -o wide | awk 'NR==2{print $6}')
NODE_PORT=$(kubectl get svc flask-api-service -n student-api -o jsonpath='{.spec.ports[0].nodePort}')
curl http://$NODE_IP:$NODE_PORT/health
```

### **Database Checks**
```sh
kubectl get pods -n student-api -l app=postgres
kubectl logs <postgres-pod-name> -n student-api
kubectl exec -it <postgres-pod-name> -n student-api -- psql -U <db_user> -d studentdb
\dt
```

### **ESO + Vault Checks**
```sh
# Check CRDs
kubectl get crd | grep external-secrets.io

# Check ESO operator
kubectl get deploy -n external-secrets
kubectl get pods -n external-secrets

# Check Vault is reachable
kubectl exec -it <vault-pod> -n vault -- sh
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="root"
vault status
vault list secret/
vault get secret/studentdb

# Port-forward Vault for quick test
kubectl port-forward -n vault svc/vault 8200:8200
curl http://localhost:8200/v1/secret/data/studentdb --header "X-Vault-Token: root"
```

### **ESO Resource Checks**
```sh
kubectl get secretstore -n external-secrets
kubectl describe secretstore vault-secretstore -n external-secrets
kubectl get externalsecret -n student-api
kubectl describe externalsecret postgres-secret -n student-api
kubectl get secret -n student-api postgres-secret -o yaml
echo <base64-string> | base64 -d
```

---

## 5. Kubernetes/ESO/Vault Checklist

- **CRDs Installed:**  
  `kubectl get crd | grep external-secrets.io`  
  Should list all ESO resources.

- **ESO Operator Running:**  
  `kubectl get deploy -n external-secrets`  
  Pod status must be Running.

- **Vault Reachable:**  
  `kubectl get svc -n vault`  
  Port-forward and curl health endpoint.

- **Vault Token Secret Exists:**  
  `kubectl get secret vault-token -n external-secrets -o yaml`  
  Must contain the root token.

- **ClusterSecretStore Configured:**  
  `kubectl get secretstore -n external-secrets`  
  Status must be Ready, path correct.

- **ExternalSecret Status:**  
  `kubectl get externalsecret postgres-secret -n student-api -o yaml`  
  Status should show Synced and Ready.

- **Secret Created in App Namespace:**  
  `kubectl get secret postgres-secret -n student-api -o yaml`  
  Must contain POSTGRES_USER and POSTGRES_PASSWORD.

---

## Importance of Each Step

- **Secure secret management** via Vault, never exposing credentials.
- **Automated secret sync** with ESO, ensuring latest credentials.
- **Declarative infra** for reproducible production setups.
- **Isolation** via namespaces, node selectors, and service accounts.
- **Health checks & logging** for fast debugging and troubleshooting.

---

## References
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [External Secrets Operator](https://external-secrets.io/latest/)
- [HashiCorp Vault](https://developer.hashicorp.com/vault/docs)
- [Flask-Migrate](https://flask-migrate.readthedocs.io/en/latest/)
