# Day 15 — Secrets Management at Scale

*The #1 security gap in most K8s clusters: secrets. Teams either commit them to Git (catastrophic), store them as plain K8s Secrets (base64 = readable by anyone with etcd access), or manually kubectl apply them (no audit trail, no rotation). Today you learn how production teams actually handle secrets.*

## 🧠 Part 1: The Secrets Problem

```
Bad:     Secret in Git repo           → anyone with repo access has it
Bad:     kubectl create secret manual → no audit trail, no rotation
Bad:     K8s Secret unencrypted       → base64 in etcd, readable by etcd admins
Better:  K8s Secret + etcd encryption → encrypted at rest, still in cluster
Best:    External secret manager      → single source of truth, full audit,
                                        automatic rotation, fine-grained access
```

*Three production patterns ranked by maturity:*

```
Level 1: Sealed Secrets      — encrypt in Git, decrypt in cluster
Level 2: External Secrets    — K8s pulls from AWS SM / GCP SM / Azure KV
Level 3: Vault Agent         — Vault injects secrets directly into pods
```

*All three solve different problems. Know when to use each.*

## 🔐 Part 2: Sealed Secrets — Encrypt Secrets for Git

*Sealed Secrets lets you commit encrypted secrets to Git safely. Only the controller in your cluster can decrypt them.*

```
You encrypt with cluster's public key  →  SealedSecret YAML (safe to commit)
Controller decrypts with private key   →  regular K8s Secret (in cluster only)
```

**Install Sealed Secrets**

```
# Use the official installation manifest
kubectl apply -f https://github.com/bitnami/sealed-secrets/releases/latest/download/controller.yaml

# Verify the installation
kubectl get pods -n kube-system | grep sealed

# Install kubeseal CLI
curl -L https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/kubeseal-0.26.0-linux-amd64.tar.gz \
  | tar -xz kubeseal
sudo mv kubeseal /usr/local/bin/

# Verify kubeseal
kubeseal --version
```

**Create and use Sealed Secrets**

```
# Step 1: Create a regular secret (never apply this directly)
kubectl create secret generic db-credentials \
  --from-literal=DB_HOST=postgres.prod.internal \
  --from-literal=DB_PASSWORD=supersecret123 \
  --from-literal=DB_USER=app_user \
  --dry-run=client \
  -o yaml > /tmp/db-secret.yaml

# Step 2: Seal it using the cluster's public key
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  < /tmp/db-secret.yaml \
  > sealed-db-secret.yaml

# Step 3: Inspect the sealed secret — safe to commit to Git
cat sealed-db-secret.yaml
# apiVersion: bitnami.com/v1alpha1
# kind: SealedSecret
# metadata:
#   name: db-credentials
#   namespace: default
# spec:
#   encryptedData:
#     DB_HOST: AgB3j7K...  ← encrypted, not base64
#     DB_PASSWORD: AgCp9X...
#     DB_USER: AgDm2R...

# Step 4: Apply — controller decrypts and creates real Secret
kubectl apply -f sealed-db-secret.yaml

# Verify the real Secret was created
kubectl get secret db-credentials
kubectl get secret db-credentials \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
# supersecret123  ← decrypted by controller

# Original SealedSecret can be safely committed to git
git add sealed-db-secret.yaml
git commit -m "add db credentials (sealed)"
```

**Namespace scoping — critical security detail**

```
# Sealed Secrets are scoped to namespace by default
# A secret sealed for "production" cannot be decrypted in "staging"

# Seal for specific namespace
kubeseal \
  --namespace production \
  --scope namespace-wide \
  --format yaml \
  < /tmp/db-secret.yaml \
  > sealed-db-production.yaml

# Seal for any namespace (less secure — use rarely)
kubeseal \
  --scope cluster-wide \
  --format yaml \
  < /tmp/db-secret.yaml \
  > sealed-db-cluster.yaml

# Seal a single key
kubectl create secret generic partial \
  --from-literal=API_KEY=abc123 \
  --dry-run=client -o yaml \
  | kubeseal \
    --namespace production \
    --name my-secret \
    -o yaml \
    --merge-into existing-sealed-secret.yaml
```

**Backup the sealing key — CRITICAL**

```
# If the controller loses its private key, ALL sealed secrets are unrecoverable
# Backup the sealing key to a secure location

kubectl get secret \
  -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master-key-backup.yaml

# Store this in: password manager, secure S3 bucket, HSM
# NEVER commit to Git
# Test restore quarterly
```

## ☁️ Part 3: External Secrets Operator

*ESO pulls secrets from external providers (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault, Azure Key Vault) and creates native K8s Secrets automatically. The secret values never live in Git — only the reference does.*

```
Git has:      ExternalSecret (which secret to fetch, no values)
ESO fetches:  Actual value from AWS SM / Vault at sync interval
ESO creates:  Regular K8s Secret (with real values, in cluster only)
```

**Install ESO**

```
helm repo add external-secrets \
  https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true

kubectl get pods -n external-secrets -w
# external-secrets (controller)
# external-secrets-cert-controller
# external-secrets-webhook
```

**Connect to AWS Secrets Manager**

```
# Step 1: Create AWS credentials secret (or use IRSA — preferred on EKS)
kubectl create secret generic aws-credentials \
  --from-literal=access-key=AKIAIOSFODNN7EXAMPLE \
  --from-literal=secret-access-key=wJalrXUtnFEMI/K7MDENG \
  -n production

# Step 2: Create a SecretStore — defines the external backend
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key
EOF

# Step 3: Create an ExternalSecret — what to fetch
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: url-shortener-secrets
  namespace: production
spec:
  refreshInterval: 1h           # resync from AWS SM every hour
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore

  target:
    name: url-shortener-secrets  # K8s Secret name to create
    creationPolicy: Owner         # ESO owns and manages the Secret
    deletionPolicy: Retain        # keep Secret if ExternalSecret deleted

  data:
  # Fetch individual keys from AWS SM
  - secretKey: DB_PASSWORD        # key in K8s Secret
    remoteRef:
      key: prod/url-shortener/database    # AWS SM secret name
      property: password                   # JSON key inside the secret

  - secretKey: REDIS_PASSWORD
    remoteRef:
      key: prod/url-shortener/redis
      property: password

  # Fetch entire secret and extract all keys
  dataFrom:
  - extract:
      key: prod/url-shortener/app-config  # fetches all key/values
EOF

# Check ESO created the K8s Secret
kubectl get externalsecret -n production
# NAME                      STORE                REFRESH INTERVAL   STATUS
# url-shortener-secrets     aws-secretsmanager   1h                 SecretSynced

kubectl get secret url-shortener-secrets -n production
# Real K8s Secret created automatically

# Check sync status and errors
kubectl describe externalsecret url-shortener-secrets -n production

# Check secretstore
kubectl get secretstore -n production

# Check externalsecret
kubectl get externalsecret -n production

# Describe the ExternalSecret
kubectl describe externalsecret url-shortener-secrets -n production

# Check the SecretStore status
kubectl describe secretstore aws-secretsmanager -n production

# Check the ESO controller logs
kubectl logs -n external-secrets deploy/external-secrets
```

**ClusterSecretStore — shared across namespaces**

```
# One store definition for all namespaces
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore      # cluster-scoped (vs SecretStore which is namespaced)
metadata:
  name: aws-global
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-west-1
      auth:
        jwt:                  # use IRSA on EKS — no static credentials
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

**ESO with HashiCorp Vault**

```
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "http://vault.vault-system.svc:8200"
      path: "secret"          # KV mount path
      version: "v2"           # KV v2
      auth:
        kubernetes:           # Vault Kubernetes auth method
          mountPath: kubernetes
          role: url-shortener-role  # Vault role that maps to K8s SA
          serviceAccountRef:
            name: url-shortener-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secrets
  namespace: production
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app-secrets
  data:
  - secretKey: DB_PASSWORD
    remoteRef:
      key: secret/prod/database    # Vault path
      property: password
```

## 🏦 Part 4: HashiCorp Vault — The Enterprise Standard

*Vault is the most powerful secrets manager. Features that matter for K8s:*

```
Dynamic secrets     — Vault generates DB credentials on demand, auto-rotates
Lease system        — secrets expire automatically, apps must renew
Audit log           — every secret access logged with identity
Multiple auth       — K8s SA, AWS IAM, OIDC, AppRole
Transit encryption  — encrypt/decrypt without storing the plaintext
```

**Deploy Vault on K8s**

```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  --namespace vault-system \
  --create-namespace \
  --set server.dev.enabled=true   # dev mode for learning — not production

kubectl get pods -n vault-system -w

# Initialize Vault (production — dev mode auto-initializes)
# kubectl exec -n vault-system vault-0 -- vault operator init

# Unseal (production — dev mode auto-unseals)
# kubectl exec -n vault-system vault-0 -- vault operator unseal <key>

# Port-forward to access Vault UI
kubectl port-forward -n vault-system svc/vault 8200:8200
# http://localhost:8200
# Token: root (dev mode only)
```

**Configure Vault for K8s auth**

```
# Exec into vault pod
kubectl exec -it -n vault-system vault-0 -- sh

# Enable KV secrets engine
vault secrets enable -path=secret kv-v2

# Write some secrets
vault kv put secret/prod/database \
  username=app_user \
  password=db_secret_password_123

vault kv put secret/prod/redis \
  password=redis_secret_456

# Enable Kubernetes auth method
vault auth enable kubernetes

# Configure it to trust your cluster
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token

# Create a policy defining what can be read
vault policy write url-shortener-policy - <<EOF
path "secret/data/prod/*" {
  capabilities = ["read", "list"]
}
path "secret/data/prod/database" {
  capabilities = ["read"]
}
EOF

# Create a role that maps K8s SA to the policy
vault write auth/kubernetes/role/url-shortener-role \
  bound_service_account_names=url-shortener-sa \
  bound_service_account_namespaces=production \
  policies=url-shortener-policy \
  ttl=1h

exit
```

**Vault Agent Injector — inject secrets as files**

```
# The Injector is a MutatingAdmissionWebhook
# Annotate your pod — Vault Agent sidecar is injected automatically

apiVersion: apps/v1
kind: Deployment
metadata:
  name: url-shortener
  namespace: production
spec:
  template:
    metadata:
      annotations:
        # These annotations trigger the Vault Agent injection
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "url-shortener-role"

        # Inject database password to /vault/secrets/database.txt
        vault.hashicorp.com/agent-inject-secret-database.txt: "secret/data/prod/database"

        # Template the output format
        vault.hashicorp.com/agent-inject-template-database.txt: |
          {{- with secret "secret/data/prod/database" -}}
          DB_USER={{ .Data.data.username }}
          DB_PASS={{ .Data.data.password }}
          {{- end }}

        # Inject redis password
        vault.hashicorp.com/agent-inject-secret-redis.txt: "secret/data/prod/redis"

        # Agent renews tokens automatically
        vault.hashicorp.com/agent-pre-populate-only: "false"

    spec:
      serviceAccountName: url-shortener-sa  # must match Vault role
      containers:
      - name: fastapi
        image: ghcr.io/youruser/url-shortener:v2
        # Read secrets from files — never env vars
        command: ["sh", "-c"]
        args:
        - |
          source /vault/secrets/database.txt
          exec python main.py
```

**Dynamic database credentials — the killer feature**

```
# Inside Vault pod:
kubectl exec -it -n vault-system vault-0 -- sh

# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL
vault write database/config/postgres \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@postgres.production.svc:5432/appdb?sslmode=disable" \
  allowed_roles="url-shortener-role" \
  username="vault_admin" \
  password="vault_admin_password"

# Create a role — Vault generates credentials on demand
vault write database/roles/url-shortener-role \
  db_name=postgres \
  creation_statements="
    CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";
  " \
  default_ttl="1h" \    # credentials expire in 1 hour
  max_ttl="24h"

# Test: generate credentials on demand
vault read database/creds/url-shortener-role
# Key            Value
# lease_id       database/creds/url-shortener-role/xxx
# username       v-k8s-url-sho-xxx    ← unique per request
# password       A1B2C3D4E5F6G7H8     ← auto-generated

# Each pod gets unique credentials — no shared passwords
# When TTL expires, Vault can auto-revoke them from Postgres

exit
```

## 🔄 Part 5: Secret Rotation

*Rotation is what separates mature security from checkbox security.*

```
# Rotation workflow with ESO:
# 1. Rotate secret in AWS SM (or Vault)
# 2. ESO detects change at next refresh interval
# 3. K8s Secret updated automatically
# 4. Pod must reload — either via volume mount (auto) or restart

# Force immediate resync
kubectl annotate externalsecret url-shortener-secrets \
  -n production \
  force-sync=$(date +%s) \
  --overwrite

# For env vars — must restart pods to pick up new Secret values
kubectl rollout restart deployment/url-shortener -n production

# For volume-mounted secrets — automatic within 60s (no restart needed)
# Use projected volume with secret for auto-rotation:
```

```
spec:
  volumes:
  - name: secrets
    projected:
      sources:
      - secret:
          name: url-shortener-secrets   # ESO keeps this updated
          items:
          - key: DB_PASSWORD
            path: db_password

  containers:
  - name: app
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
    # App reads /etc/secrets/db_password on each use
    # When ESO rotates the Secret, file updates within 60s
    # No pod restart needed
```

## 🖥️ Part 6: Hands-On Exercises

**Exercise 1: Full Sealed Secrets workflow**

```
# Install (from Part 2)
# Create a secret for your URL shortener

kubectl create namespace production 2>/dev/null || true

kubectl create secret generic url-shortener-secrets \
  --from-literal=REDIS_PASSWORD=redis_prod_pass \
  --from-literal=DB_PASSWORD=postgres_prod_pass \
  --from-literal=API_SECRET_KEY=jwt_signing_key_abc123 \
  --namespace production \
  --dry-run=client -o yaml > /tmp/app-secrets.yaml

# Seal it
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --namespace production \
  --format yaml \
  < /tmp/app-secrets.yaml \
  > sealed-app-secrets.yaml

# Inspect — confirm values are encrypted not base64
cat sealed-app-secrets.yaml | grep -A 5 encryptedData

# Apply and verify
kubectl apply -f sealed-app-secrets.yaml

kubectl get sealedsecret -n production
kubectl get secret url-shortener-secrets -n production
kubectl get secret url-shortener-secrets -n production \
  -o jsonpath='{.data.REDIS_PASSWORD}' | base64 -d
# redis_prod_pass

# Prove namespace scoping — try applying to different namespace
kubectl create namespace staging 2>/dev/null || true
sed 's/namespace: production/namespace: staging/' \
  sealed-app-secrets.yaml \
  | kubectl apply -f -

kubectl get secret -n staging   # empty — controller rejected it
kubectl describe sealedsecret -n staging url-shortener-secrets
# Error: no key could decrypt the message
```

**Exercise 2: ESO with a local fake provider**

```
# Install ESO
helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true

kubectl get pods -n external-secrets -w

# Use the Fake provider for learning (no AWS needed)
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: fake-store
spec:
  provider:
    fake:
      data:
      - key: "/prod/database/password"
        value: "fake_db_password_from_store"
      - key: "/prod/redis/password"
        value: "fake_redis_password_from_store"
      - key: "/prod/app/api_key"
        value: "fake_api_key_abc123"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: default
spec:
  refreshInterval: 30s
  secretStoreRef:
    name: fake-store
    kind: ClusterSecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
  - secretKey: DB_PASSWORD
    remoteRef:
      key: /prod/database/password
  - secretKey: REDIS_PASSWORD
    remoteRef:
      key: /prod/redis/password
  - secretKey: API_KEY
    remoteRef:
      key: /prod/app/api_key
EOF

# Watch ESO create the real secret
kubectl get externalsecret app-secrets -w
# STATUS: SecretSynced

# Verify real Secret created
kubectl get secret app-secrets -o yaml

# Decode values
kubectl get secret app-secrets \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
# fake_db_password_from_store

# Update the fake provider value — simulate rotation
kubectl patch clustersecretstore fake-store --type=merge -p '
{
  "spec": {
    "provider": {
      "fake": {
        "data": [
          {"key": "/prod/database/password", "value": "ROTATED_DB_PASSWORD_v2"},
          {"key": "/prod/redis/password", "value": "fake_redis_password_from_store"},
          {"key": "/prod/app/api_key", "value": "fake_api_key_abc123"}
        ]
      }
    }
  }
}'

# Wait 30s (refreshInterval) — Secret auto-updates
sleep 35
kubectl get secret app-secrets \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
# ROTATED_DB_PASSWORD_v2  ← auto-rotated
```

**Exercise 3: Vault dynamic secrets**

```
# Install Vault in dev mode
helm install vault hashicorp/vault \
  --namespace vault-system \
  --create-namespace \
  --set server.dev.enabled=true \
  --set injector.enabled=true

kubectl get pods -n vault-system -w

# Configure Vault (follow Part 4 steps)
kubectl exec -it -n vault-system vault-0 -- sh

vault secrets enable -path=secret kv-v2

vault kv put secret/prod/app \
  DB_PASSWORD=vault_managed_password \
  API_KEY=vault_api_key_xyz \
  REDIS_PASSWORD=vault_redis_pass

vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token

vault policy write app-policy - <<'POLICY'
path "secret/data/prod/*" {
  capabilities = ["read"]
}
POLICY

vault write auth/kubernetes/role/app-role \
  bound_service_account_names=default \
  bound_service_account_namespaces=default \
  policies=app-policy \
  ttl=1h

exit

# Port-forward and verify
kubectl port-forward -n vault-system svc/vault 8200:8200 &
curl http://localhost:8200/v1/sys/health | jq .status
```

**Exercise 4: Audit which secrets exist and who can access them**

```
# Find all secrets across the cluster
kubectl get secrets -A \
  --no-headers \
  -o custom-columns=\
'NAMESPACE:.metadata.namespace,NAME:.metadata.name,TYPE:.type,AGE:.metadata.creationTimestamp' \
  | sort | column -t

# Find secrets of type Opaque (most sensitive)
kubectl get secrets -A \
  --field-selector=type=Opaque \
  -o custom-columns=\
'NAMESPACE:.metadata.namespace,NAME:.metadata.name'

# Who can read secrets in production?
kubectl auth can-i get secrets \
  --as=system:serviceaccount:production:default \
  -n production

# List all SAs that have secret access in production
for sa in $(kubectl get sa -n production -o jsonpath='{.items[*].metadata.name}'); do
  result=$(kubectl auth can-i get secrets \
    --as=system:serviceaccount:production:$sa \
    -n production 2>/dev/null)
  echo "SA: $sa → get secrets: $result"
done

# Find secrets not managed by ESO or Sealed Secrets
# (these might be manually created — security risk)
kubectl get secrets -A -o json \
  | jq -r '.items[] |
    select(
      .metadata.ownerReferences == null and
      .type == "Opaque"
    ) |
    "\(.metadata.namespace)/\(.metadata.name)"'
```

## 🎯 Part 7: Interview Questions — Day 18

**Q1: Sealed Secrets vs External Secrets Operator — when do you use each?**

Sealed Secrets when you want secrets in Git (GitOps-native) with no external dependencies — everything lives in the cluster, simpler to operate, works air-gapped. Downside: if the sealing key is lost, all secrets are unrecoverable. External Secrets when you already have a secrets manager (AWS SM, Vault) or need secret rotation, audit trails, dynamic credentials, or cross-cluster secret sharing. ESO is more operationally complex but far more powerful. Most mature teams use ESO + Vault and never put secrets in Git at all.

**Q2: A developer accidentally committed a secret to Git. What do you do?**

Act immediately — assume the secret is compromised the moment it hits a remote repo, even if you delete it within seconds (forks, CI logs, search indexes may have cached it). Rotate the credential immediately in the source system — don't just delete from Git. Use git filter-branch or BFG Repo Cleaner to scrub history, then force-push. Notify the security team. Post-incident: add pre-commit hooks (git-secrets, trufflehog) to catch secrets before push. Add secret scanning in CI (GitHub Advanced Security, Trivy --scanners secret). Migrate to ESO so secrets never need to be in Git.

**Q3: What are dynamic secrets and why are they better than static ones?**

Dynamic secrets are generated on-demand by Vault with a short TTL and automatically revoked when they expire. Each application instance gets unique credentials — if one pod is compromised, the attacker gets credentials that expire in an hour and only work for that pod's access patterns. Static secrets are shared, long-lived, and hard to rotate without downtime. With dynamic DB credentials, Vault creates a unique Postgres user per request, grants minimal permissions, and drops it after TTL. This makes lateral movement after a breach dramatically harder.

**Q4: How does Vault Kubernetes auth work? Walk through the token validation flow.**

The pod has a projected ServiceAccount token mounted at /var/run/secrets/kubernetes.io/serviceaccount/token. When the Vault Agent asks to authenticate, it sends this JWT to Vault. Vault calls the K8s TokenReview API (using its own SA) to validate the token — is it real, is it from the expected SA, is it for the expected namespace? If valid, Vault checks its role mapping — does this SA have a role? If yes, it returns a Vault token with the role's policies attached. The Vault Agent uses this token to fetch secrets. The JWT is short-lived (audience-bound), and the Vault token inherits the TTL from the role.

**Q5: How do you handle secret rotation with zero downtime?**

Volume-mounted secrets update automatically within 60s when the K8s Secret changes — no pod restart needed if the app reads the file on each use (not once at startup). For env vars, a rolling restart is needed. The pattern: ESO rotates the K8s Secret when AWS SM updates → apps using volume mounts get the new value automatically → apps using env vars get a triggered rolling restart via a Deployment annotation checksum (like the ConfigMap checksum pattern from Day 6). For DB credentials, Vault's dynamic secrets with overlapping TTLs handle rotation transparently — old credentials stay valid while new ones are issued.

**Q6: Someone deleted the Sealed Secrets controller and its key. What happens?**

All existing K8s Secrets that were created from SealedSecrets still exist and still work — the controller only runs at reconciliation time. But you cannot create new SealedSecrets or update existing ones without the private key, because you can't decrypt or re-encrypt. If you have the key backup (which you should — from Day 17's backup step), restore it as a K8s Secret in kube-system with the correct labels and reinstall the controller. If you have no backup, you must re-seal every secret from the original plaintext values, which means you need access to all original secret values — this is a serious incident. This is why key backup is non-negotiable.












