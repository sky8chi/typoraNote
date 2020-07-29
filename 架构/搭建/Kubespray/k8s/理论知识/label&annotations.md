# label 与selector

## label

标签其实就一对 key/value，被关联到k8s对象上，比如pod/node等。标签可以用来划分特定组的对象,用户可通过标签来组织pod和所有其他Kubemetes对象。

```
metadata:
  labels:
    app: pc
```



## selector

```
 spec:
   selector:
     app: pc
```

像service之类的根据selector把相同的label合并到一块，可以多label



# annotations

网上文章只说是注解，用来搜索看的，不过貌似不少应用会以他做配置使用