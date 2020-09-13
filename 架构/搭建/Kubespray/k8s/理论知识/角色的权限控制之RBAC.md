# 基本概念

* ROLE: 角色    一组规则，定义对kubenates api对象的操作权限
* Subjects: 被作用者     即可以是人，也可以是机器，也可以是定义的用户
* RoleBinding: 定义了角色与被作用者关系

# 对namespace权限控制

## Role

```shell
vim role.yaml

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace
  name: example-role
rules:  # 规则 
- apiGroups: [""] 
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

* namespace	指定作用域
* apiGroups  ""代表默认core api
* verbs 权限["get", "list", "watch", "create", "update", "patch", "delete"]

```shell
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
```

resourceNames 指定对名叫"my-config"的ConfigMap对象,有进行GET操作权限.****

## RoleBinding

```shell
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:   #被作用者
- kind: User    # User, Group, ServiceAccount 
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:  # 引用之前定义的role对象，与subject进行绑定
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

# 作用所有namespace或node权限控制

因为是针对所有的，所以上下区别在于不需要指定namespace

## ClusterRole

```shell
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

## ClusterRoleBinding

```shell
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrolebinding
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: example-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

# ServieAccount分配权限的过程

大多数时候,我们其实都不太使用"用户"这个功能,而是直接使用Kubernetes中的"内置用户",而这个"内置用户"就是:ServiceAccount

## 定义命名空间

```shell
vim namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
   name: mynamespace
   labels:
     name: mynamespace
```



## 定义ServiceAccount

```shell
vim svc-account.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: mynamespace
  name: example-sa
```

## 分配权限

```shell
vim role-binding.yaml

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: ServiceAccount
  name: example-sa
  namespace: mynamespace
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

## 创建对象

```shell
 kubectl create -f namespace.yaml
 kubectl create -f svc-account.yaml
 kubectl create -f role.yaml
 kubectl create -f role-binding.yaml
```

## 查看serviceAccount详细信息

```
kubectl get sa -n mynamespace -o yaml

apiVersion: v1
items:
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    creationTimestamp: "2020-07-08T05:03:54Z"
    name: default
    namespace: mynamespace
    resourceVersion: "2820248"
    selfLink: /api/v1/namespaces/mynamespace/serviceaccounts/default
    uid: b0049fc4-5070-4976-bef6-cf1476f762a5
  secrets:
  - name: default-token-g66sk
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    creationTimestamp: "2020-07-08T05:03:43Z"
    name: example-sa
    namespace: mynamespace
    resourceVersion: "2820267"
    selfLink: /api/v1/namespaces/mynamespace/serviceaccounts/example-sa
    uid: c5280008-7a41-401f-870e-ee3c031f1220
  secrets:
  - name: example-sa-token-8jgcs
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

kubernates会为serviceaccount自动分配一个secret对象，与apiserver进行交互的授权文件： token

token内容一般为证书或密码，以secret对象存储在etcd中

## 查看secret

```shell
kubectl get secret example-sa-token-8jgcs -o yaml -n mynamespace
```



## serviceaccount应用

```shell
vim deploy-demon.yaml 

apiVersion: v1
kind: Pod
metadata:
  name: sa-demo
  labels:
    app: myapp
    release: canary
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v2
    ports:
    - name: httpd
      containerPort: 80
  serviceAccountName: admin  #此处指令为指定sa的名称 不指定默认是default
```

* 在 pod 创建之初 service account 就必须已经存在，否则创建将被拒绝。需要注意的是<font color="red">不能更新已创建的 pod 的 service account。</font>

* 默认的service account 仅仅只能获取当前Pod自身的相关属性，无法观察到其他名称空间Pod的相关属性信息。如果想要扩展Pod，假设有一个Pod需要用于管理其他Pod或者是其他资源对象，是无法通过自身的名称空间的serviceaccount进行获取其他Pod的相关属性信息的，此时就需要进行手动创建一个serviceaccount，并在创建Pod时进行定义。
  