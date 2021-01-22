# 安装pip

```shell
yum -y install epel-release
yum -y install python3
#yum -y install python3-pip
```

# clone项目

> https://github.com/kubernetes-sigs/kubespray

```shell
git clone https://github.com/kubernetes-sigs/kubespray.git
# 线上不使用master分支，建议切换至指定版本
git checkout release-2.14

# 我的版本
git checkout my-2.14

```

# 准备

```shell
cd kubespray/
pip3 install -r requirements.txt
# 如果源有问题，切换
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple -r requirements.txt

cp -rfp inventory/sample inventory/mycluster

# 执行自动生成 inventory/mycluster/hosts.yaml，后续调整应用节点自己修改
declare -a IPS=(10.200.120.241 10.200.120.242 10.200.120.243 10.200.120.244 10.200.120.245 10.200.120.246 10.200.120.247 10.200.120.248)

CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

```

[开启nginx-ingress](k8s/ingress-nginx.md)

## 修改私有云

* 国内访问可能涉及的镜像修改

> https://chenyongjun.vip/articles/128

```
https://storage.googleapis.com/kubernetes-release/release/v1.18.4/bin/linux/amd64/kubeadm
https://storage.googleapis.com/kubernetes-release/release/v1.18.4/bin/linux/amd64/kubectl
https://storage.googleapis.com/kubernetes-release/release/v1.18.4/bin/linux/amd64/kubelet
```

* 我的私有云

[建私有云方法](翻墙修改.md)

可以直接使用我的私有云：registry.cn-shanghai.aliyuncs.com/gcr_org

```shell
# 修改配置
# 主要是 group_vars/all/all.yml 和 group_vars/k8s-cluster/k8s-cluster.yml
# 优先级 k8s-cluster.yml > all.yml > roles/xxx/defalut/main.yml
# 所以想要覆盖 role 里面的默认配置, 优先看 k8s-cluster.yml 里面是否有同名配置, 如果有就同时修改 k8s-cluster.yml 和 all.yml, 没有就在 all.yml 里面添加
# 或者直接使用 ansible-playbook -e @foo.yml 的方式, 因为 -e 指定的变量具有最高优先级
# kubespray 常用变量参考: https://kubespray.io/#/docs/vars?id=common-vars-that-are-used-in-kubespray


vim inventory/mycluster/group_vars/all/download.yml
# 我是新建文件统一管理 其他人可以放在 vim inventory/mycluster/group_vars/all/all.yml
# 加载内核模块，否则 ceph, gfs 等无法挂载客户端
kubelet_load_modules: true


gcr_image_repo: "registry.cn-shanghai.aliyuncs.com/gcr_org"
nodelocaldns_image_repo: "{{ gcr_image_repo }}/dns_k8s-dns-node-cache"
dnsautoscaler_image_repo: "{{ gcr_image_repo }}/cluster-proportional-autoscaler-{{ image_arch }}"
ingress_nginx_controller_image_repo: "{{ gcr_image_repo }}/ingress-nginx_controller"

# kube_image_repo 貌似没有使用到
kube_image_repo: "registry.cn-shanghai.aliyuncs.com/kube_org"

quay_image_repo: "registry.cn-shanghai.aliyuncs.com/quay_org"
calico_node_image_repo: "{{ quay_image_repo }}/calico_node"
etcd_image_repo: "{{ quay_image_repo }}/coreos_etcd"
calico_cni_image_repo: "{{ quay_image_repo }}/calico_cni"
calico_policy_image_repo: "{{ quay_image_repo }}/calico_kube-controllers"

# 不同系统转阿里云，可以不设置，默认docker.com官方
# docker_fedora_repo_base_url：
# docker_ubuntu_repo_base_url：
# docker_debian_repo_base_url：
# docker_fedora_repo_base_url：
docker_rh_repo_base_url: "http://mirrors.aliyun.com/docker-ce/linux/centos/8/x86_64/stable/"
docker_rh_repo_gpgkey: 'http://mirrors.aliyun.com/docker-ce/linux/centos/gpg'



# 如果访问不通， 按需修改你的地址
kubelet_download_url:
kubectl_download_url: 
kubeadm_download_url:


# 修改下载缓存在本地加快集群部署，每台机器都从网上拉太慢了, 缓存目录 /tmp/kubespray_cache/
# 链接:https://pan.baidu.com/s/16EQhkr2ADHBuq0ca2RSLwQ  密码:q1ul
# 可以提前下载放到 /tmp/kubespray_cache/ 下，这样就不用下载直接使用了，前提是使用我的版本
download_run_once: true
download_localhost: true


vim inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
kube_image_repo: "registry.cn-shanghai.aliyuncs.com/gcr_org"
```



* 修改过镜像地址，执行本地预下载

  ```shell
  # 如果没修改本地下载缓存，请忽略
  # 要进行本地缓存，需要当前机器支持docker
  # 安装见https://docs.docker.com/engine/install/centos/
  # 注 centos8 安装会有问题冲突   yum erase podman buildah
  
  ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml --tags download --skip-tags upload,upgrade
  ```

  

  

# 开启ingress-nginx

[ingress-nginx](k8s/ingress-nginx.md)



# 安装

```shell
# 如果报sudo: sorry, you must have a tty to run sudo
# 修改ansible.cfg 中 pipelining = False  （会降速）
# 或者受控端 /etc/sudoers文件中关闭requiretty

cp inventory/mycluster/inventory.ini inventory/mycluster/hosts.yaml

# -e download_run_once=true -e download_localhost=true 下载到本地
# https://techdocs.broadcom.com/us/en/ca-enterprise-software/it-operations-management/dx-platform-on-premise/1-0/installing/reference-information/install-kubernetes-using-kubespray-offline.html
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```



# 安装异常节点处理

可以尝试先删除失败节点，再新增节点

```
centos: x509: certificate has expired or is not yet valid 这种错误,一般都是本地系统时间错误导致报错证书过期,所以先查看本地系统时间

systemctl enable chronyd; systemctl restart chronyd
```



# 验证安装是否成功

登录Kubernete集群的Mater集群，执行如下命令：

```
kubectl get no
```



# 管理节点

## 部署节点

```shell
ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml -b -vvv
```

## 扩容节点

```shell
# 只能用来添加node节点，不能添加master节点
# 在hosts.yml新增节点执行   此操作实测，只有新增时有效，无法用于恢复节点， 可以通过先卸载节点再操作
ansible-playbook -i inventory/mycluster/hosts.yaml scale.yml -b
```

## 删除节点

```shell
# 删除配置里的集群 host，不是像扩容一样差异化，切记，别把集群删除了
ansible-playbook -i inventory/mycluster/hosts.yaml remove-node.yml -b -v 
# 如果文件中只有node6 则会进入 notready状态
node6   NotReady   <none>   24m   v1.18.4


# 正确用法：如果文件中是所有节点，可以指定节点删除，删除了node6同时NotReady也会不存在
--extra-vars "node=<nodename>,<nodename2>"
ansible-playbook -i inventory/mycluster/hosts.yaml remove-node.yml -b -v --extra-vars "node=node6"

# 貌似不以恢复master， master节点添加执行部署节点
# 方法一:
# 如果要恢复节点，需要先执行一遍新增节点，会报kubelet错误
# 去恢复节点执行 systemctl daemon-reload， 这样kubelet服务就会启动
# 再次执行新增节点，就成功了  （如果想跳过第一步执行一次新增，先启动kubelet，再执行新增，貌似会不成功，可能缺少部分东西）

# 方法二:
# 直接卸载节点，再新增




```

## 卸载节点

```shell
# 除非卸载整个集群，否则不要直接卸载，先删除节点再卸载，否则节点在集群中还是存在的，只是not ready状态
ansible-playbook -i inventory/mycluster/hosts.yaml reset.yml -b
```

## 升级节点

```shell
ansible-playbook upgrade-cluster.yml -b -i inventory/mycluster/hosts.yaml -e kube_version=vX.XX.XX -vvv
```

## 注意事项

* 节点名字小写