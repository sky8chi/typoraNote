# 下载

> https://github.com/istio/istio/releases/download/1.8.0/istioctl-1.8.0-linux-amd64.tar.gz

解压复制到/usr/local/bin/

# 命令

```shell
# 显示配置列表
istioctl profile list

# 显示配置具体内容
istioctl profile dump xxxx

# 显示配置子集具体内容
istioctl profile dump --config-path components.pilot

# 比较配置不同
istioctl profile diff demo default

# 安装前生成配置清单；根据情况检查清单执行
istioctl manifest generate > 1.yml
kubectl apply -f 1.yaml

# 检测是否安装成功
istioctl verify-install -f 1.yml

# 定制配置
## 要在默认配置文件中禁用遥测功能
istioctl manifest install --set telemetry.enabled=false
## 运行完整的配置
istioctl manifest install -f samples/pilot-k8s.yaml

# 卸载istio
istioctl manifest generate <your original installation options> | kubectl delete -f -
```



# 安装

> https://istio.io/latest/docs/setup/install/operator/

```shell
# 该命令会创建一个 namespace istio-operator，并将 Istio operator 部署在此 namespace 中
istioctl operator init --tag 1.8.0
kubectl -n istio-operator get pod

istioctl operator init --watchedNamespaces=istio-namespace1,istio-namespace2

kubectl create ns istio-system

kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: istiocontrolplane
spec:
  profile: default
  # 机器内存太小没办法只能减少，否则启动不起来
  components:
    pilot:
      k8s:
        resources:
          requests:
            memory: 1024Mi
EOF

# 等待执行
kubectl get svc -n istio-system

# NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                      AGE
# istio-ingressgateway   LoadBalancer   10.233.39.151   <pending>     15021:30978/TCP,80:32141/TCP,443:30253/TCP,15012:30487/TCP,15443:30557/TCP   2m36s
# istiod                 ClusterIP      10.233.63.213   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP                                        7m37s

kubectl get pods -n istio-system

# NAME                                    READY   STATUS              RESTARTS   AGE
# istio-ingressgateway-74df9684c8-mbtxf   0/1     ContainerCreating   0          8m17s
# istiod-cfcffb65d-9m66g                  0/1     Pending             0          13m
```



# 其他

> https://juejin.cn/post/6844903810393964552
>
> https://my.oschina.net/u/4437985/blog/4424935
>
> https://istio.io/latest/docs/setup/install/operator/
>
> http://www.mydlq.club/article/62/
>
> https://preliminary.istio.io/latest/zh/docs/setup/getting-started/



http://www.mamicode.com/info-detail-2423356.html



kubectl patch service istio-ingressgateway -n istio-system -p '{"spec":{"type":"NodePort"}}'



https://blog.csdn.net/xxhhbb1538/article/details/108779118

kubectl edit daemonset ingress-nginx-controller -n ingress-nginx

https://blog.csdn.net/kozazyh/article/details/81477629?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control





https://blog.csdn.net/qq_37950254/article/details/89603147?utm_medium=distribute.pc_relevant.none-task-blog-title-14&spm=1001.2101.3001.4242



https://blog.csdn.net/sniperking2008/article/details/97647679