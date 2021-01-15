# 开启dashboard外网端口

# NodePort

```shell
# 查看是否安装 
kubectl describe services kubernetes-dashboard --namespace=kube-system

# 编辑 NodePort方式访问
kubectl edit service -n kube-system kubernetes-dashboard
spec:
  clusterIP: 10.233.60.151
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30443   #增加端口
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort  #修改默认的type  ClusterIp 为NodePort
status
# 查看是否修改成功
kubectl get svc -n kube-system kubernetes-dashboard
```

> https://www.cnblogs.com/hixiaowei/p/9992691.html

```shell
# 获取 命名空间 kube-system 所有安全信息
kubectl get secrets -n kube-system
kubectl get secret kubernetes-dashboard-certs -n kube-system -o yaml
```

```
kubectl delete secret kubernetes-dashboard-certs -n kube-system
kubectl create secret generic kubernetes-dashboard-certs --from-file=/etc/kubernetes/pki/apiserver.key --from-file=/etc/kubernetes/pki/apiserver.crt -n kube-system

# 查找并删除dashboard pod 自动重启
kubectl get pod -n kube-system  | grep dashboard
kubectl delete pod kubernetes-dashboard-7dbcd59666-r2r48 -n kube-system
```



此方式有证书问题未调通，firefox可以跳过

## proxy

```shell
# master ip 10.200.120.240 执行
kubectl proxy --address=10.200.120.240 --accept-hosts='^*$' &

http://10.200.120.240:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default
```



```shell
# 查看dashboard token
kubectl  -n kube-system get secret | grep dashboard
kubernetes-dashboard-certs                       Opaque                                0      19h
kubernetes-dashboard-csrf                        Opaque                                1      19h
kubernetes-dashboard-key-holder                  Opaque                                2      19h
kubernetes-dashboard-token-8vjzc                 kubernetes.io/service-account-token   3      19h

#查看token
kubectl -n kube-system describe secret kubernetes-dashboard-token-8vjzc
```

# ingress

## 授权账号

```shell
vim dashboard-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard
  namespace: kube-system

---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dashboard
subjects:
  - kind: ServiceAccount
    name: dashboard
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

## 应用账号

```shell
kubectl create -f dashboard-rbac.yaml
```

## 获取token

```shell
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep dashboard-token | awk '{print $1}')

```

## ingress 暴露

```shell
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout ./tls.key -out ./tls.crt -subj "/CN=dashboard.kube.kxfo.com"
kubectl -n kube-system create secret tls k8s-dashboard-secret --key ./tls.key --cert ./tls.crt
```

```shell
vim ingress.yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: k8s-dashboard
  namespace: kube-system
  annotations:
    nginx.ingress.kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
  - hosts:
    - dashboard.kube.kxfo.com
    secretName: k8s-dashboard-secret
  rules:
  - host: dashboard.kube.kxfo.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```

```
kubectl apply -n kube-system -f ingress.yml
```

