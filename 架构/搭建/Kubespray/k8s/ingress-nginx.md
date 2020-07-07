# 修改kubespray配置

```shell
vim inventory/sample/group_vars/k8s-cluster/addons.yml
ingress_nginx_enabled: true			//开启nginx暴露代理
ingress_nginx_host_network: true	//使用主机网络
```

