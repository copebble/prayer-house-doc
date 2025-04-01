# cert manager

about how to set up the **cert manager** in k8s cluster and with **AWS Route53**

- [cert manager official docs](https://cert-manager.io/docs/)

<br>

## :pushpin: installation

In my case, I installed the cert manager by using helm chart

[how to install with helm](https://cert-manager.io/docs/installation/helm/)

```shell
helm repo add jetstack https://charts.jetstack.io
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.0 \
  --set crds.enabled=true
```

<br>

## :pushpin: setting k8s objects

Three objects was applied.

- ClusterIssuer
- Certificate
- Secret (to use Route53 IAM)
