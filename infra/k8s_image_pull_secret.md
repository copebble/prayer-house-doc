# Image Pull Secret 관련

k8s deployment 새로 배포할 때 image에 대한 추가 설정 필요
본인 프로젝트는 AWS ECR에 docker image를 push하고 해당 image를 pull 받아서 pod에 적용하기 때문에 AWS ECR에 대한 image pull secret이 필요한 상황

AWS ECR이 6시간?, 12시간? 마다 갱신을 해줘야 하는데 갱신하지 않으면 CI/CD 자동화 과정에서 새로운 pod 생성할 때 image pull 관련 에러 발생

> **목표**
> k8s cluster에 새로운 deployment 스펙을 적용하려고 하는데 image pull secret 필수
> 해당 secret은 주기적으로 갱신해야 하는 이슈가 존재하기에 k8s cronjob을 통해 자동으로 갱신하도록 설정하는 것이 목표

<br>

## :pushpin: References

https://b.cublr.com/posts/kubernetes에서-aws-ecr-인증-정보를-자동-갱신하기