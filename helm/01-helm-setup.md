## 1. Vault (Helm) — install, validate, keep safe

### Why Vault first

- ESO and other components depend on Vault being reachable and containing secrets. Vault must exist and be reachable before ESO is installed.

### Install (recommended via HashiCorp Helm chart)

**Add repo and install:**

```sh
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```
```sh
# install (dev config shown — adapt for production)
cat > vault-values.yaml <<'EOF'
server:
  standalone:
    enabled: true
    config: |
      ui = true
      listener "tcp" {
        address     = "0.0.0.0:8200"
        tls_disable = 1
      }
  ha:
    enabled: false
ui:
  enabled: true
EOF
```
```sh
helm upgrade --install vault hashicorp/vault -n vault --create-namespace -f vault-values.yaml
```
*(If you already have a working Vault pod in dev mode like your cluster does, keep it — do not delete it unless you exported secrets and know what you’re doing.)*

### Verify Vault
```sh
kubectl get pods -n vault
kubectl logs -n vault -l app.kubernetes.io/name=vault

# exec into vault pod and check status (replace pod name if different)
VAULT_POD=$(kubectl get pod -n vault -l app.kubernetes.io/name=vault -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n vault -it $VAULT_POD -- /bin/sh -c "export VAULT_ADDR='http://127.0.0.1:8200'; export VAULT_TOKEN='root'; vault status"
```

- Look for:
  - Pod Running
  - vault status showing Sealed: false
  - Vault UI reachable (optional) via port-forward: kubectl port-forward -n vault $VAULT_POD 8200:8200

### Store a secret into Vault (example)

#### From inside vault pod (or from CLI with VAULT_ADDR/VAULT_TOKEN):

```sh
kubectl exec -n vault -it $VAULT_POD -- /bin/sh -c "
export VAULT_ADDR='http://127.0.0.1:8200';
export VAULT_TOKEN='root';
vault kv put secret/studentdb POSTGRES_USER=postgres POSTGRES_PASSWORD=postgres123
vault kv get -format=json secret/studentdb
"
```

#### Create a Vault policy and token for ESO

Inside vault pod:
```sh
cat > /tmp/eso-policy.hcl <<'EOF'
path "secret/data/studentdb" {
  capabilities = ["read"]
}
EOF

kubectl exec -n vault -it $VAULT_POD -- /bin/sh -c "
export VAULT_ADDR='http://127.0.0.1:8200';
export VAULT_TOKEN='root';
vault policy write eso-policy /tmp/eso-policy.hcl;
vault token create -policy=eso-policy -field=token > /tmp/eso-token
cat /tmp/eso-token
"
```
- Copy the token (or create a limited token manually) for ESO to use.
- Important: In production use AppRole or Kubernetes auth rather than long-lived root tokens.
### Do not delete Vault
- Keep Vault running; it is the source of truth. If you must recreate Vault, export secrets before deleting.

## 2. External-Secrets Operator (ESO) — Helm install + connect to Vault

### What it must do

- Read secrets from Vault (via a SecretStore/ClusterSecretStore) and create Kubernetes Secrets.
- Must have RBAC and a k8s Secret with Vault token (or use AppRole/k8s auth).

#### Key design decisions we used
- Use ClusterSecretStore (cluster-scoped) pointing to Vault.
- ESO will create a Kubernetes Secret postgres-secret in student-api.
- Disable unused controllers (e.g., pushsecret) to avoid RBAC errors.

#### values.yaml (example for your chart)

Use or adapt your helm/external-secrets/values.yaml — make sure it contains the args to disable pushsecret:
```yaml
namespaces:
  eso: external-secrets
  app: student-api

replicaCount: 1

image:
  repository: ghcr.io/external-secrets/external-secrets
  tag: v0.9.9
  pullPolicy: IfNotPresent

serviceAccount:
  name: external-secrets

vault:
  secretName: vault-token
  token: "root"        # replace with token created for ESO
  server: "http://vault.vault.svc.cluster.local:8200"
  path: "secret"
  version: "v2"

database:
  secretName: postgres-secret
  vaultKey: studentdb
  usernameKey: POSTGRES_USER
  passwordKey: POSTGRES_PASSWORD

nodeSelector:
  type: dependent_services

# Important: disable controllers you don't use, e.g. pushsecret
args:
  - --disable-controllers=pushsecret
```

#### Pre-create the vault token k8s Secret (in external-secrets ns)

- (Use the limited token, not root in prod.)

```sh
kubectl create namespace external-secrets --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic vault-token --from-literal=token=<ESO_TOKEN_FROM_VAULT> -n external-secrets
```
#### Install ESO via Helm (chart in your repo or official repo)

- If you created a chart locally:

```sh
helm upgrade --install external-secrets ./helm/external-secrets -n external-secrets -f ./helm/external-secrets/values.yaml
```

- Or via official chart repo (if using the chart repo):
```sh
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm upgrade --install external-secrets external-secrets/external-secrets -n external-secrets -f ./helm/external-secrets/values.yaml
```

#### Verify ESO & ClusterSecretStore & ExternalSecret
```sh
kubectl get pods -n external-secrets
kubectl logs -n external-secrets -l app=external-secrets -f

kubectl get clustersecretstore -n external-secrets
kubectl describe clustersecretstore vault-secretstore -n external-secrets

kubectl get externalsecret -n student-api
kubectl describe externalsecret postgres-secret -n student-api
```

#### Expect:
- ESO pod Running
- clustersecretstore vault-secretstore status Valid/Ready
- externalsecret postgres-secret status SecretSynced (Ready=True)
- K8s Secret postgres-secret present in student-api:
```sh
kubectl get secret postgres-secret -n student-api -o yaml
kubectl get secret postgres-secret -n student-api -o jsonpath="{.data.POSTGRES_USER}" | base64 --decode
```

### Troubleshooting ESO — quick checklist & commands

#### Common failures & fixes

**1. CrashLoopBackOff / cache sync timeouts: usually RBAC. Give ESO SA proper cluster permissions or disable controllers you don’t use (--disable-controllers=pushsecret).**

  - Check logs:
  ```sh
  kubectl logs -n external-secrets -l app=external-secrets
  ```
  - If logs show forbidden for pushsecrets, either add RBAC for PushSecret resources or disable the controller.

**2. SecretSyncedError: ESO cannot access Vault or token invalid.**

  - Verify vault-token secret exists in external-secrets namespace:
  ```sh
  kubectl get secret vault-token -n external-secrets -o yaml
  ```
  - Test connectivity from a debug pod:
  ```sh
  kubectl run -n external-secrets -i --rm --restart=Never debug --image=curlimages/curl -- sh -c "curl -sS http://vault.vault.svc.cluster.local:8200/v1/sys/health"
  ```

**3. Missing secret in k8s:** 
  - Inspect ExternalSecret status and ESO logs.

## 3. Postgres (database) — Helm + use ESO secret

There are two common approaches depending on whether you want the Postgres Helm chart to initialize the DB password or to consume an externally-synced secret:

**Option A — Use a chart (e.g., Bitnami) that accepts an existing secret (recommended)**

- If the chart supports an existing secret option, you can have ESO create the secret first, then deploy the Postgres chart specifying the existing secret so Postgres uses it for initialization.

**values-postgres.yaml (example for bitnami with existingSecret):**
```yaml
global:
  postgresql:
    auth:
      existingSecret: postgres-secret   # name of secret created by ESO in student-api
      # or some charts use postgresql.postgresqlPassword; check chart docs
persistence:
  enabled: true
  size: 1Gi
```

- Install after ESO has created the secret:
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm upgrade --install student-db bitnami/postgresql -n student-api -f values-postgres.yaml
```
- If the chart doesn’t support existingSecret, see Option B.

**Option B — Deploy Postgres as a deployment/statefulset referencing the k8s secret**

- This is what your project used in practice: a PostgreSQL pod that reads POSTGRES_USER/POSTGRES_PASSWORD from postgres-secret (created by ESO).

- postgres-deployment snippet (template):
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: student-api
spec:
  selector:
    matchLabels:
      app: postgres
  serviceName: postgres
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
            - name: POSTGRES_DB
              value: studentdb
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

- Install via Helm chart (custom chart) or kubectl apply -f (but prefer Helm templating to keep ownership).

#### Verify Postgres
```sh
kubectl get pods -n student-api -l app=postgres
kubectl logs -n student-api -l app=postgres
kubectl exec -it -n student-api $(kubectl get pod -l app=postgres -o jsonpath='{.items[0].metadata.name}') -- psql -U $POSTGRES_USER -c '\l'
```
  - (You may need to kubectl exec and set env vars or use psql -U postgres -d postgres).

### Troubleshooting Postgres

**1. CreateContainerConfigError**
  - usually missing secret or missing PVC:
  *kubectl describe pod postgres-... -n student-api — check Events for secret "postgres-secret" not found or pod has unbound immediate PersistentVolumeClaims.*

**2. PVC Pending**
  - there’s no PV provisioner or storage class; fix storage class or create PV manually.

**3. DB init problems**
  - check logs: 
    ```sh
    kubectl logs <postgres-pod> -n student-api.
    ```

## 4. Flask application — Helm chart, init container, secrets usage

Your Flask app needs DB credentials from postgres-secret and waits for Postgres using an init container that runs migrations.

### Helm chart (values + deployment template)

#### values.yaml (flask chart):
```yaml
replicaCount: 2
image:
  repository: akhilthyadi/flask-app
  tag: 7.0.0
  pullPolicy: IfNotPresent

env:
  POSTGRES_HOST: postgres.student-api.svc.cluster.local
  POSTGRES_PORT: "5432"
  FLASK_APP: wsgi.py
```

#### deployment template snippet (key parts):
```yaml
initContainers:
  - name: db-migrations
    image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
    command:
      - sh
      - -c
      - |
        echo "Waiting for Postgres at $POSTGRES_HOST:$POSTGRES_PORT..."
        until pg_isready -h $POSTGRES_HOST -p $POSTGRES_PORT -U $POSTGRES_USER; do sleep 2; done
        echo "Postgres is ready, running migrations..."
        flask db upgrade
    env:
      - name: POSTGRES_USER
        valueFrom:
          secretKeyRef:
            name: {{ .Values.database.secretName | default "postgres-secret" }}
            key: POSTGRES_USER
      - name: POSTGRES_PASSWORD
        valueFrom:
          secretKeyRef:
            name: {{ .Values.database.secretName | default "postgres-secret" }}
            key: POSTGRES_PASSWORD
      - name: POSTGRES_HOST
        value: {{ .Values.env.POSTGRES_HOST | quote }}
      - name: POSTGRES_PORT
        value: {{ .Values.env.POSTGRES_PORT | quote }}

containers:
  - name: flask-api
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    ports:
      - containerPort: 5000
    env:
      - name: POSTGRES_USER
        valueFrom:
          secretKeyRef:
            name: {{ .Values.database.secretName | default "postgres-secret" }}
            key: POSTGRES_USER
      - name: POSTGRES_PASSWORD
        valueFrom:
          secretKeyRef:
            name: {{ .Values.database.secretName | default "postgres-secret" }}
            key: POSTGRES_PASSWORD
      - name: POSTGRES_HOST
        value: {{ .Values.env.POSTGRES_HOST | quote }}
      - name: POSTGRES_PORT
        value: {{ .Values.env.POSTGRES_PORT | quote }}
```

#### Install Flask via Helm
```sh
helm upgrade --install flask-api ./helm/flask-api -n student-api -f ./helm/flask-api/values.yaml
```
#### Validate Flask
```sh
kubectl get pods -n student-api -l app=flask-api
kubectl logs -n student-api -l app=flask-api
# port-forward and test
kubectl port-forward svc/flask-api-service 5000:80 -n student-api &
curl -sS http://127.0.0.1:5000/health
```
#### Look for:
- Init container db-migrations completes with Exit Code 0.
- Flask container Running and Ready.
- Flask logs show migrations success and app serving.

### Troubleshooting Flask

**1. Init:CreateContainerConfigError**

  - missing postgres-secret. 
  - Check kubectl describe pod event for "secret 'postgres-secret' not found".

**2. Init failing when migrations run**
  - look at init container logs:
  ```sh
  kubectl logs <flask-pod> -n student-api -c db-migrations
  ```
  - Confirm environment variables:
  ```sh
  kubectl exec -it <flask-pod> -n student-api -- env | grep POSTGRES
  ```

## 5. Order of operations (the canonical flow)
- Vault — install & ensure secret exists (e.g., secret/studentdb).
- Create Vault policy & token for ESO, create k8s Secret vault-token in external-secrets namespace.
- ESO (External-Secrets Operator) — install with Helm (make sure token, clustersecretstore config are in values or chart).
- Confirm ESO synced the secret → k8s Secret postgres-secret appears in student-api.
- Database — install Postgres chart referencing existing secret (or use custom chart that reads the secret).
- Flask app — install chart; init container should wait for DB and run migrations.
- Verify app responds.

## 6. Helpful commands & debugging checklist (copy-paste)

- **Install / upgrade**
  ```sh
  # Vault
  helm upgrade --install vault hashicorp/vault -n vault -f vault-values.yaml
  
  # ESO
  helm upgrade --install external-secrets ./helm/external-secrets -n external-secrets -f ./helm/external-secrets/values.yaml

  # Postgres (bitnami example)
  helm repo add bitnami https://charts.bitnami.com/bitnami
  helm upgrade --install student-db bitnami/postgresql -n student-api -f ./helm/postgres/values.yaml

  # Flask
  helm upgrade --install flask-api ./helm/flask-api -n student-api -f ./helm/flask-api/values.yaml
  ```

- **Verify quickly**
  ```sh
  kubectl get pods -n vault
  kubectl get pods -n external-secrets
  kubectl get pods -n student-api

  kubectl get clustersecretstore -n external-secrets
  kubectl get externalsecret -n student-api
  kubectl get secret postgres-secret -n student-api -o yaml
  kubectl logs -n external-secrets -l app=external-secrets
  kubectl logs -n student-api -l app=postgres
  kubectl logs -n student-api -l app=flask-api
  ```

- **If something fails — first diagnostics**
  ```sh
  # Inspect failing pod
  kubectl describe pod <pod-name> -n <namespace>

  # Tail logs (operator / app)
  kubectl logs -n external-secrets -l app=external-secrets -f
  kubectl logs -n student-api -l app=flask-api -f

  # Check ExternalSecret and ClusterSecretStore
  kubectl describe externalsecret postgres-secret -n student-api
  kubectl describe clustersecretstore vault-secretstore -n external-secrets

  # Check k8s Secret created
  kubectl get secret postgres-secret -n student-api -o yaml
  ```

- **Clean up and reinstall (safe order)**
  ```sh
  # uninstall apps
  helm uninstall flask-api -n student-api
  helm uninstall student-db -n student-api
  helm uninstall external-secrets -n external-secrets

  # remove leftover cluster-level resources if you want a full clean (be careful)
  kubectl delete clustersecretstore vault-secretstore
  kubectl delete externalsecret postgres-secret -n student-api
  kubectl delete secret vault-token -n external-secrets
  kubectl delete clusterrole external-secrets-cluster-role
  kubectl delete clusterrolebinding external-secrets-cluster-role-binding
  ```

## 7. Final notes — gotchas & best practices

- Do not delete Vault unless you have exported secrets. Vault is the source of truth.
- Prefer least-privilege: create a Vault policy that only allows ESO to read required keys; create a token restricted to that policy and store token as a k8s Secret.
- Use existingSecret or custom init for Postgres to avoid mismatched passwords at init time.
- Use PVCs for Postgres and Vault if persistence matters.
- Helm ownership: if resources were created previously (manually or by old release), Helm install can fail — delete the old resource or set serviceAccount.create=false etc. We handled those exact conflicts earlier.
- Disable unused ESO controllers (pushsecret etc.) to avoid RBAC friction, or provide full RBAC for ESO service account.
