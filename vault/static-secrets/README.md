# Working with static secrets

<p align="center">
  <img src="https://content.hashicorp.com/api/assets?product=tutorials&version=main&asset=public%2Fimg%2Fvault%2Fkubernetes%2Fdiagram-secrets-operator.png">
</p>

## Configure Vault

```bash
# Vault Shell 접근
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh

# Kubernetes 인증 활성화
vault auth enable kubernetes

# Kubernetes 인증구성
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

# KV v1 / v2 시크릿엔진 활성화
vault secrets enable -path=kvv2 kv-v2
vault secrets enable -path=kv kv

# 정책추가
vault policy write dev - <<EOF
path "kv/*" {
  capabilities = ["read"]
}

path "kvv2/*" {
  capabilities = ["read"]
}
EOF

# Role 추가
vault write auth/kubernetes/role/role1 \
      bound_service_account_names=default \
      bound_service_account_namespaces=app \
      policies=dev \
      audience=vault \
      ttl=24h

# KV v1 샘플데이터 추가
vault kv put kv/webapp/config username="static-user" password="static-password"

# KV v2 샘플데이터 추가
vault kv put kvv2/webapp/config username="static-user-kvv2" password="static-password-kvv2"

exit
```

## Create a new namespace for the demo app & the static secret CRDs

```bash
# app 네임스페이스 생성
kubectl create ns app

# VaultStaticSecret CRD 배포
kubectl apply -f vault/static-secrets/vault-kv-secret.yaml
kubectl apply -f vault/static-secrets/vault-kvv2-secret.yaml
```

## Verify the static secrets were created

```bash
kubectl get secret secretkv -n app -o json | jq -r .data._raw | base64 -D
kubectl get secret secretkvv2 -n app -o json | jq -r .data._raw | base64 -D
```

## Change the secrets and verify they are synced

```bash
# Vault Shell 접근
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh

vault kv put kv/webapp/config username="new-static-user" password="new-static-password"

vault kv put kvv2/webapp/config username="new-static-user-kvv2" password="new-static-password-kvv2"

exit
```

## Verify the static secrets were updated (wait 30s)
> 📌 참고 : 30초마다 데이터 갱신을 확인  
> Vault KV 저장소에 저장된 데이터 수정 후 실제 Secret에서 데이터 변경되는지 확인!

```bash
# 생성된 Secret 데이터 확인
kubectl get secret secretkv -n app -o json | jq -r .data._raw | base64 -D
kubectl get secret secretkvv2 -n app -o json | jq -r .data._raw | base64 -D
```
