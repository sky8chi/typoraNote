> http://www.10qianwan.com/articledetail/191132.html
>
> https://zhuanlan.zhihu.com/p/76328818

# git地址

```
https://github.com/sky8chi/openvpn-install.git
```

# 安装

```
sh openvpn-install.sh
```

# 修改服务端配置

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

