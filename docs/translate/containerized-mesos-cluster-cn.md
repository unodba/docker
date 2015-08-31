We often use [Apache Mesos](http://mesos.apache.org/) clusters to run containerized applications for our clients and for ourselves. Even though we run applications in containers, we still install Mesos with its dependencies and frameworks on host machines from standard [Debian packages](https://mesosphere.com/downloads/). Although that’s what most Mesos users do and it’s a straightforward and safe way of installing Mesos, we thought we might stretch the Mesos installation a little bit and try to run this stuff in containers too.

我们经常为我们的客户和自己用[Apache Mesos](http://mesos.apache.org/) 集群运行容器化的应用。尽管
我们在容器中运行应用，我们还是从标准的[Debian packages](https://mesosphere.com/downloads/)安装Mesos的相关
依赖和框架。虽说这是很多Mesos用户的做法而且是Mesos安装最直接、保险的方式，我们想可以略微延伸一点Mesos的安装方式并尝试在容器中运行。

Apart from this being a lot of fun and educational, this approach helped us to solve one very practical issue. We are now able to run arbitrary versions of Mesos and its frameworks, including the release candidate versions, because the problem shrank to running different versions of the Docker images in our setup.

除了有很大的乐趣和教育意义外，这种方式会帮助我们解决一个非常实际的问题。现在可以运行任意版本的Mesos和它的框架，
包括发布的候选版本，因为在我们的设置下这个问题变成运行不同版本的Docker镜像。

Programming against the most recent versions of Mesos (and other tools) is important for us when we develop [our own Mesos frameworks](https://github.com/mesos/elasticsearch). Additionally, this gives us more time to apply security fixes, sometimes before they are even released.

当我们进行开发时，能使用不同版本的Mesos（和其它工具）是非常重要的。同时，这给我们更多时间进行安全问题修复，有时甚至赶在它们发布之前。

# Mesos, Marathon and Zookeeper Docker Images

# Mesos, Marathon 和 Zookeeper Docker 镜像

There are a few different [Docker images for Apache Mesos](https://registry.hub.docker.com/search?q=mesos) available on the Docker Hub at the moment (note that Mesos master and slave are two different images). We have chosen the images from [Mesosphere](https://registry.hub.docker.com/repos/mesosphere/), who are a significant contributor to the ecosystem (and our friends). [Redjack](https://registry.hub.docker.com/repos/redjack/)’s repository also contains quality Mesos images with comprehensive documentation. It’s up to you which one you use, both worked just fine for us, but be aware that there might be small differences in the configuration, as this article is written for Mesosphere images.

目前在Docker Hub上有几个不同的Apache Mesos Docker镜像可用 (Mesos master 和 slave是两个镜像)。我们选择[Mesosphere](https://registry.hub.docker.com/repos/mesosphere/)的镜像，(译者注：https://hub.docker.com/r/mesosphere/mesos/，原文地址不可用)一个卓越的生态系统贡献者也是我们的朋友。[Redjack](https://registry.hub.docker.com/repos/redjack/)的仓库同样提供高质量、文档丰富的Mesos镜像。使用哪个取决于你，我们使用的每个都工作良好，不过需要当心配置可能会有些不同，因为这篇文章是为Mesosphere镜像而写。

[Thijs Schnitger](https://github.com/thijsschnitger) created a [Docker image for Zookeeper 3.5](https://registry.hub.docker.com/u/containersol/zookeeper/) last week. This version of Apache Zookeeper introduces dynamic hosts reconfiguration, which is a very cool feature that is especially useful in a dynamic environment, which a cluster might be. So we use Container Solutions’ Zookeeper image in this setup.

上周Thijs Schnitger创建了一个Zookeeper3.5的镜像。Apache Zookeeper在这个版本引进了动态主机重配置，一个非常酷的特性，特别是在一个经常变化的环境中特别有用，集群也是。所以在设置过程中我们使用Zookeeper容器解决方案

We also use the [Marathon Docker image from Mesosphere](https://registry.hub.docker.com/u/mesosphere/marathon/).

我们也会用到[Marathon Docker image from Mesosphere](https://registry.hub.docker.com/u/mesosphere/marathon/).

# Containerized Cluster Configuration

# 容器化集群配置

We use [Terraform](https://terraform.io/) to launch some of our clusters. Refer to [terraform-mesos](https://github.com/ContainerSolutions/terraform-mesos) GitHub repository and check out its [containers](https://github.com/ContainerSolutions/terraform-mesos/tree/containers) branch, where you can find a working solution (at least at the moment of writing). Later on, it will likely be merged to the master branch.

我们使用Terraform启动集群。参考GitHub repository[terraform-mesos](https://github.com/ContainerSolutions/terraform-mesos)并check out 它的[容器](https://github.com/ContainerSolutions/terraform-mesos/tree/containers) 分支，你会找到一个解决方案（至少在写这篇文章时）。不久，它会被合并到master分支。

Let’s take a look on the interesting parts of the setup, the docker run commands for Mesos, Marathon and Zookeeper.

让我们来看一下配置过程的有趣部分，docker为Mesos，Marathon和Zookeeper运行命令。

## Mesos Master

## Mesos 主节点

```shell
QUORUM=2 # number of masters divided by 2 plus 1
CLUSTERNAME="cluster7"
ZK="zk://<quorum_string>/mesos"
MESOS_VERSION="0.22.1-1.0.ubuntu1404"

docker run -d \
-e MESOS_QUORUM=${QUORUM} \
-e MESOS_WORK_DIR=/var/lib/mesos \
-e MESOS_LOG_DIR=/var/log \
-e MESOS_CLUSTER=${CLUSTERNAME} \
-e MESOS_ZK=${ZK}/mesos \
--net="host" \
redjack/mesos-master:${MESOSVERSION}
--mesosphere/mesos-master:${MESOSVERSION}
```

As you can see, we simply pass a reference to a version tag of the image to get a specific version of Mesos running. Aside from that, we need to enable [host networking](https://docs.docker.com/articles/networking/) for Mesos to work properly and pass several environment variables to the container. As a convention, the `MESOS_` prefixed variables hold the configuration values, we’d normally write to the config files in `/etc/mesos*` directories on the host.

如你所见，我们仅仅传递给镜像几个相关的版本tag就能运行制定版本的Mesos。除此之外，为了Mesos正常工作并且传递几个环境变量给容器，需要启用host networking。简而言之，MESOS_`前缀变量hold配置的值，一般我们会写到主机
`/etc/mesos*`目录下的配置文件。

We run a master container on each host in the cluster.

在集群中的每台主机上运行主节点容器。

## Mesos Slave

## Mesos从节点

```shell
ZK="zk://<quorum_string>/mesos"
HOSTNAME="host1" # use a hostname of the host
IP="10.20.30.2" # use an IP address of the host
MESOSVERSION="0.22.1-1.0.ubuntu1404"

docker run -d \
 -e MESOS_LOG_DIR=/var/log/mesos \
 -e MESOS_MASTER=${ZK} \
 -e MESOS_EXECUTOR_REGISTRATION_TIMEOUT=5mins \
 -e MESOS_HOSTNAME=${HOSTNAME} \
 -e MESOS_ISOLATOR=cgroups/cpu,cgroups/mem \
 -e MESOS_CONTAINERIZERS=docker,mesos \
 -e MESOS_PORT=5051 \
 -e MESOS_IP=${IP} \
 -v /var/run/docker.sock:/var/run/docker.sock \
 -v /usr/bin/docker:/usr/bin/docker \
 -v /sys:/sys:ro \
 --net="host" \
redjack/mesos-slave
 --mesosphere/mesos-slave:${MESOSVERSION}
 ```

 For the Mesos slave, the configuration is a bit more complex, mainly because we need to make sure the slave can access the Docker daemon running on the host (the same daemon that’s used to run this Mesos slave instance). We do it by mounting `/var/run/docker.sock` file, `/usr/bin/docker` binary and the `/sys`  directory (read-only) to the Mesos slave container. Please note, that this is not a bulletproof solution and might need to be further tweaked, e.g. when running another Mesos framework by Marathon.

对Mesos从节点，配置有点复杂，主要因为需要确保从节点可以访问主机上的Docker后台进程（同样的后台进程用来运行这个Mesos从节点实例）。通过挂载`/var/run/docker.sock`文件，`/usr/bin/docker`二进制和`/sys` 目录
（只读）到Mesos从节点容器实现。请注意，那还不是一个完美的解决方案并且可能会被进一步拧巴，如。当从Marathon运行另一个Mesos框架。

 ## Zookeeper

 ## Zookeeper

 ```shell
# first node

docker run -d -p 2181:2181 containersol/zookeeper 1

# other nodes
MYID="2" # 3,4,5,...
FIRST_NODE="cluster7-mesos-master-0" # hostname of the first node

docker run -d -p 2181:2181 containersol/zookeeper ${MYID} ${FIRST_NODE}
--需要更改端口2888:2888，3888:3888
```
Running containerized Zookeeper in this setup requires a small trick. We need to run it on the first node in a “standalone” mode and then on all the other nodes refer to an already running Zookeeper instance and get it connect to it. Zookeeper then automatically reconfigures itself to sync all the nodes.

在这个配置中，运行容器化的Zookeeper需要一点小技巧。需要在第一个节点运行“standalone” 模式，等其他所有节点成为已经运行的Zookeeper实例并且能连接到它。Zookeeper接着会自动重新配置它自己向所有节点同步。

## Marathon

## Marathon

Because we want to run application containers and/or other frameworks, we install the “datacenter init system” Marathon framework on top of Mesos.

因为我们想运行应用容器和其它框架，我们在Mesos之上安装“datacenter init system” Marathon框架。
```shell
MARATHONVERSION="v0.8.2"
ZK="zk://<quorum>" # zookeeper quorum string

docker run -d \
 -p 8080:8080 \
 -p 5051:5051 \
 mesosphere/marathon:${MARATHONVERSION} \
 --master ${ZK}/mesos \
 --zk ${ZK}/marathon
 ```

 Note that we expose port `8080`, where the Marathon UI runs, and port `5051`, where Marathon listens. We run the image in an arbitrary version and pass it two mandatory parameters:

注意，我们指定8080端口，Marathon UI 运行和5051端口，Marathon监听。我们运行镜像在任意版本并且传递两个mandatory参数：

 `master`  – quorum hostname/ip and Mesos registration path
 `zk`  – quorum hostname/ip and Marathon registration path

 This way the key components of a Mesos cluster can run in Docker containers. Aside from our main motivation – running specific versions of the components – this approach naturally brings all the advantages of containerization to the cluster management level: faster deployments, simplified maintenance and portability across machines.

这样Mesos集群的关键组件可以在Docker容器中运行。除了我们主要的动机-运行不同版本的组件-这个方式自然而然的带来在集群管理层次容器化的所有好处：快速部署，维护简单和设备可移植性。

 We still need to figure out how to set up haproxy-marathon bridge and [Mesos DNS](http://mesosphere.github.io/mesos-dns/) in this setup.

在这一过程，我们还需要解决haproxy-marathon bridge配置和[Mesos DNS](http://mesosphere.github.io/mesos-dns/)的问题。

 If you have any questions or remarks, please comment under the article or [create issues in GitHub](https://github.com/ContainerSolutions/terraform-mesos/issues).

如果你有任何问题或者评论，请直接在文章或者 [create issues in GitHub](https://github.com/ContainerSolutions/terraform-mesos/issues)留言。

From：http://container-solutions.com/containerized-mesos-cluster/
