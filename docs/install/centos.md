Docker支持运行在以下CentOS版本：

* CentOS 7.X

安装在二进制兼容的EL7版本如 Scientific Linux也是可能成功的，但是Docker
没有测试过并且不官方支持。

此文带你通过使用Docker管理的发行包和安装机制来安装。使用这些报能确保你使用最新的Docker版本。
如果你希望使用CentOS管理的包，请阅读你的CentOS文档。

## 要求

不过你的系统版本是多少，Docker都要求64位。并且当CentOS7时你的内核必须不小于3.10。

检查当前内核版本：
```shell
# uname -r
3.10.0-229.el7.x86_64
```
建议将系统升级到最新。

## 安装
有两种方式可安装Docker Engine。脚本安装和yum安装。

### 脚本安装
1.使用root权限登陆系统。

2.更新系统包到最新。
```shell
# yum -y update
```

3.执行Docker安装脚本
```shell
# curl -sSL https://get.docker.com/ | sh
# yum install -y docker-selinux
```
这个脚本会添加`docker.repo` 配置并安装Docker。

4.启动Docker
```shell
# systemctl start docker.service
```

5.验证docker已经正常安装
```shell
# docker run hello-world
```

### yum安装

1.使用root权限登陆系统。

2.更新系统包到最新。
```shell
# yum -y update
```

3.添加yum仓库
```shell
# cat >/etc/yum.repos.d/docker.repo <<-EOF
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```

4.安装Docker包
```shell
# yum install -y docker-engine
# yum install -y docker-selinux
# yum list installed | grep docker
docker-engine.x86_64             1.8.1-1.el7.centos                    @dockerrepo
docker-selinux.x86_64            1.7.1-108.el7.centos                  @extras  
```
这里有个非常坑的情况，官方文档没有提到docker-selinux的安装，笔者在使用VirtualBox，配置一个桥接，一个Host-Only的网卡时，只安装docker-engine启动会报错，需要在安装docker-selinux方可。
可以看github上的两个issues，[1.8.0: Systemd can't start docker on Centos 7.1 #15498](https://github.com/docker/docker/issues/15498),[Docker start times out if firewalld is started #13019](https://github.com/docker/docker/issues/13019)。

5.启动Docker
```shell
# systemctl start docker.service
```

6.验证docker已经正常安装
```shell
# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
535020c3e8ad: Pull complete
af340544ed62: Already exists
library/hello-world:latest: The image you are pulling has been verified. Important: image verification is a tech preview feature and should not be relied on to provide security.
Digest: sha256:d5fbd996e6562438f7ea5389d7da867fe58e04d581810e230df4cc073271ea52
Status: Downloaded newer image for hello-world:latest

Hello from Docker.
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker Hub account:
 https://hub.docker.com

For more examples and ideas, visit:
 https://docs.docker.com/userguide/
```

## 创建docker组
Docker daemon绑定到Unix socket而不是TCP端口。默认下Unix socket属于root用户拥有并且其他用户可以通过sudo访问。因为这个原因
docker daemon总是使用root用户运行。

为了避免每次使用docker命令都使用sudo，创建一个docker的Unix组并且将用户加入。当docker daemon启动时，使得Unix socket对docker组是可读写的。

警告：docker组等同于root用户；关于这会如何影响你系统的安全性，详情参考
[Docker Daemon Attack Surface](https://docs.docker.com/articles/security/#docker-daemon-attack-surface) .

1.使用超户权限登陆并添加docker用户
```shell
# useradd -g docker docker
# echo "docker" | passwd --stdin docker
```

2.创建docker用户组
```shell
# usermod -aG docker your-user
```
your-user指预使用的普通用户，第一步如果创建docker用户时需要制定属于docker组，不然会再次创建docker，无法创建docker用户。

3.登出并使用docker重新登陆
这保证你的用户正确的权限运行。

4.不使用sudo运行docker验证正常
```shell
$ docker run hello-world
```
## 开机自启动

配置Docker开机自启动：
```shell
  #  systemctl enable docker.service
```
If you need to add an HTTP Proxy, set a different directory or partition for the Docker runtime files, or make other customizations, read our Systemd article to learn how to customize your Systemd Docker daemon options.

## 卸载

使用yum卸载Docker。

1.列出安装的软件包
```shell
$ yum list installed | grep docker
yum list installed | grep docker
docker-engine.x86_64             1.8.1-1.el7.centos                    @dockerrepo
docker-selinux.x86_64            1.7.1-108.el7.centos                  @extras  
```

2.移除软件包
```shell
$ sudo yum -y remove docker-engine.x86_64
```
上面的命令不会删除镜像，容器，卷组和用户自配置文件。

3.删除所有镜像，容器和卷组
```shell
$ rm -rf /var/lib/docker
```

4.删除用户自配置文件
