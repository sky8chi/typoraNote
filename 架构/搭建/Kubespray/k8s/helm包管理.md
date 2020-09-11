> https://helm.sh/docs/intro/install/

# 下载

```shell
wget https://get.helm.sh/helm-v3.3.1-linux-amd64.tar.gz
tar -zxvf helm-v3.3.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```



# 修改源

```shell
# 先移除原先的仓库
helm repo remove stable
# 添加新的仓库地址
helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
# 更新仓库
helm repo update
```



# 命令补全

```
为了方便 helm 命令的使用，helm 提供了自动补全功能，如果使用 zsh 请执行：

source <(helm completion zsh)
如果使用 bash 请执行：

source <(helm completion bash)
```



# 命令

```shell
# 查看配置
helm  inspect values  stable/prometheus
# 下载chart包
helm  fetch stable/prometheus
# 服务的升级
helm  upgrade --set imageTag=5.7.15  bdqn-mysql stable/mysql -f value
# 回滚
helm  history  zhb
helm  rollback  zhb  1
```

