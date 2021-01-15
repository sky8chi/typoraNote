# 安装

## 使用二机制包在macOS安装

```bash
1、下载软件（最新版本）
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s  

2、赋予执行权限
chmod +x ./kubectl
3、将其拷贝到已经添加到环境变量的某个目录中 方便后期执行
sudo mv ./kubectl /usr/local/bin/kubectl
4、执行命令测试
kubectl version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:36:53Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"14+", GitVersion:"v1.14.6-aliyun.1", GitCommit:"a4182a8", GitTreeState:"", BuildDate:"2019-08-27T06:03:13Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

## 使用二进制包在CentOS 7.x上面安装

```bash
1、下载软件

curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s  
指定版本
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.13.5/bin/linux/amd64/kubectl 
如果您要下载最新版本的安装包，使用如下命令即可： 仅需将v1.13.5替换为$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)即可

2、赋予执行权限
chmod +x ./kubectl
3、将其拷贝到已经添加到环境变量的某个目录中 方便后期执行
sudo mv ./kubectl /usr/local/bin/kubectl
4、测试
[root@izuf6beunt2a2ki2nqvphuz ~]# kubectl  version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:36:53Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"14+", GitVersion:"v1.14.6-aliyun.1", GitCommit:"a4182a8", GitTreeState:"", BuildDate:"2019-08-27T06:03:13Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

# 替换配置

```
下载服务器端的~/.kube/config到本地，替换本地的config文件
```

