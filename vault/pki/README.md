# Working with PKI certificates

그림

## Configure Vault

1. Vault Shell 접근
```bash
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```

2. PKI 시크릿엔진 활성화
```bash
vault secrets enable -path=pki pki
```

3. PKI Roles 생성 : 미리 Role을 구성해 놓으면 사용자 및 앱은 지정된 규칙에 따라 인증서를 발급받을 수 있음
```bash
vault write pki/roles/secret -<<EOF
{
  "ttl": "3600",
  "allow_ip_sans": true,
  "key_type": "rsa",
  "key_bits": 4096,
  "allowed_domains": ["example.com"],
  "allow_subdomains": true,
  "allowed_uri_sans": ["uri1.example.com", "uri2.example.com"]
}
EOF
```

4. ROOT CA 생성
```bash
vault write pki/root/generate/internal \
  common_name="Root CA" \
  ttl="315360000" \
  format="pem" \
  private_key_format="der" \
  key_type="rsa" \
  key_bits=4096 \
  exclude_cn_from_sans=true \
  ou="My OU" \
  organization="My organization"
```

## Kubernetes Auth 설정
```bash
vault auth enable -path demo-pki kubernetes

vault write auth/demo-pki/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    disable_iss_validation=true

vault write auth/demo-pki/role/pki-role -<<EOF
{
  "bound_service_account_names": ["default"],
  "bound_service_account_namespaces": ["pki-demo-ns"],
  "token_ttl": 3600,
  "token_policies": ["pki-dev"],
  "audience": "vault"
}
EOF

vault policy write pki-dev -<<EOF
path "pki/*" {
  capabilities = ["read", "create", "update"]
}
EOF
```

## 네임스페이스 생성 및 PKI Secrets CRD 생성

1. 네임스페이스 생성 : `pki-demo-ns`
```bash
kubectl create namespace pki-demo-ns
```

2. 시크릿 생성 : `pki-tls`
```bash
kubectl create secret generic pki-tls -n pki-demo-ns

# VaultPKISecret CRD 배포
kubectl apply -f vault/pki/vault-pki-secret.yaml
```

3. PKI Secrets 생성확인

```bash
# 명령어 확인 추가
# kubectl get secret secretkv -n app -o json | jq -r .data._raw | base64 -D
```

## 샘플 애플리케이션 배포
- Ingress
- SVC
- Deployment

```bash
kubectl apply -f vault/pki/app-pki-deploy-ingress-svc.yaml
```

## Secrets 변경 및 Sync 확인

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

---

## Ingress Controller 배포

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```