# 查看所有

```shell
kubectl get namespace
```



# 查看当前

```shell
kubectl config get-contexts
```

# 切换

```shell
kubectl config set-context --current --namespace=mongo
```

# 切回

```shell
kubectl config set-context --current --namespace=""
```

