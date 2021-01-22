# 修改kubespray配置

```shell
# sample 目录根据自己复制的配置名修改
vim inventory/mycluster/group_vars/k8s-cluster/addons.yml
ingress_nginx_enabled: true			//开启nginx暴露代理
ingress_nginx_host_network: true	//使用主机网络
```

