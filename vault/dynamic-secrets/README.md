# Working with dynamic secrets


## Deploy Postgres Server

```bash
# 네임스페이스 생성 : postgres
kubectl create ns postgres

# Bitnami Chart 저장소 추가
helm repo add bitnami https://charts.bitnami.com/bitnami

# postgresql 설치
helm upgrade --install postgres bitnami/postgresql --namespace postgres --set auth.audit.logConnections=true

# 발급된 POSTGRES_PASSWORD 환경변수 설정
export POSTGRES_PASSWORD=$(kubectl get secret --namespace postgres postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

# POSTGRES_PASSWORD 확인
echo $POSTGRES_PASSWORD
```

### For OpenShift

```bash
helm upgrade --install postgres bitnami/postgresql --namespace postgres -f postgres/values.yaml

export POSTGRES_PASSWORD=$(kubectl get secret --namespace postgres postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

echo $POSTGRES_PASSWORD
```

## Configure Postgres backend in Vault

```bash
# Vault pod shell 접근
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh

# 시크릿 엔진 활성화 : database
vault secrets enable -path=demo-db database

# DB 플러그인(Plugin) 및 구성(Config)
vault write demo-db/config/demo-db \
  plugin_name=postgresql-database-plugin \
  allowed_roles="dev-postgres" \
  connection_url="postgresql://{{username}}:{{password}}@postgres-postgresql.postgres.svc.cluster.local:5432/postgres?sslmode=disable" \
  username="postgres" \
  password="<위 Deploy Postgres Server 스텝에서 확인한 POSTGRES_PASSWORD 값>"

# Role 생성
# 참고1:주기적으로 DB Credentials 정보가 변경되는 것을 확인하기 위해 TTL 60s 설정
# 참고2:데모시연 시 실제 K8s Secret 정보가 Sync되고 demo deployment 앱이 재기동 되는 것을 확인할 수 있음
vault write demo-db/roles/dev-postgres \
  db_name=demo-db \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
      GRANT ALL PRIVILEGES ON DATABASE postgres TO \"{{name}}\";" \
  backend=demo-db \
  name=dev-postgres \
  default_ttl="60s" \
  max_ttl="1h"

# 정책생성
vault policy write demo-auth-policy-db - <<EOT
path "demo-db/creds/dev-postgres" {
  capabilities = ["read"]
}
EOT
```

## Configure K8s Auth in Vault

```bash
# Enable Kubernetes auth backend
vault auth enable -path demo-auth-mount kubernetes

# Configure Kubernetes auth backend
vault write auth/demo-auth-mount/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  disable_iss_validation=true

# Create Kubernetes auth role
vault write auth/demo-auth-mount/role/auth-role \
  bound_service_account_names=default \
  bound_service_account_namespaces=demo-ns \
  token_ttl=0 \
  token_max_ttl=120 \
  token_policies=demo-auth-policy-db \
  audience=vault
```

## Configure Transit engine in Vault

```bash
# Create a transit backend mount
vault secrets enable -path=demo-transit transit

# Create a cache secret cache configuration
# $ vault write demo-transit/config/caching size=500
vault write demo-transit/cache-config size=500

# Create a transit key
vault write -force demo-transit/keys/vso-client-cache

# Create a policy for the operator role
vault policy write demo-auth-policy-operator - <<EOF
path "demo-transit/encrypt/vso-client-cache" {
  capabilities = ["create", "update"]
}
path "demo-transit/decrypt/vso-client-cache" {
  capabilities = ["create", "update"]
}
EOF

# Create Kubernetes auth role
vault write auth/demo-auth-mount/role/auth-role-operator \
  bound_service_account_names=demo-operator \
  bound_service_account_namespaces=vault-secrets-operator-system \
  token_ttl=0 \
  token_max_ttl=120 \
  token_policies=demo-auth-policy-db \
  audience=vault

exit
```

## Create the App

```bash
kubectl create ns demo-ns

kubectl apply -f vault/dynamic-secrets/.
```

### For OpenShift

> **📌 프로덕션 환경에서는 권장하지 않습니다.**

```bash
oc create sa demo-sa -n demo-ns
oc adm policy add-scc-to-user privileged -z demo-sa -n demo-ns
oc set sa deployment vso-db-demo demo-sa -n demo-ns
```

## Verify the App pods are running

```bash
kubectl get pods -n demo-ns
kubectl get secret vso-db-demo -n demo-ns -o json | jq -r .data._raw | base64 -D
kubectl get secret vso-db-demo-created -n demo-ns -o json | jq -r .data._raw | base64 -D
```