# 亲和性/反亲和性调度策略

| 调度策略        | 匹配标签 | 操作符                                  | 拓扑域支持 | 调度目标                         |
| --------------- | -------- | --------------------------------------- | ---------- | -------------------------------- |
| nodeAffinity    | 主机     | In、NotIn、Exists、DoesNotExist、Gt、Lt | 否         | 指定主机                         |
| podAffinity     | Pod      | In、NotIn、Exists、DoesNotExist         | 是         | Pod 与指定 Pod 在同一个拓扑域    |
| podAnitAffinity | Pod      | In、NotIn、Exists、DoesNotExist         | 是         | Pod 与 指定 Pod 不在同一个拓扑域 |

# 键值运算关系 

affinity.nodeAffinity.nodeSelectorTerms.operator

* In：label 的值在某个列表中

* NotIn：label的值不在某个列表中

* Gt：label的值大于某个值
* Lt：label的值小于某个值

* Exists：某个 label 存在

* DoesNotExist：某个 label 不存在



# 节点亲和性策略

pod.spec.nodeAffinity

## 硬策略

requiredDuringSchedulingIgnoredDuringExecution：

## 软策略

preferredDuringSchedulingIgnoredDuringExecution


# topologyKey

可以设置node上的label的值来表示node的name,zone,region等信息，pod的规则中指定topologykey的值表示指定topology范围内的node上运行的pod满足指定规则

* kubernetes.io/hostname　　＃Node

* failure-domain.beta.kubernetes.io/zone　＃Zone
* failure-domain.beta.kubernetes.io/region   #Region

# 案例

首先通过亲和性来指定节点分布状况，通过策略条件来做限制

## 标签管理

```shell
# 查看标签
kubectl get pod --show-labels

# 修改标签  node1 -> node2
kubectl label pod node1 app=node2 --overwrite

#语法
kubectl label nodes <node-name> <label-key>=<label-value>

#示例，如果节点名称是harmonycloud,需要添加‘disktype=ssd’标签
kubectl lable nodes harmonycloud disktype=ssd

# 验证，使用如下命令查看是否成功给node添加标签
kubectl get nodes --show-labels
```



# 硬策略

```yaml
# 选择 kubernetes.io/hostname != k8s-node2 的节点进行匹配。
# 指定节点永远不会在k8s-node2这个节点上运行

apiVersion: v1	
kind: Pod
metadata: 
  name: affinity
  labels: 
    app: node-affinity-pod
spec:	
 containers:
   - name: nginx
     image: nginx:1.9.1
 affinity: 	#亲和性
   nodeAffinity: #node节点亲和性
     requiredDuringSchedulingIgnoredDuringExecution:  	#硬策略限制
       nodeSelectorTerms: 	#node选择方案
         - matchExpressions: 	
           - key: kubernetes.io/hostname	#键名是 kubernetes.io/hostname
             operator: NotIn	# 运算关系: 不在，不是
             values: 			
               - k8s-node2		# key的值，不是k8s-node2即可。
```

```yaml
# 通过标签来运行 选择策略是podAffinity， 必须在一个节点运行
# 如果这个发现这个节点上已经有标签是app=node1的pod在运行，则归到同一节点运行

apiVersion: v1
kind: Pod
metadata: 
  name: pod-3
  labels: 
    app: pod-3
spec: 
  containers: 
    - name: pod-3
      image: nginx:1.9.1
  affinity:			#亲和性
    podAffinity: 	#Pod亲和性,运行在同一个 Pod
      requiredDuringSchedulingIgnoredDuringExecution: 	#硬限制策略
        - labelSelector: 		#标签匹配
             matchExpressions:  #匹配规则，如果有 app=node1，则运行在该Pod
               - key: app		#当app的值
                 operator: In 	#是
                 values: 
                   - node1		#是node1，则进行匹配
          topologyKey: kubernetes.io/hostname	#通过hostname标签进行判断
```



## 软策略

```yaml
# 优先匹配，匹配不到不强求，选择其他节点
# 先匹配 kubernetes.io/hostname=k8s-node3的节点， 如果木有这个节点，则在其他节点运行

apiVersion: v1	
kind: Pod
metadata: 
  name: affinity
  labels: 
    app: node-affinity-pod
spec:	
 containers:
   - name: nginx
     image: nginx:1.9.1
 affinity: 				#亲和性
   nodeAffinity:		#node节点亲和性
     preferredDuringSchedulingIgnoredDuringExecution:  	#软策略限制
       - weight: 1		#权重为1，如果有多个不同的软策略，则根据权重高的来进行匹配
         preference: 	
           - matchExpressions: 	#匹配规则
             - key: kubernetes.io/hostname	#键名是 kubernetes.io/hostname=
               operator: In		# 运算关系: 是，在
               values: 			
                - k8s-node3		# key的值，最好是k8s-node3，如果没有node3则在其他节点运行。
```



```yaml
# 选择亲和是PodAntiAffinity， 必须不在一个节点
# 如果当前机器有app=node2标签的pod在运行，则不能在这个节点上再次运行此pod
apiVersion: v1
kind: Pod
metadata: 
  name: pod-3
  labels: 
    app: pod-3
spec: 
  containers: 
    - name: pod-3
      image: nginx:1.9.1
  affinity:	
    PodAntiAffinity: #Pod亲和性，必须不在同一个节点
      preferredDuringSchedulingIgnoreDuringExectuion: 	#软策略
        - weight: 1				#权重
          podAffinityTerm: 		
            labelSelector: 
              matchExpressions: #如果有 app=node2 则不运行在该 node2 节点运行。
                - key: app
                  operator: In
                  values: 
                    - node2
            topologykey: kubernetes.io/hostname 
```



## 软硬兼施策略

```yaml
# 硬条件永远优于软条件
# 节点不能运行在 k8s-node2 节点上， 选择其他节点时，系统最好是linux，不强求

apiVersion: v1	
kind: Pod
metadata: 
  name: affinity
  labels: 
    app: node-affinity-pod
spec:	
 containers:
   - name: nginx
     image: nginx:1.9.1
 affinity: 					#亲和性
   nodeAffinity: 			#node节点亲和性
     requiredDuringSchedulingIgnoredDuringExecution:  	#硬策略限制
       nodeSelectorTerms: 	#node选择方案
         - matchExpressions: 	
           - key: kubernetes.io/hostname	#键名是 kubernetes.io/hostname=
             operator: NotIn	# 运算关系: 不在，不是
             values: 			
               - k8s-node2		# key的值，不是k8s-node2即可。
     preferredDuringSchedulingIgnoredDuringExecution: 	#软策略限制
       - weight: 1			#权重1
         preference: 		
           matchExpressions: 	#匹配规则
             - key: kubernetes.io/os	# 键名kubernetes.io/os
               operator: In				# 是，在
               values: 					
                - linux					# 匹配 os 是 linux
```



