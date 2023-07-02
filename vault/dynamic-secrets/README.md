# Working with dynamic secrets

> 본 챕터에서는 Vault Dynamic Secrets을 

## Postgres Server 배포

1. 네임스페이스 생성 : postgres
```bash
kubectl create ns postgres
```

2. Bitnami Chart 저장소 추가
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

3. postgresql 설치
```bash
helm upgrade --install postgres bitnami/postgresql --namespace postgres --set audit.logConnections=true --set auth.postgresPassword="HashiCorp@"
```

4. 발급된 POSTGRES_PASSWORD 환경변수 설정
```bash
export POSTGRES_PASSWORD=$(kubectl get secret --namespace postgres postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
```

5. POSTGRES_PASSWORD 확인
```bash
echo $POSTGRES_PASSWORD
```

### For OpenShift

```bash
helm upgrade --install postgres bitnami/postgresql --namespace postgres -f postgres/values.yaml

export POSTGRES_PASSWORD=$(kubectl get secret --namespace postgres postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

echo $POSTGRES_PASSWORD
```

## Vault와 Postgres 연동준비

### 1) Postgres 구성준비

1. Vault pod shell 접근
```bash
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```

2. 시크릿 엔진 활성화 : database
```bash
vault secrets enable -path=demo-db database
```

3. DB 플러그인(Plugin) 및 구성(Config)
```bash
vault write demo-db/config/demo-db \
  plugin_name=postgresql-database-plugin \
  allowed_roles="dev-postgres" \
  connection_url="postgresql://{{username}}:{{password}}@postgres-postgresql.postgres.svc.cluster.local:5432/postgres?sslmode=disable" \
  username="postgres" \
  password="HashiCorp@"

# 출력예시
Success! Data written to: demo-db/config/demo-db
```

### 2) Role & Plolicy 생성

1. Role 생성
- 주기적으로 DB Credentials 정보가 변경되는 것을 확인하기 위해 TTL 1m(60s) 설정
- 데모시연 시 실제 K8s Secret 정보가 Sync되고 demo deployment 앱이 재기동 되는 것을 확인할 수 있음
```bash
vault write demo-db/roles/dev-postgres \
  db_name=demo-db \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
      GRANT ALL PRIVILEGES ON DATABASE postgres TO \"{{name}}\";" \
  backend=demo-db \
  name=dev-postgres \
  default_ttl="1m" \
  max_ttl="1m"
```

2. 정책생성
```bash
vault policy write demo-auth-policy-db - <<EOT
path "demo-db/creds/dev-postgres" {
  capabilities = ["read"]
}
EOT
```

## Kubernetes Setup

1. K8s Auth Method 활성화 및 Config 설정
```bash
# Enable Kubernetes auth backend
vault auth enable -path demo-auth-mount kubernetes

# K8s Auth 구성
vault write auth/demo-auth-mount/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  disable_iss_validation=true
```

2. Kubernetes Auth Role 생성
```bash
vault write auth/demo-auth-mount/role/auth-role \
  bound_service_account_names=default \
  bound_service_account_namespaces=demo-ns \
  token_ttl=0 \
  token_max_ttl=120 \
  token_policies=demo-auth-policy-db \
  audience=vault
```

## Transit 엔진 설정
> Transit 사용하는 이유 : 

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

## 샘플 애플리케이션 생성(Create the application)

1. 네임스페이스 생성
```bash
kubectl create ns demo-ns
```

2. 리소스 배포
- Deployment, SVC, SA, Secrets
- Vault Connection, Vault Authentication
```bash
kubectl apply -f vault/dynamic-secrets/.

# 배포결과
deployment.apps/vso-db-demo created
secret/vso-db-demo created
vaultauth.secrets.hashicorp.com/dynamic-auth created
vaultauth.secrets.hashicorp.com/transit-auth created
vaultdynamicsecret.secrets.hashicorp.com/vso-db-demo-create created
vaultdynamicsecret.secrets.hashicorp.com/vso-db-demo created
serviceaccount/demo-operator created
```

### For OpenShift

> **📌 프로덕션 환경에서는 권장하지 않습니다.**

```bash
oc create sa demo-sa -n demo-ns
oc adm policy add-scc-to-user privileged -z demo-sa -n demo-ns
oc set sa deployment vso-db-demo demo-sa -n demo-ns
```

## 배포된 파드 및 시크릿 확인

```bash
kubectl get secret vso-db-demo -n demo-ns -o json | jq -r .data._raw | base64 -d
kubectl get secret vso-db-demo-created -n demo-ns -o json | jq -r .data._raw | base64 -d
```

```bash
kubectl get pods -n demo-ns
POD1=$(kubectl get pods -n demo-ns -o jsonpath='{.items[0].metadata.name}')
POD2=$(kubectl get pods -n demo-ns -o jsonpath='{.items[1].metadata.name}')

# 파드1 Shell 에서 /etc/secrets 확인
kubectl exec -n demo-ns -it $POD1 -- cat /etc/secrets/username
kubectl exec -n demo-ns -it $POD1 -- cat /etc/secrets/password

# 파드2 Shell 에서 /etc/secrets 확인
kubectl exec -n demo-ns -it $POD2 -- cat /etc/secrets/username
kubectl exec -n demo-ns -it $POD2 -- cat /etc/secrets/password
```

### 확인 명령

```bash
# PostgreSQL pod에 접속
kubectl exec -n postgres postgres-postgresql-0 -it -- /bin/sh

# 로그인
export PGPASSWORD="HashiCorp@"
psql -U postgres -c "SELECT usename, valuntil FROM pg_user;"
```