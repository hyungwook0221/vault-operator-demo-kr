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
      bound_service_account_namespaces=demo-ns \
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
# demo-ns 네임스페이스 생성(다른 실습에서 생성하였다면 생략)
kubectl create ns demo-ns 

# VaultStaticSecret CRD 배포
kubectl apply -f vault/static-secrets/vault-kv-secret.yaml
kubectl apply -f vault/static-secrets/vault-kvv2-secret.yaml
```

## Verify the static secrets were created

```bash
kubectl get secret secretkv -n demo-ns -o json | jq -r .data._raw | base64 -D
kubectl get secret secretkvv2 -n demo-ns -o json | jq -r .data._raw | base64 -D
```

## Create the App

```bash
kubectl apply -f vault/static-secrets/app-static-deployment.yaml
# kubectl apply -f vault/static-secrets/app-static-secret.yaml
```

### Check `index.html` Files on Web UI
- static-user-kvv2 및 static-password-kvv2 확인
<!-- ![img](https://raw.githubusercontent.com/hyungwook0221/img/main/uPic/ZUjqTL.jpg) -->
<p align="center">
  <img src="https://raw.githubusercontent.com/hyungwook0221/img/main/uPic/ZUjqTL.jpg" width=50% height=50%>
</p>

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
kubectl get secret secret-kv -n demo-ns -o json | jq -r .data._raw | base64 -D
kubectl get secret secret-kvv2 -n demo-ns -o json | jq -r .data._raw | base64 -D
```

### Check `index.html` Files on Web UI
- new-static-user-kvv2 및 new-static-password-kvv2 로 변경된 것 확인
<!-- ![img](https://raw.githubusercontent.com/hyungwook0221/img/main/uPic/Xozc0D.jpg) -->
<p align="center">
  <img src="https://raw.githubusercontent.com/hyungwook0221/img/main/uPic/Xozc0D.jpg" width=50% height=50%>
</p>