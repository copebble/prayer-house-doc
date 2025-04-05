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