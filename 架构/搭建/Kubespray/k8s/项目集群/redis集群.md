> 
>
> https://github.com/zuxqoj/kubernetes-redis-cluster/blob/master/statefulset/redis-statefulset.yaml
>
> https://blog.csdn.net/tianyouyexin/article/details/95981408

# 创建namespace

```shell
vim namespace.yml

---
apiVersion: v1
kind: Namespace
metadata:
  name: redis
  labels:
    name: redis
```



# 切换redis namespace空间

```shell
kubectl config set-context --current --namespace=redis
```



# redis配置

```shell
vim redis.conf

appendonly yes
cluster-enabled yes
cluster-config-file /var/lib/redis/nodes.conf
cluster-node-timeout 5000
dir /var/lib/redis
port 6379

# 创建 cm
kubectl create configmap redis-conf --from-file=./redis.conf 

# 查看 cm
kubectl describe cm redis-conf
```



# 创建headlessService

```shell
vim headless.yml

apiVersion: v1
kind: Service
metadata:
  name: redis-service
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    port: 6379
  clusterIP: None
  selector:
    app: redis
    appCluster: redis-cluster
```



# 创建statefulset

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-app
spec:
  serviceName: "redis-service"
  replicas: 6
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        appCluster: redis-cluster
    spec:
      terminationGracePeriodSeconds: 20
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
      containers:
      - name: redis
        image: "redis:3.2.8"
        command:
          - "redis-server"
        args:
          - "/etc/redis/redis.conf"
          - "--protected-mode"
          - "no"
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
        ports:
            - name: redis
              containerPort: 6379
              protocol: "TCP"
            - name: cluster
              containerPort: 16379
              protocol: "TCP"
        volumeMounts:
          - name: "redis-conf"
            mountPath: "/etc/redis"
          - name: "redis-data"
            mountPath: "/var/lib/redis"
      volumes:
      - name: "redis-conf"
        configMap:
          name: "redis-conf"
          items:
            - key: "redis.conf"
              path: "redis.conf"
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteMany" ]
      storageClassName: managed-nfs-storage
      resources:
        requests:
          storage: 1Gi              
```

# 初始化集群

```shell
# 启动一个Ubuntu的容器，可以在该容器中安装Redis-tribe，进而初始化Redis集群
kubectl run -it ubuntu --image=ubuntu --restart=Never /bin/bash

# 更改apt源
cat > /etc/apt/sources.list << EOF
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
EOF

# 准备环境
apt-get update
apt-get install -y vim wget python2.7 python-pip redis-tools dnsutils

# 安装 redis-trib
pip install redis-trib==0.5.1

# 创建master节点
redis-trib.py create \
  `dig +short redis-app-0.redis-service.redis.svc.cluster.local`:6379 \
  `dig +short redis-app-1.redis-service.redis.svc.cluster.local`:6379 \
  `dig +short redis-app-2.redis-service.redis.svc.cluster.local`:6379


# 为master添加slaver
redis-trib.py replicate \
  --master-addr `dig +short redis-app-0.redis-service.redis.svc.cluster.local`:6379 \
  --slave-addr `dig +short redis-app-3.redis-service.redis.svc.cluster.local`:6379

redis-trib.py replicate \
  --master-addr `dig +short redis-app-1.redis-service.redis.svc.cluster.local`:6379 \
  --slave-addr `dig +short redis-app-4.redis-service.redis.svc.cluster.local`:6379

redis-trib.py replicate \
  --master-addr `dig +short redis-app-2.redis-service.redis.svc.cluster.local`:6379 \
  --slave-addr `dig +short redis-app-5.redis-service.redis.svc.cluster.local`:6379
```

# 连到任意一个Redis Pod中检验一下：

```shell
# 进入pod查看
kubectl exec -it redis-app-2 -- /bin/bash
# 然后进入redis 
/usr/local/bin/redis-cli -c
# 执行cluster nodes 和cluster info 查看信息
```

# 测试主从

```shell
# 进入redis-app-0查看
kubectl exec -it redis-app-0 /bin/bash

/usr/local/bin/redis-cli -c
role
# 查看到app-0为master

# 手动删除redis-app-0：
kubectl delete pod redis-app-0

# 再进入redis-app-0内部查看：
kubectl exec -it redis-app-0 /bin/bash

/usr/local/bin/redis-cli -c
role
# 查看redis-app-0变成了slave
```



# 暴露外网

```shell
# 此方法能连，因为返回具体的内部slot ip，没有开放访问，所以基本无用
vim /etc/kubernetes/addons/ingress_nginx/cm-tcp-services.yml

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
data:
  6379: "redis/redis-service:6379"
```



# 错误集合

```shell
# (error) MOVED 6918 10.233.90.15:6379  redis-cli必须加-c

redis-cli -c

# -> Redirected to slot [6918] located at 10.233.90.15:6379
# Could not connect to Redis at 10.233.90.15:6379: Connection timed out

这个没有找到方法 redis计算slot返回客户端内部ip，又无法一个个暴露动态ip，端口， 目前无解，只能内部访问
```



# 单实例部署

```shell
vim redis-easymock.conf
requirepass 123
bind 0.0.0.0
save 900 1
save 300 10
save 60 10000

kubectl create configmap redis-easymock --from-file=./redis-easymock.conf
```



```yaml
vim redis-easymock-dp.yml

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: easymock-claim
spec:
  storageClassName: managed-nfs-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-easymock
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-easymock
  template:
    metadata:
      labels:
        app: redis-easymock
    spec:
      containers:
      - name: redis
        image: "redis:3.2.8"
        command:
          - "redis-server"
        args:
          - "/etc/redis/redis.conf"
          - "--protected-mode"
          - "no"
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
        ports:
          - name: redis
            containerPort: 6379
            protocol: "TCP"
        volumeMounts:
          - name: "redis-easymock"
            mountPath: "/etc/redis"
          - name: "redis-easymock-data"
            mountPath: "/var/lib/redis"
      volumes:
      - name: "redis-easymock"
        configMap:
          name: "redis-easymock"
          items:
            - key: "redis-easymock.conf"
              path: "redis.conf"
      - name: "redis-easymock-data"
        persistentVolumeClaim:
          claimName: easymock-claim

---

apiVersion: v1
kind: Service
metadata:
  name: redis-easymock-svc
  namespace: redis
  labels:
    app: redis-easymock-svc
spec:
  selector:
    app: redis-easymock
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
```



# 集群ip更新(参考)

## 断电后集群重启ip变了

node.conf里还是原ip, 导致cluster一直是down状态

网上说指定cluster-announce-ip，试了下无效，可能是针对单台挂掉的更新

```yaml
containers:
      - name: redis
        image: redis:4.0.6
        command: ["redis-server"]
        args:
        - /etc/redis/redis.conf
        - --cluster-announce-ip
        - "$(MY_POD_IP)"
        env:
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
```

实际我是通过删除data文件操作相当于重新部署个新的集群了

```
删除 
rm -rf /var/lib/redis
或者删除对应的pvc
重新部署
```

