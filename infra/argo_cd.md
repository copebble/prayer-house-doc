# Argo CD

- [공식 문서](https://argo-cd.readthedocs.io/en/stable/) 
- [공식 문서 installation](https://argo-cd.readthedocs.io/en/stable/getting_started/)

CD 툴로서 github action과 같이 사용하는 케이스가 많아보여서 선택하게 됐음
우선 쿠베 컨테이너 특성에 맞는 편리한 UI를 제공해주는 것 같아 좋아보인다.

(실무에서는 jenkins 사용했는데 kubectl rollout 로그로 확인하니 답답해서 쿠베 dashboard, matrix 두 개 번갈아 봐야하는 불편함이 존재했음)

<br>

## :pushpin: Installation

helm chart 사용해서 설치했음
공식 문서에서 제공하는 기본 설치방법대로 kubectl apply로 yaml 스펙 그대로 설치해도 전혀 상관 없다.

다만 helm chart에서 `values.yaml`에 있는 내용에 설정 정보들을 custom하게 바꿀 수 있어서 본인은 helm chart 설치 방식 선택
(심지어 helm chart는 따로 Revision도 관리하고 있고 설정 내용을 변경해서 upgrade 해줄 수 있어서 사용하기 간편)

### Helm Installation

> helm 설치 전에 기본적으로 k8s cluster가 구축이 되어 있어야 하고 
> kubectl도 적용된 상태여야 함

helm chart가 설치 안된 상태라면 설치 필요
- [helm document](https://helm.sh/docs/intro/install/)

```shell
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```
서버가 debian 계열의 apt package를 사용하고 있기 때문에 apt 방식으로 설치

```shell
helm version
```

### helm install Argo CD

[argo-helm Github](https://github.com/argoproj/argo-helm/tree/main)

```shell
helm repo add argo https://argoproj.github.io/argo-helm
helm pull argo/argo-cd
```
helm repo를 먼저 설정하고 git pull 받듯이 argo-cd 내용을 땡겨온다.

```shell
tar -xzf argo-cd-7.5.0.tgz
cd argo-cd/
```
압축 풀고 이동

```shell
helm install argocd --create-namespace -n argocd ./ -f values.yaml
```
customizing이 몇 개 필요해서 `values-custom.yaml` 따로 copy해서 사용

(기본적인 설치는 완료)

<br>

## :pushpin: Argo CD UI 외부 노출(with HTTPS)

> nginx ingress controller 설치된 상태여야 함

argocd ui를 https 외부 노출을 