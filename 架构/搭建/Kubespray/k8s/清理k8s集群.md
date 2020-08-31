# 删除镜像

> https://blog.csdn.net/sdmei/article/details/100746092

## 根据镜像名，正则删除，不大靠谱

ansible-playbook -i hosts clean_k8s.xml

```
---
- hosts: k8s
  gather_facts: False
  tasks:
    - name: "clean images"
      shell: docker images |grep 'harbor.kxfo.com/mall/demo'|awk '{print $3}'|awk 'BEGIN {FS=" "} NR > 1 {print $NF}' |xargs docker rmi
```

## 根据镜像label 及根据镜像创建时间删除

打包镜像时，需要添加 LABEL app=springbootDemo 用来搜索镜像

ansible-playbook -i hosts clean_k8s.xml

```
---
- hosts: k8s
  gather_facts: False
  tasks:
    - name: "clean images"
      shell: docker image prune -a --force --filter "label=app=springbootDemo" --filter "until=24h"

```



