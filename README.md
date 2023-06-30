# Vault Operator Demo
이 저장소에는 다양한 클라우드 호스팅 Kubernetes 솔루션에서 Vault 오퍼레이터를 사용하는 방법에 대한 몇 가지 샘플을 제공합니다.

![img](https://raw.githubusercontent.com/hyungwook0221/img/main/uPic/vso-img2.png)

> 💡 참고  
Vault Secret Operator(VSO)에 대하여 더욱 자세한 내용은 다음에서 확인할 수 있습니다.
- [Vault Secrets Operator 개요](https://docmoa.github.io/04-HashiCorp/06-Vault/01-Information/vault-secret-operator/1-vso-overview.html)
- [Kubernetes Vault 통합방안 3가지 비교](https://docmoa.github.io/04-HashiCorp/06-Vault/04-UseCase/vault-k8s-integration-three-methods.html)

## Deploy your Kubernetes cluster
다음은 각종 Kubernetes 클러스터 배포를 위한 샘플 구성파일 입니다.
- KinD
- GKE
- EKS
- AKS

### KinD

```bash
# kind 클러스터 구성
kind get clusters | grep --silent "^kind$$" || kind create cluster --wait=5m \
    --image kindest/node:v1.25.3 --name kind --config infra/kind/config.yaml

# context 설정
kubectl config use-context kind-kind
```

### Google Cloud (GKE)

```bash
# 초기 설정 작업을 수행       
gcloud init

gcloud auth application-default login

echo 'project_id = "'$(gcloud config get-value project)'"' > infra/gke/terraform.tfvars && echo 'region = "us-west1"' >> infra/gke/terraform.tfvars

terraform -chdir=infra/gke/ init -upgrade

terraform -chdir=infra/gke/ apply -auto-approve

gcloud container clusters get-credentials $(terraform -chdir=infra/gke/ output -raw kubernetes_cluster_name) --region $(terraform -chdir=infra/gke/ output -raw region)
```

### AWS (EKS)

```bash
# AWS 리소스 접근을 위해 환경변수를 통해 인증
# aws credentials 명령도 가능
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
                          
terraform -chdir=infra/eks/ init -upgrade
terraform -chdir=infra/eks/ apply -auto-approve

aws eks --region $(terraform -chdir=infra/eks/ output -raw region) \
    update-kubeconfig --name $(terraform -chdir=infra/eks/ output -raw cluster_name)
```

### Azure (AKS)

```bash
az config set core.allow_broker=true && az account clear && az login
az account set --subscription <subscription_id>
az ad sp create-for-rbac --output json | jq -r '. | "appId = \"" + .appId + "\"\npassword = \"" + .password + "\"" ' > infra/aks/terraform.tfvars

terraform -chdir=infra/aks/ init -upgrade
terraform -chdir=infra/aks/ apply -auto-approve

az aks get-credentials --resource-group $(terraform -chdir=infra/aks/ output -raw resource_group_name) --name $(terraform -chdir=infra/aks/ output -raw kubernetes_cluster_name)
```

## Deploy Vault
Helm Chart를 통해 Vault Cluster를 dev 모드로 배포합니다.
```bash
# 저장소 추가
helm repo add hashicorp https://helm.releases.hashicorp.com

# 저장소 업데이트
helm repo update

# 저장소 추가확인
helm search repo hashicorp/vault

# vault 헬름차트 배포
helm install vault hashicorp/vault -n vault \
    --create-namespace --values vault/vault-server-values.yaml
```

For OpenShift

```bash
helm install vault hashicorp/vault -n vault \
    --create-namespace --values vault/vault-server-values.yaml \
    --set "global.openshift=true"
```

## Deploy the Vault Secret Operator(VSO)
Helm Chart를 통해 Vault Secret Operator를 배포합니다.
```bash
helm install vault-secrets-operator hashicorp/vault-secrets-operator \
    --version 0.1.0 -n vault-secrets-operator-system \
    --create-namespace --values vault/vault-operator-values.yaml
```

## Vault Secret Operator 활용방안 3가지

### * [Working with Static secrets](/vault/static-secrets/README.md)
### * [Working with Dynamic secrets](/vault/dynamic-secrets/README.md)
### * [Working with PKI](/vault/pki/README.md)