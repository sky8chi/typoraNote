# namespace

```yaml
vim namespace.yml
---
apiVersion: v1
kind: Namespace
metadata:
  name: easy-mock
  labels:
    name: easy-mock
```



# 配置

```shell
kubectl create configmap production-json --from-file=./production.json

{
    "port": 7300,
    "host": "0.0.0.0",
    "pageSize": 30,
    "proxy": false,
    "db": "mongodb://mongo-0.mongo.mongo.svc.cluster.local:27017,mongo-1.mongo.mongo.svc.cluster.local:27017,mongo-2.mongo.mongo.svc.cluster.local:27017,mongo-3.mongo.mongo.svc.cluster.local:27017/?replicaSet=rs0",
    "unsplashClientId": "",
    "redis": {
      "keyPrefix": "[Easy Mock]",
      "port": 6379,
      "host": "redis-service.redis",
      "password": "",
      "db": 0
    },
    "blackList": {
      "projects": [],
      "ips": []
    },
    "rateLimit": {
      "max": 1000,
      "duration": 1000
    },
    "jwt": {
      "expire": "14 days",
      "secret": "shared-secret"
    },
    "upload": {
      "types": [".jpg", ".jpeg", ".png", ".gif", ".json", ".yml", ".yaml"],
      "size": 5242880,
      "dir": "../public/upload",
      "expire": {
        "types": [".json", ".yml", ".yaml"],
        "day": -1
      }
    },
    "ldap": {
      "server": "ldap://xxxxx:389",
      "bindDN": "cn=AD认证,ou=运维,ou=技术部,ou=xxxxx,dc=office,dc=xxxxx,dc=com",
      "password": "xxxxxxx",
      "filter": {
        "base": "ou=xxxxx,dc=office,dc=xxxx,dc=com",
        "attributeName": "sAMAccountName"
      }
    },
    "fe": {
      "copyright": "",
      "storageNamespace": "easy-mock_",
      "timeout": 25000,
      "publicPath": "/dist/"
    }
  }
```



# deployment启动管理

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: easymock-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: easymock
  template:
    metadata:
      labels:
        app: easymock
    spec:
      containers:
      - name: easymock
        image: easymock/easymock:1.6.0
        command: ["/bin/bash", "-c", "npm start"]
        #- /bin/bash -c "npm start"
        ports:
        - containerPort: 7300
        volumeMounts:
        - name: "production-json"
          mountPath: "/home/easy-mock/easy-mock/config"
      volumes:
      - name: "production-json"
        configMap:
          name: "production-json"
          items:
          - key: production.json
            path: "production.json"
```



# 添加service 集群访问

```yaml
apiVersion: v1
kind: Service
metadata:
  name: easymock-service
  labels:
    name: easymock-service
spec:
  type: ClusterIP
  ports:
  - port: 7300
    targetPort: 7300
  selector:
    app: easymock
```



# 添加ingress 外网访问

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: k8s-easymock
  namespace: easy-mock
  annotations:
    nginx.ingress.kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "http"
spec:
  rules:
  - host: easymock.kube.kxfo.com
    http:
      paths:
      - path: /
        backend:
          serviceName: easymock-service
          servicePort: 7300
```

