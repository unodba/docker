We often use [Apache Mesos](http://mesos.apache.org/) clusters to run containerized applications for our clients and for ourselves. Even though we run applications in containers, we still install Mesos with its dependencies and frameworks on host machines from standard [Debian packages](https://mesosphere.com/downloads/). Although that’s what most Mesos users do and it’s a straightforward and safe way of installing Mesos, we thought we might stretch the Mesos installation a little bit and try to run this stuff in containers too.

Apart from this being a lot of fun and educational, this approach helped us to solve one very practical issue. We are now able to run arbitrary versions of Mesos and its frameworks, including the release candidate versions, because the problem shrank to running different versions of the Docker images in our setup.

Programming against the most recent versions of Mesos (and other tools) is important for us when we develop [our own Mesos frameworks](https://github.com/mesos/elasticsearch). Additionally, this gives us more time to apply security fixes, sometimes before they are even released.

# Mesos, Marathon and Zookeeper Docker Images

There are a few different [Docker images for Apache Mesos](https://registry.hub.docker.com/search?q=mesos) available on the Docker Hub at the moment (note that Mesos master and slave are two different images). We have chosen the images from [Mesosphere](https://registry.hub.docker.com/repos/mesosphere/), who are a significant contributor to the ecosystem (and our friends). [Redjack](https://registry.hub.docker.com/repos/redjack/)’s repository also contains quality Mesos images with comprehensive documentation. It’s up to you which one you use, both worked just fine for us, but be aware that there might be small differences in the configuration, as this article is written for Mesosphere images.

[Thijs Schnitger](https://github.com/thijsschnitger) created a [Docker image for Zookeeper 3.5](https://registry.hub.docker.com/u/containersol/zookeeper/) last week. This version of Apache Zookeeper introduces dynamic hosts reconfiguration, which is a very cool feature that is especially useful in a dynamic environment, which a cluster might be. So we use Container Solutions’ Zookeeper image in this setup.

We also use the [Marathon Docker image from Mesosphere](https://registry.hub.docker.com/u/mesosphere/marathon/).

# Containerized Cluster Configuration

We use [Terraform](https://terraform.io/) to launch some of our clusters. Refer to [terraform-mesos](https://github.com/ContainerSolutions/terraform-mesos) GitHub repository and check out its [containers](https://github.com/ContainerSolutions/terraform-mesos/tree/containers) branch, where you can find a working solution (at least at the moment of writing). Later on, it will likely be merged to the master branch.

Let’s take a look on the interesting parts of the setup, the docker run commands for Mesos, Marathon and Zookeeper.

## Mesos Master

```shell
QUORUM=2 # number of masters divided by 2 plus 1
CLUSTERNAME="cluster7"
ZK="zk://<quorum_string>/mesos"
MESOS_VERSION="0.22.1-1.0.ubuntu1404"

docker run -d
-e MESOS_QUORUM=${QUORUM}
-e MESOS_WORK_DIR=/var/lib/mesos
-e MESOS_LOG_DIR=/var/log
-e MESOS_CLUSTER=${CLUSTERNAME}
-e MESOS_ZK=${ZK}/mesos
--net=host
mesosphere/mesos-master:${MESOSVERSION}
```
As you can see, we simply pass a reference to a version tag of the image to get a specific version of Mesos running. Aside from that, we need to enable [host networking](https://docs.docker.com/articles/networking/) for Mesos to work properly and pass several environment variables to the container. As a convention, the `MESOS_` prefixed variables hold the configuration values, we’d normally write to the config files in `/etc/mesos*` directories on the host.

We run a master container on each host in the cluster.

## Mesos Slave

```shell
ZK="zk://<quorum_string>/mesos"
HOSTNAME="host1" # use a hostname of the host
IP="10.20.30.2" # use an IP address of the host
MESOSVERSION="0.22.1-1.0.ubuntu1404"

docker run -d
 -e MESOS_LOG_DIR=/var/log/mesos
 -e MESOS_MASTER=${ZK}
 -e MESOS_EXECUTOR_REGISTRATION_TIMEOUT=5mins
 -e MESOS_HOSTNAME=${HOSTNAME}
 -e MESOS_ISOLATOR=cgroups/cpu,cgroups/mem
 -e MESOS_CONTAINERIZERS=docker,mesos
 -e MESOS_PORT=5051
 -e MESOS_IP=${IP}
 -v /var/run/docker.sock:/var/run/docker.sock
 -v /usr/bin/docker:/usr/bin/docker
 -v /sys:/sys:ro
 --net=host
 mesosphere/mesos-slave:${MESOSVERSION}
 ```

 For the Mesos slave, the configuration is a bit more complex, mainly because we need to make sure the slave can access the Docker daemon running on the host (the same daemon that’s used to run this Mesos slave instance). We do it by mounting `/var/run/docker.sock` file, `/usr/bin/docker` binary and the `/sys`  directory (read-only) to the Mesos slave container. Please note, that this is not a bulletproof solution and might need to be further tweaked, e.g. when running another Mesos framework by Marathon.

 Zookeeper
 ```shell
 # first node

docker run -d -p 2181:2181 containersol/zookeeper 1

# other nodes
MYID="2" # 3,4,5,...
FIRST_NODE="cluster7-mesos-master-0" # hostname of the first node

docker run -d -p 2181:2181 containersol/zookeeper ${MYID} ${FIRST_NODE}
```
Running containerized Zookeeper in this setup requires a small trick. We need to run it on the first node in a “standalone” mode and then on all the other nodes refer to an already running Zookeeper instance and get it connect to it. Zookeeper then automatically reconfigures itself to sync all the nodes.

## Marathon

Because we want to run application containers and/or other frameworks, we install the “datacenter init system” Marathon framework on top of Mesos.

```shell
MARATHONVERSION="v0.8.2"
ZK="zk://<quorum>" # zookeeper quorum string

docker run -d
 -p 8080:8080
 -p 5051:5051
 mesosphere/marathon:${MARATHONVERSION}
 --master ${ZK}/mesos
 --zk ${ZK}/marathon
 ```

 Note that we expose port `8080`, where the Marathon UI runs, and port `5051`, where Marathon listens. We run the image in an arbitrary version and pass it two mandatory parameters:

 `master`  – quorum hostname/ip and Mesos registration path
 `zk`  – quorum hostname/ip and Marathon registration path

 This way the key components of a Mesos cluster can run in Docker containers. Aside from our main motivation – running specific versions of the components – this approach naturally brings all the advantages of containerization to the cluster management level: faster deployments, simplified maintenance and portability across machines.

 We still need to figure out how to set up haproxy-marathon bridge and [Mesos DNS](http://mesosphere.github.io/mesos-dns/) in this setup.

 If you have any questions or remarks, please comment under the article or [create issues in GitHub](https://github.com/ContainerSolutions/terraform-mesos/issues).

From：http://container-solutions.com/containerized-mesos-cluster/
