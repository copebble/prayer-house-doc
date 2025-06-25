# kubectl useful command

<br>

## :pushpin: deletion

```shell
kubectl delete pod <pod_name> -n <namespace> --grace-period 0 --force
```
delete pod forcefully

<br>

## :pushpin: labelling

- node labelling

```shell
kubectl label node [NODE_NAME] node-role.kubernetes.io/worker=
kubectl get nodes
```
When new node joined the cluster, it's role will be represented `<none>`.
It can be set to specific role by kubectl command.

```shell
kubectl label node [NODE_NAME] node-role.kubernetes.io/worker-
```
If you want to delete specific label of node, just replace `=` with `-`.

<br>

## :pushpin: Get objects

- get all active pods in specific node.

### Secret

- retrieve decoded secret data value

```shell
kubectl get secret [SECRET_NAME] -o jsonpath='{.data}' | base64 --decode
```
just input `-o jsonpath` option.
```shell
# data:
#   hello.world: xxxx
kubectl get secret [SECRET_NAME] -o jsonpath='{.data.hello\.world}' | base64 --decode
```
In case the key name has any character like `.`, escape character should be input with that character.

- create secret

```shell
kubectl -n [namespace] create secret generic [secret-name] --from-file=key-name=[path/to/key.pem]
```

<br>

## :pushpin: network check

```shell
kubectl run netshoot-tools --rm -i --tty --image nicolaka/netshoot
```