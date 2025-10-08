## Troubleshooting Playbook

### 1. Is the manifest applied and namespace present?

- Why: If resources aren’t created in the right namespace nothing else will work.

#### Command
```sh
kubectl apply -f application.yaml
kubectl get ns
kubectl get all -n student-api
```
#### What to look for
- kubectl apply reports configured/created.
- kubectl get all -n student-api shows your pods, svc, deployments, etc.

#### Common problems & fixes
- Resource missing: re-apply kubectl apply -f application.yaml.
- Wrong namespace in manifest: update metadata.namespace or use -n when applying.

---
### 2. Are pods coming up? What’s their status?

- Why: Pod status is the immediate indicator of whether containers started, are initializing, or failing.

#### Commands
```sh
kubectl get pods -n student-api -o wide
kubectl describe pod <pod-name> -n student-api
kubectl get events -n student-api --sort-by='.lastTimestamp'
```

#### What to look for
- STATUS values:
  - Running → good.
  - Init:Error, Init:0/1, PodInitializing → init container issue.
  - CrashLoopBackOff, Error → main container crashed.
  - ImagePullBackOff → image pulling issue.
  - kubectl describe pod → check Events at bottom (image pull errors, OOMKilled, permission denied, etc.)

#### Common problems & fixes
- Init stuck: check init container logs (next step).
- ImagePullBackOff → check image name/tag, registry auth, imagePullSecrets.
- OOMKilled → increase memory requests/limits.

---
#### 3. Are init container migrations running and finishing?

- Why: Init containers run before the main app — migrations failing prevents the main container from starting.

#### Commands
```sh
kubectl logs <pod> -n student-api -c db-migrations
kubectl logs <pod> -n student-api -c db-migrations --previous  # if restarted
```

#### What to look for
- Expected: Waiting for Postgres at ..., Postgres is ready, running migrations..., alembic INFO lines.
- Failure patterns:
  - DNS/connectivity: could not translate host name "postgres" to address
  - Authentication: password authentication failed
  - App import errors: Error: Could not import 'wsgi' or Flask errors
  - Malformed DB URI: ValueError: invalid literal for int() with base 10: 'tcp:'

#### Fixes
- DNS error → ensure Service name matches host env var or create alias Service named postgres.
- Auth error → ensure secret exists and env var maps to correct secret key; consider PGPASSWORD (or use ~/.pgpass).
- Import error → check FLASK_APP env or workingDir; ensure entrypoint & working dir correct in image.
- Malformed URI → check env var names and values (see step 6).
---

### 4. Are the main container logs healthy?

- Why: After init success main app should start — logs show app startup or runtime errors.

#### Command
```sh
kubectl logs <pod> -n student-api -c flask-api
```

#### What to look for
- “Student Management API started successfully.” or similar ready message
- Stack traces → inspect the traceback for root cause (missing env, DB errors, import errors).

#### Fixes
- Missing environment variables → check printenv inside pod.
- Exceptions referencing DB → check DB connection config and Postgres readiness.
---
### 5. Are environment variables set inside the container and do they match config.py?
- Why: Mismatched variable names are a frequent root cause (we saw DB_HOST vs POSTGRES_HOST, DB_USER vs POSTGRES_USER).

#### Commands
```sh
kubectl exec -n student-api -it <pod> -c db-migrations -- printenv | grep -i POSTGRES
kubectl exec -n student-api -it <pod> -c flask-api -- printenv | grep -i POSTGRES
```

#### What to look for
- Variables your app expects must be present with correct names: POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_HOST, POSTGRES_PORT, POSTGRES_DB.
- No tcp: prefixes. PORT must be numeric string (“5432”).

#### Fixes
- If env names differ from config.py: update Deployment env names to match config.py (or change config.py).
- Use kubectl set env deployment/flask-api POSTGRES_HOST=... for quick patching.
---
### 6. Is DB URI malformed? (ValueError: 'tcp:')
- Why: If the URI has an unexpected token (like tcp:) SQLAlchemy fails to parse the port.

#### Typical cause
- config.py forms postgresql://{user}:{pass}@{host}:{port}/{db} but host/port variables include tcp: (e.g. tcp:postgres).

#### How to confirm
- Check printenv inside pod and inspect POSTGRES_HOST & POSTGRES_PORT.
- Search pod logs for ValueError: invalid literal for int() with base 10: 'tcp:'.

#### Fix
- Remove tcp: from env value or fix variable mapping. Ensure:
```txt
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
```
---
### 7. Is DNS resolution working inside the pod?

- Why: Kubernetes cluster DNS must resolve service names to cluster IPs.

#### Commands
```sh
kubectl exec -n student-api -it <pod> -c db-migrations -- nslookup postgres
kubectl exec -n student-api -it <pod> -c db-migrations -- ping -c 1 postgres || true
kubectl get svc -n student-api
kubectl get endpoints -n student-api postgres
```

#### What to look for
- nslookup should return ClusterIP for postgres.
- kubectl get endpoints shows pod IP(s) behind the service.

#### Fix
- If nslookup fails: Service name mismatch — create Service named postgres or update POSTGRES_HOST.
- If endpoints empty: Service selector wrong — ensure selector in Postgres Service matches Postgres pod labels.
---
### 8. Does the DB accept connections from the pod? (auth & readiness)

- Why: The init container waits for DB readiness; auth errors block migrations.

#### Commands
```sh
# (from a pod that has psql or a container with psql installed)
kubectl exec -n student-api -it <pod> -- bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -h $POSTGRES_HOST -U $POSTGRES_USER -d $POSTGRES_DB -c "\dt"'
# or use the Postgres pod directly:
kubectl exec -n student-api -it <postgres-pod> -- psql -U $POSTGRES_USER -d $POSTGRES_DB -c "\dt"
```

#### What to look for
- Connection success and list of tables; migrations should have created tables.
- If authentication fails: FATAL: password authentication failed for user ...

#### Fix
- Confirm secrets (postgres-secret) contain correct values.
- Ensure ExternalSecret created the secret (see step 10).
---
### 9. Is the secret created by ExternalSecret present and correct?

- Why: App credentials are stored in secrets pulled from Vault via ExternalSecret.

#### Commands
```sh
kubectl get externalsecret -n student-api
kubectl describe externalsecret postgres-secret -n student-api
kubectl get secret postgres-secret -n student-api -o yaml
kubectl describe secret postgres-secret -n student-api
kubectl get secret postgres-secret -n student-api -o jsonpath='{.data}'  # base64 values
kubectl get secret postgres-secret -n student-api -o jsonpath='{.data.POSTGRES_USER}' | base64 --decode
```
#### What to look for
- ExternalSecret shows no errors and target secret exists.
- Secret contains keys POSTGRES_USER, POSTGRES_PASSWORD.

#### Fix
- If ExternalSecret controller shows errors: check controller logs (kubectl -n <controller-namespace> logs ...) and RBAC/ClusterSecretStore connection.
- If secret missing: re-check remoteRef (path/key) and Vault access.
---
### 10. Check Service & endpoints — is the app exposed correctly?

- Why: Service misconfiguration prevents access (targetPort mismatch, selector mismatch).

#### Commands
```sh
kubectl get svc -n student-api
kubectl describe svc flask-api-service -n student-api
kubectl get endpoints -n student-api flask-api-service
```

#### What to look for
- targetPort should match the containerPort (5000).
- Endpoints should list pod IPs for the service.
- If NodePort used, check nodePort value.

#### Fix
- Fix selector labels in service/deployment to match app: flask-api.
- Adjust targetPort or container port to align.
---
### 11. Validate application endpoints (port-forward, health check)

- Why: Confirms the app is running and responding.

#### Commands
```sh
kubectl port-forward svc/flask-api-service -n student-api 5000:80
curl -v http://localhost:5000/health
# or direct node access if NodePort is allowed:
NODE_IP=$(kubectl get nodes -o wide | awk 'NR==2{print $6}')
NODE_PORT=$(kubectl get svc flask-api-service -n student-api -o jsonpath='{.spec.ports[0].nodePort}')
curl http://$NODE_IP:$NODE_PORT/health
```

#### What to look for
- HTTP 200 and healthy response body.
- If curl times out: check firewall, NodePort exposure, or service endpoints.
---
### 12. If you see PodInitializing after init logs show migrations done
- Why: Sometimes pod still shows PodInitializing briefly — it’s waiting for init to finish and main container to become ready.

#### Commands
```sh
kubectl get pods -n student-api -w
kubectl logs <pod> -n student-api -c flask-api
kubectl describe pod <pod> -n student-api
```

#### What to look for
- If main container fails to become ready: check readiness probes, liveness probes, or runtime errors in logs.
- If probes failing: inspect probe settings in Deployment.

#### Fix
- Adjust readiness probe timeout, fix app startup time or fix app runtime errors.
---

### 13. Read common error messages & their meaning (quick reference)

- could not translate host name "postgres" to address
→ DNS/service name mismatch. Fix by matching Service name or updating POSTGRES_HOST.

- password authentication failed for user
→ Wrong password in secret; check ExternalSecret creation and secret content.

- ValueError: invalid literal for int() with base 10: 'tcp:'
→ Malformed POSTGRES_HOST or POSTGRES_PORT (extra tcp:). Check env vars.

- Error: Could not import 'wsgi' / No such command 'db'
→ Wrong FLASK_APP, wrong workingDir, or missing packages in image.

- ImagePullBackOff
→ Bad image tag or registry auth.

- CrashLoopBackOff with OOMKilled
→ Out-of-memory. Increase resources.

---
### 14. Quick fixes you can run from the cluster
- Set env in Deployment (quick patch)
```sh
kubectl set env deployment/flask-api POSTGRES_HOST=postgres POSTGRES_PORT=5432 -n student-api
kubectl rollout restart deployment/flask-api -n student-api
```

- Patch ConfigMap and reapply
```sh
kubectl edit configmap flask-config -n student-api
# or replace file and then:
kubectl apply -f application.yaml
```

- Delete pod to force a restart
```sh
kubectl delete pod <pod> -n student-api
```

- Create Service alias named postgres if needed
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: student-api
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```
```sh
kubectl apply -f postgres-alias.yaml
```
---

### 15. Final checklist (ordered, quick)

- **kubectl apply -f application.yaml**
- **kubectl get pods -n student-api — note statuses**
- If pods not Running → **kubectl describe pod <pod> -n student-api** → check Events
- **kubectl logs <pod> -n student-api -c db-migrations** — migrations output
- **kubectl logs <pod> -n student-api -c flask-api** — app logs
- **kubectl exec <pod> -n student-api -- printenv | grep POSTGRES** — env verification
- **kubectl get svc -n student-api and kubectl get endpoints -n student-api** — service checks
- **kubectl port-forward svc/flask-api-service -n student-api 5000:80** → curl /health
- If DB auth or secret issues → **kubectl get secret postgres-secret -n student-api** and check ExternalSecret

---
### Extra debugging tips & good practices

- Add PGPASSWORD to initContainer env when using pg_isready (non-interactive auth).
- Use --previous with kubectl logs to see crash logs before restart.
- Use kubectl cp to copy a debug image into pod namespace if your image lacks tools: create a debug pod (kubectl run -it debug --image=bitnami/kubectl -- sh) and use it to nslookup/psql.
- Keep non-sensitive config in ConfigMaps and secrets in Secrets/Vault. Keep env names consistent with application config.
- Use kubectl rollout status deployment/flask-api -n student-api to watch deployment progress.
