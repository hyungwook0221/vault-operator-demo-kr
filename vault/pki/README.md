# Working with PKI certificates

그림

## Configure Vault

1. Vault Shell 접근
```bash
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```

2. PKI 시크릿엔진 활성화
```bash
vault secrets enable pki
```

3. PKI Roles 생성 : 미리 Role을 구성해 놓으면 사용자 및 앱은 지정된 규칙에 따라 인증서를 발급받을 수 있음
```bash
vault write pki/roles/default \
    allowed_domains=example.com \
    allowed_domains=localhost \
    allow_subdomains=true \
    max_ttl=72h
```

4. CRL 생성
```bash
vault write pki/config/urls \
    issuing_certificates="http://127.0.0.1:8200/v1/pki/ca" \
    crl_distribution_points="http://127.0.0.1:8200/v1/pki/crl"
```

5. ROOT CA 생성
```bash
vault write pki/root/generate/internal \
    common_name=example.com \
    ttl=72h
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

exit
```

## 네임스페이스 생성 및 PKI Secrets CRD 생성

1. 네임스페이스 생성 : `pki-demo-ns`
```bash
kubectl create namespace pki-demo-ns
```

2. 각종 리소스 배포
- Ingress
- SVC
- Deployment

```bash
kubectl apply -f vault/pki/.
```

3. Ingress Controller 배포

> 필자는 인증서에 대한 갱신 및 활용방안을 위해 IngressController를 추가배포 후 실습을 진행합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

4. PKI Secrets 생성확인

```bash
# 명령어 확인 추가
kubectl get secret pki-tls -n pki-demo-ns -o json | jq -r .data._raw | base64 -d
```

![img](https://raw.githubusercontent.com/hyungwook0221/img/main/uPic/9GmI0T.jpg)


> 📌 참고 : 30초마다 데이터 갱신을 확인  
> Vault KV 저장소에 저장된 데이터 수정 후 실제 Secret에서 데이터 변경되는지 확인!

## 인증서 확인(명령어)
```bash
$ curl -k https://localhost:38443/tls-app/hostname
tls-app

$ curl -kvI https://localhost:38443/tls-app/hostname
# 갱신 전
* Server certificate:
*  subject: CN=localhost
*  start date: Jun 30 13:03:31 2023 GMT
*  expire date: Jun 30 13:05:01 2023 GMT
*  issuer: CN=example.com
...

# 갱신 후
* Server certificate:
*  subject: CN=localhost
*  start date: Jun 30 13:04:16 2023 GMT
*  expire date: Jun 30 13:05:46 2023 GMT
*  issuer: CN=example.com

...
```

## 인증서 확인(UI)

- 인증서 자동갱신 전
![img](https://raw.githubusercontent.com/hyungwook0221/img/main/uPic/HazyNh.jpg)

- 인증서 자동갱신 후
![img](https://raw.githubusercontent.com/hyungwook0221/img/main/uPic/VyhNaq.jpg)