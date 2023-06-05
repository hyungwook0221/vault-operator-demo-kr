# Working with PKI certificates

그림

## Configure Vault

```bash
# Vault Shell 접근
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh

# Kubernetes 인증 활성화(다른 실습에서 생성하였다면 생략)
vault auth enable kubernetes

# Kubernetes 인증구성(다른 실습에서 생성하였다면 생략)
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

# PKI 시크릿엔진 활성화
vault secrets disable pki
vault secrets enable pki

# PKI 정책추가
vault policy write pki-policy - <<EOF
path "pki/*" {
  capabilities = ["read", "create", "update"]
}
EOF

# Role 추가
vault write auth/kubernetes/role/pki-role \
    bound_service_account_names=default \
    bound_service_account_namespaces=demo-ns \
    policies=pki-policy \
    audience=vault \
    ttl=24h

# root CA 생성
vault write pki/root/generate/internal \
    common_name=example.com \
    ttl=768h

# CRL 생성 : Certificate Revocation List(인증서 해지 목록) 엔드포인트 작성
vault write pki/config/urls \
    issuing_certificates="http://127.0.0.1:8200/v1/pki/ca" \
    crl_distribution_points="http://127.0.0.1:8200/v1/pki/crl"

# Role 생성 : 미리 Role을 구성해 놓으면 사용자 및 앱은 지정된 규칙에 따라 인증서를 발급받을 수 있음
vault write pki/roles/default \
    allowed_domains=example.com \
    allowed_domains=localhost \
    allow_subdomains=true \
    max_ttl=72h

exit
```

## Create a new namespace for the demo app & the PKI secret CRDs

```bash
# demo-ns 네임스페이스 생성(다른 실습에서 생성하였다면 생략)
kubectl create namespace demo-ns

# VaultPKISecret CRD 배포
kubectl apply -f vault/pki/vault-pki-secret.yaml
```

## Verify the PKI secrets were created

```bash
# 명령어 확인 추가
# kubectl get secret secretkv -n app -o json | jq -r .data._raw | base64 -D
```

## Create the App
- Ingress + SVC + Deployment 배포

```bash
kubectl apply -f vault/pki/app-pki-deploy-ingress-svc.yaml
```

## Change the secrets and verify they are synced

```bash
# Vault Shell 접근
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh

exit
```

## Verify the static secrets were updated (wait 30s)
> 📌 참고 : 30초마다 데이터 갱신을 확인  
> Vault KV 저장소에 저장된 데이터 수정 후 실제 Secret에서 데이터 변경되는지 확인!

```bash
# 생성된 Secret 데이터 확인
curl -kv

```