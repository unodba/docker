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
# yum update
```
3.执行Docker安装脚本

```shell
# curl -sSL https://get.docker.com/ | sh
```
This script adds the `docker.repo` repository and installs Docker.

4.启动Docker
```shell
# service docker start
```

5.验证docker已经正常安装
```shell
# docker run hello-world
```

### yum安装

1. 使用root权限登陆系统。

2. 更新系统包到最新。
```shell
# yum -y update
```
3. 添加yum仓库
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

4. 安装Docker包
```shell
# yum install -y docker-engine
# yum install -y docker-selinux
# yum list installed | grep docker
```

5. 启动Docker
```shell
# systemctl stop firewalld.service
# systemctl start docker.service
```

6. 验证docker已经正常安装
```shell
$ docker run hello-world
```

## 创建docker组
Docker daemon绑定到Unix socket而不是TCP端口。默认下Unix socket属于root用户拥有并且其他用户可以通过sudo访问。因为这个原因
docker daemon总是使用root用户运行。

为了避免每次使用docker命令都使用sudo，创建一个docker的Unix组并且将用户加入。当docker daemon启动时，使得Unix socket对docker组是可读写的。

警告：docker组等同于root用户；关于这会如何影响你系统的安全性，详情参考
[Docker Daemon Attack Surface](https://docs.docker.com/articles/security/#docker-daemon-attack-surface) .

1. 使用超户权限登陆

2. 创建docker用户组并添加用户
```shell
$ sudo usermod -aG docker your_username
```
3. 登出并重新登陆
这保证你的用户正确的权限运行。

4. 不使用sudo运行docker验证正常
```shell
$ docker run hello-world
```
## 开机自启动

配置Docker开机自启动：
```shell
  $ sudo chkconfig docker on
```
If you need to add an HTTP Proxy, set a different directory or partition for the Docker runtime files, or make other customizations, read our Systemd article to learn how to customize your Systemd Docker daemon options.

## 卸载

使用yum卸载Docker。

1. 列出安装的软件包
```shell
$ yum list installed | grep docker
yum list installed | grep docker
docker-engine.x86_64                1.7.1-1.el7
```
2. 移除软件包
```shell
$ sudo yum -y remove docker-engine.x86_64
```
上面的命令不会删除镜像，容器，卷组和用户自配置文件。

3. 删除所有镜像，容器和卷组
```shell
$ rm -rf /var/lib/docker
```
4. 删除用户自配置文件
