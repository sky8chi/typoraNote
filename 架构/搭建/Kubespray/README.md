# 机器准备

## 机器规划

* [机器规划](机器规划)

## 机器系统

centos8.3.2011

用esxi安装系统建议手动分区，不同的空间或磁盘会导致分区不一样



# 部署准备

## 进入控制机l1

```shell
yum install git -y

cd /data/gitwork
# 机器控制集成
git clone git@github.com:sky8chi/devops.git
# k8s 集成软件 (我的clone修改, 官方看下面安装)
git clone git@github.com:sky8chi/kubespray.git
git checkout my-2.14
```

## 机器基本安装

[安装ansible](../ansible/安装.)

```shell
cd /data/gitwork/devops
# 配置机器
vim inventory/cluster/hosts.yml
# centos8 没有liblinux-python模块 改用python3替换， 修改版本为python3 (  ansible_python_interpreter: /usr/bin/python3 )
vim inventory/cluster/group_vars/all/all.yml
# 如果非centos8 可能默认python2, yum默认绑定也是python2，使用python3运行yum会出错，针对特殊每台机器指定python2 (  ansible_python_interpreter: /usr/bin/python2 )
vim inventory/cluster/hosts.yml

sh run_base.sh
```



# kubespary安装

* [安装向导](install.md)

