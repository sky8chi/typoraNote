> http://www.10qianwan.com/articledetail/191132.html
>
> https://zhuanlan.zhihu.com/p/76328818

# git地址

```
https://github.com/sky8chi/openvpn-install.git
```

# 安装

## 服务端

```
sh openvpn-install.sh
```

## 客户端

```
yum install -y epel-release
yum install -y openvpn
```



# 修改服务端配置

# 默认网关

```shell
# 这条会在client添加一条默认网关全导致server
push "redirect-gateway def1 bypass-dhcp"
```

## 局域网访问

vim /etc/openvpn/server/server.conf

```shell
client-config-dir ccd
client-to-client
push "route 172.16.250.0 255.255.255.0" # 告诉客户端服务端路由
route 10.200.15.0 255.255.255.0  # 告诉服务端客户端路由
```

创建 ccd

```shell
mkdir /etc/openvpn/server/ccd

vim sky8chi  # 此文件名，要和生成的客户端名一致
iroute 10.200.15.0 255.255.255.0

```

# 重启

```shell
systemctl stop openvpn-server@server.service
systemctl start openvpn-server@server.service
```



# 客户端执行

复制服务端生成的/root/sky8chi.ovpn到客户端

```shell
nohup openvpn sky8chi.ovpn &
```

# 局域网访问

## client 局域网内ip 访问server

正常应该在交换机配置，权限有限目前只在局域网机器配置

```shell
# server A
# client B
# client局域网  C
# C要通过B才能访问A
# C添加： ip route add A局域网ip段  via  B
ip route add 172.16.250.0/24 via 10.200.15.45

# 另一种nat（未印证）
-A POSTROUTING -s 10.200.15.0/24 -j SNAT --to-source 10.8.0.1

```



# 不通查找的方向

* iptables -L 查看forward链 是否是全局拒绝策略

* ip forward 是否打开

  ```shell
  /etc/sysctl.conf
  net.ipv4.ip_forward = 1
  sysctl -p
  
  cat /proc/sys/net/ipv4/ip_forward
  ```

* 如果是snat方式

  ```shell
  # 查看postrouting
  iptables -L -t nat
  ```

* 路由追踪

  ```shell
  mtr ip
  ```

* 路由添加不进去

  ```shell
  # 网卡 掩码是32位  会失去本机路由功能，每次请求都会转发网关来分配路由
  ```

  