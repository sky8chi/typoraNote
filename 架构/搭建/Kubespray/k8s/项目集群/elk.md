> https://www.cnblogs.com/Dev0ps/p/11465673.html

# 创建命名空间

Vim namespace.yml

```shell
---
apiVersion: v1
kind: Namespace
metadata:
  name: elk
  labels:
    name: elk
```

kubectl apply -f namespace.yml

# 切换命名空间

```
kubectl config set-context --current --namespace=elk
```



# helm仓库

```
helm repo add elastic https://helm.elastic.co
```



# 安装elk

```
helm fetch elastic/elasticsearch --version 7.9.1

tar -zxvf elasticsearch-7.9.1.tgz
```

## 修改配置

```yaml
# 存储
volumeClaimTemplate:
  accessModes: ["ReadWriteOnce"]
  storageClassName: managed-nfs-storage
  resources:
    requests:
      storage: 100Gi
```

```shell
helm install --namespace elk --name-template my-elk -f values.yaml .
```

 

# 安装filebeat

```shell
helm fetch elastic/filebeat --version 7.9.1
tar -zxvf filebeat-7.9.1.tgz
```



```yaml
filebeatConfig:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      # 添加java多行收集 
      multiline.pattern: '^[0-9]'
      multiline.negate: true
      multiline.match: after
```



```
helm install --namespace elk my-filebeat -f values.yaml .

kubectl get pods --namespace=elk -l app=my-filebeat-filebeat
```



# 安装kibana

```
helm fetch elastic/kibana --version 7.9.1
tar -zxvf kibana-7.9.1.tgz
```



```

ingress:
  enabled: true
  annotations:
    nginx.ingress.kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "http"
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  path: /
  hosts:
    - kibana.kube.kxfo.com
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local
```

```
helm install --namespace elk my-kibana -f values.yaml .

```

