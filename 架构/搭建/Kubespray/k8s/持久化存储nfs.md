> https://www.cnblogs.com/panwenbin-logs/p/12196286.html
>
> https://blog.csdn.net/dkfajsldfsdfsd/article/details/81319735

```shell
#service端：nfs安装配置完成后，检测配置是否正确
exportfs -a
# client端验证
showmount -e <NFS server name>
```



# 创建命名空间

```shell
vim namespace.yml

---
apiVersion: v1
kind: Namespace
metadata:
  name: mongo
  labels:
    name: mongo
```



# 创建权限

```shell
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: mongo

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-client-provisioner-cluster-role
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nfs-client-provisioner-cluster-role-binding
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: mongo
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-cluster-role
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: nfs-client-provisioner-role
  namespace: mongo
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nfs-client-provisioner-role-binding
  namespace: mongo
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: mongo
roleRef:
  kind: Role
  name: nfs-client-provisioner-role
  apiGroup: rbac.authorization.k8s.io

```

# 启动nfs provisioner

# 启动nfs provisioner

```shell
vim nfs-provisioner.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  namespace:
    mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes   # 此处不能更改，这是镜像默认生成pv地址，当时改了这块，死活不生效，pvc pv又是创建成功，查了好久
          env:
            - name: PROVISIONER_NAME
              value: qgg-nfs-storage
            - name: NFS_SERVER
              value: 10.200.120.240
            - name: NFS_PATH
              value: /data/nfs
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.200.120.240
            path: /data/nfs
```



```shell
# 获取对应的pod
kubectl get pod | grep nfs-client-provisioner
nfs-client-provisioner-68b78f9b59-jvkgq   1/1     Running     0          31m
# 进pod执行
kubectl exec -it nfs-client-provisioner-68b78f9b59-jvkgq -- df
10.200.120.240:/data/nfs 29624320   5813248  23811072  20% /persistentvolumes
```



# 创建storage class

```shell
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
  namespace: mongo
provisioner: qgg-nfs-storage # 此处要与上面deployment env中 PROVISIONER_NAME 保持一致 
parameters:
  archiveOnDelete: "false"
```



# 创建pvc

```shell
vim test-claim.yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  namespace: mongo
  #annotations:
  #  volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  storageClassName: managed-nfs-storage   # 以前是annotations的方式，现在是这种方式，貌似以前的还可以使用
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```



```shell
# 查看pvc
kubectl get pvc

test-claim   Bound    pvc-75142613-c16e-419f-8b4e-61286e1ec13a   1Mi        RWX            managed-nfs-storage   32m

# 查看自动生成的pv, 取上面生成的volume查询  
kubectl get pv | grep pvc-75142613-c16e-419f-8b4e-61286e1ec13a

pvc-75142613-c16e-419f-8b4e-61286e1ec13a   1Mi        RWX            Delete           Bound    mongo/test-claim   managed-nfs-storage            31m

# 查看pv目录是否生成 ： [命名空间]-[pvc名称]-pv实例
find /data/nfs/ -name *pvc-75142613-c16e-419f-8b4e-61286e1ec13a

/data/nfs/mongo-test-claim-pvc-75142613-c16e-419f-8b4e-61286e1ec13a
```



# 写个pod测试

```shell
vim test-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-pod
      image: busybox
      command:
        - "/bin/sh"
      args:
        - "-c"
        - "touch /mnt/SUCCESS && exit 0 || exit 1"
      volumeMounts:
        - name: nfs-pvc
          mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim
```



```shell
# 查看pod是否执行成功
kubectl get pod test-pod
test-pod                                  0/1     Completed   0          13m
# 查看是否生成SUCCESS
ll /data/nfs/mongo-test-claim-pvc-75142613-c16e-419f-8b4e-61286e1ec13a/
```



