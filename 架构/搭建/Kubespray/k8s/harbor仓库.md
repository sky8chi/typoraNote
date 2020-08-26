> https://www.cnblogs.com/wdliu/p/10250385.html

# 安装docker

```shell
#软件包安装
yum install -y yum-utils  
#添加yum源
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
#查看可安装版本
yum list docker-ce --showduplicates | sort -r
#安装最新稳定版本docker-ce
$ sudo yum install docker-ce docker-ce-cli containerd.io
#启动docker
systemctl start docker
#查看docker版本
docker version

systemctl  disable firewalld.service;systemctl  disable postfix.service
sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
reboot
```

# 安装docker-compose

```shell
sudo curl -L "https://github.com/docker/compose/releases/download/1.26.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

yum -y install bash-completion
sudo curl -L https://raw.githubusercontent.com/docker/compose/1.26.2/contrib/completion/bash/docker-compose -o /etc/bash_completion.d/docker-compose


```

# 安装harbor

```shell
# https://github.com/goharbor/harbor/releases 调整版本

yum install -y wget 
wget https://github.com/goharbor/harbor/releases/download/v2.0.2/harbor-offline-installer-v2.0.2.tgz
```



# 生成https证书

```shell
# 1. 创建ca证书
mkdir ca
cd ca
# req：申请证书签署请求；-newkey 新密钥 ；-x509：可以用来显示证书的内容，转换其格式，给CSR签名等X.509证书的管理工作，这里用来自签名。
# 其他的全部回车  Common Name (eg, your name or your server's hostname) []:10.200.120.236
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 365 -out ca.crt

openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=harbor.kxfo.com" \
 -key ca.key \
 -out ca.crt


# 2. 生成服务端请求证书
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout harbor.key -out harbor.csr

openssl genrsa -out harbor.kxfo.com.key 4096
openssl req -sha512 -new \
    -subj "/C=CN/ST=Shanghai/L=Shanghai/O=example/OU=Personal/CN=harbor.kxfo.com" \
    -key harbor.kxfo.com.key \
    -out harbor.kxfo.com.csr

# 3. 生成证书

cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.kxfo.com
DNS.2=harbor.kxfo
DNS.3=harbor
EOF


# 4. 生成远程使用证书
openssl x509 -inform PEM -in harbor.kxfo.com.crt -out harbor.kxfo.com.cert

# 5. 其他docker机器使用
# 下载3个文件
harbor.kxfo.com.cert
harbor.kxfo.com.key
ca.crt
# centos放置到
/etc/docker/certs.d/harbor.kxfo.com
# mac
# 方法1：
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain /你下载放置的目标/ca.crt
# 方法2：
钥匙串导入证书

# 重启docker

```



# 修改配置

```yaml
hostname: 10.200.120.236   # 仓库地址，主机IP或者域名
harbor_admin_password: Harbor12345 # 默认管理员密码
https:
	certificate: /data/install/harbor/ca/harbor.kxfo.com.crt
  private_key: /data/install/harbor/ca/harbor.kxfo.com.key
```

# 访问

```shell
systemctl stop firewalld

#修改并重启
vim /etc/selinux/config
#SELINUX=enforcing
SELINUX=disabled

```

# 项目应用

## pom配置

```xml
    <properties>
        <java.version>1.8</java.version>
        <docker.registry>harbor.kxfo.com</docker.registry>
        <docker.image.prefix>mall</docker.image.prefix>
    </properties>
		<build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>dockerfile-maven-plugin</artifactId>
                <version>1.4.13</version>
                <executions>
                    <execution>
                        <id>default</id>
                        <goals>
                            <!--如果package时不想用docker打包,就注释掉这个goal-->
                            <goal>build</goal>
                            <goal>push</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <!-- harbor 仓库用户名及密码-->
<!--                    <username>admin</username>-->
<!--                    <password>Chtx87_98</password>-->
                    <repository>${docker.registry}/${docker.image.prefix}/${project.artifactId}</repository>
                    <tag>${project.version}</tag>
                    <buildArgs>
                        <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
                    </buildArgs>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

## Dockfile配置

项目要目录下要放置一个Dockerfile， pom文件会引用

```dockerfile
FROM openjdk:8-jdk-alpine
MAINTAINER xxxxxx
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
VOLUME /logs
# pom文件传参覆盖
ARG JAR_FILE=target/demo.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```



## build & push

打包环境必须启动docker

build是在本地打成镜像

push是推送到远程



## 找台docker机器

```shell
docker pull harbor.kxfo.com/mall/demo:1.0.0.0

docker run -it harbor.kxfo.com/mall/demo:1.0.0.0
```

