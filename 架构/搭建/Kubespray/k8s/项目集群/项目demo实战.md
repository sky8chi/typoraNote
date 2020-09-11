# 代码上传见

[harbor仓库](../项目基础应用/harbor仓库.md)

# k8s配置

Vim namespace.yml

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: springbootdemo
  labels:
    name: springbootdemo
```



```yaml
# cat configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: log
      paths:
        - "/var/log/yum.log"
        - "/log/*"
    output.elasticsearch:
      hosts: ["192.168.1.19:9600"]
      index: "filebeat-tomcat-shopxx"
      
# 多份 cat configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: log
      enabled: true 
      paths:
        - "/log/error.log"
      multiline.pattern: '^[0-9]'
      multiline.negate: true
      multiline.match: after
      fields:
        type: "error"
    - type: log
      paths:
        - "/log/warn.log"
      multiline.pattern: '^[0-9]'
      multiline.negate: true
      multiline.match: after
      fields:
        type: "warn"
    - type: log
      paths:
        - "/log/info.log"
      multiline.pattern: '^[0-9]'
      multiline.negate: true
      multiline.match: after
      fields:
        type: "info"
    output.elasticsearch:
      hosts: ["elasticsearch-master.elk:9200"]
      indices:
      - index: "springbootdemo-info-log"
        when.equals:
          fields.type: "info"
      - index: "springbootdemo-warn-log"
        when.equals:
          fields.type: "warn"
      - index: "springbootdemo-error-log"
        when.equals:
          fields.type: "error"
```



vim deployment.yml

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: springbootdemo
  namespace: springbootdemo
  labels:
    app: springbootdemo
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: springbootdemo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springbootdemo
  namespace: springbootdemo
  labels:
    app: springbootdemo
spec:
  replicas: 3 
  selector:
    matchLabels:
      app: springbootdemo
  template:
    metadata:
      labels:
        app: springbootdemo
    spec:
      containers:
      - image: docker.elastic.co/beats/filebeat:7.9.1
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        imagePullPolicy: Always
        name: filebeat
        volumeMounts:
        - name: app-logs
          mountPath: /log
        - name: filebeat-config
          mountPath: /etc/filebeat.yml
          subPath: filebeat.yml
      - name: springbootdemo
        image: harbor.kxfo.com/mall/demo:1.0.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080 
        readinessProbe:
          initialDelaySeconds: 20
          periodSeconds: 5
          timeoutSeconds: 10
          httpGet:
            scheme: HTTP
            port: 8080
            path: /test/keepAlive
        livenessProbe:
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          httpGet:
            scheme: HTTP
            port: 8080
            path: /test/keepAlive
        volumeMounts:
        - name: app-logs
          mountPath: /home/tomcat/logs
      volumes:
      - name: app-logs
        emptyDir: {}
      - name: filebeat-config
        configMap:
          name: filebeat-config
```



Vim ingress.yml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: k8s-springbootdemo
  namespace: springbootdemo
  annotations:
    nginx.ingress.kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "http"
spec:
  rules:
  - host: springbootdemo.kube.kxfo.com
    http:
      paths:
      - path: /
        backend:
          serviceName: springbootdemo
          servicePort: 8080
```



# 项目更新

## 配置文件方式

```
sed -i 's/demo:1.0.0.0/demo:2.0.0.0/g' deployment.yml

kubectl apply -f deployment.yml

kubectl get pod
curl ""
```

## 使用patch命令

```shell
kubectl get deploy

NAME             READY   UP-TO-DATE   AVAILABLE   AGE
springbootdemo   3/3     3            3           18m


kubectl patch deployment springbootdemo --patch '{"spec": {"template": {"spec": {"containers": [{"name": "springbootdemo","image":"harbor.kxfo.com/mall/demo:1.0.0.0"}]}}}}'

kubectl get pod
```

## 使用set image命令

```shell
kubectl set image deploy springbootdemo *=harbor.kxfo.com/mall/demo:2.0.0.0

kubectl get pod
```

# 扩容缩容

```
kubectl scale deploy springbootdemo -n springbootdemo --replicas=4
```



