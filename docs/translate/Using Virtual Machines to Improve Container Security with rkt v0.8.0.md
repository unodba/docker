在rkt v0.8.0版本使用虚拟机改善容器安全性
==

【编者的话】本文来自CoreOS官方博客，上周rkt发布了0.8.0版本，rkt 0.8.0引入了很多新功能特性，包括初步支持用户命名空间和使用硬件虚拟化增强容器隔离，同时改进了主机日志集成，容器套接字激活，改进的镜像缓存和提升速度。本文重点介绍新引入
的硬件嵌入式虚拟化技术，最后简单介绍了一下开放容器平台的进展和如何向rkt贡献。

今天，我们发布rkt [v0.8.0](https://github.com/coreos/rkt/releases/tag/v0.8.0)。rkt是一个应用容器运行时，为了构建高效、安全和可组合的生产环境。

这次发布的版本包括新的安全功能，其中包括初步支持用户命名空间和使用硬件虚拟化增强容器隔离。
我们同样也引入了一系列的改善措施，如主机日志集成，容器套接字激活，改进的镜像缓存和速度提升。

# 英特尔使用虚拟化贡献rkt stage1

rkt的模块化设计支持不同的执行引擎并且能构建和接入容器化的系统。这通过阶段体系架构[staged architecture](https://github.com/coreos/rkt/blob/master/Documentation/devel/architecture.md)来实现，其中第二阶段("stage1")负责创建和启动容器。当我们启动rkt时，它以一个，缺省的stage1利用Linux的cgroups和namespaces(通常他们被称作"Linux 容器")。

在英特尔工程师的帮助下，我们增加了一个新的利用虚拟化技术的rkt stage1运行时。这意味着在rkt下运行的应用，使用这个新的stage1可以像
Linux的KVM管理程序使用的硬件特性一样从主机内核层面进行隔离。

今年五月份，英特尔宣布了这项用于rkt之上的概念验证，作为[Intel® Clear Containers](https://lwn.net/Articles/644675/)利用硬件嵌入式虚拟化技术特性来更好的保护容器运行时和
隔离应用的成果之一。我们非常激动的看到这项工作正在进行，并且rkt正在渐渐成型，因为它验证了我们做的一些早期设计原型，比如运行时阶段和资料库的理念。下面是[Intel's Open Source Technology Center](https://01.org/blogs/2015/optimizing-commercial-products-intel-clear-linux)
的Arjan van de Ven说过的话：

> "多亏了rkt的基于阶段的体系结构，Intel®Clear Containers team才能够迅速的整合我们的工作，把进一步增强安全性的英特尔®虚拟化技术(Intel® VT-x)
带到容器生态中。我们非常高兴能继续与rkt社区共同工作，来实现我们在传递容器应用部署好处的同时使用硬件嵌入技术增强容器安全性的愿景。"

自从五月份发布原型以来，我们一直在与英特尔的团队共同协作，来确保当使用虚拟化时诸如每个pod一个IP地址和卷组这些特性还是以原来的方式运行。今天rkt的发布见证了这一功能完全融合，使得后端的lkvm是一流的stage1体验。
那么，让我们来试试吧！

在这个示例中，我们首先使用默认的cgroups/namespace-based stage1运行一个pod。我们使用[systemd-run](http://www.freedesktop.org/software/systemd/man/systemd-run.html)启动容器，这会动态构造一个单位文件并启动它。检查这个单元的状态会让我们弄清楚在引擎下到底发生了什么。

```shell
$ sudo systemd-run --uid=0 \
   ./rkt run \
   --private-net --port=client:2379 \
   --volume data-dir,kind=host,source=/tmp/etcd \
   coreos.com/etcd,version=v2.2.0-alpha.0 \
   -- --advertise-client-urls="http://127.0.0.1:2379" \  
   --listen-client-urls="http://0.0.0.0:2379"
Running as unit run-1377.service.

$ systemctl status run-1377.service
● run-1377.service
   CGroup: /system.slice/run-1377.service
           ├─1378 stage1/rootfs/usr/bin/systemd-nspawn
           ├─1425 /usr/lib/systemd/systemd
           └─system.slice
             ├─etcd.service
             │ └─1430 /etcd
             └─systemd-journald.service
               └─1426 /usr/lib/systemd/systemd-journald
```
请注意，我们可以看到pod内的整个过程层次结构，包括systemd实例和etcd进程。

接着，我们通过添加--stage1-image标签在新的基于KVM的stage1下启动这个容器：

```shell
$ sudo systemd-run -t --uid=0 \
  ./rkt run --stage1-image=sha512-c5b3b60ed4493fd77222afcb860543b9 \
  --private-net --port=client:2379 \
  --volume data-dir,kind=host,source=/tmp/etcd2 \
  coreos.com/etcd,version=v2.2.0-alpha.0 \
  -- --advertise-client-urls="http://127.0.0.1:2379" \
  --listen-client-urls="http://0.0.0.0:2379"
...

$ systemctl status run-1505.service
● run-1505.service
   CGroup: /system.slice/run-1505.service
           └─1506 ./stage1/rootfs/lkvm
```

请注意，该进程层级到lkvm就结束了。这是因为整个pod现在是在KVM进程内执行，包括systemd进程和etcd进程：对主机系统而言，它就像一个虚拟机进程。
通过在调用容器时添加一个标签，我们就利用了公有云用来隔离租户的KVM技术来隔离我们的应用容器，给主机上又加了一个安全层。

感谢来自英特尔的Piotr Skamruk, Paweł Pałucki, Dimitri John Ledkov, Arjan van de Ven的支持和付出。关于这个功能的更多细节请参考
[lkvm stage1 guide](https://github.com/coreos/rkt/blob/master/Documentation/running-lkvm-stage1.md)。

# 无缝集成主机级别日志

在systemd主机上，[日志](http://www.freedesktop.org/software/systemd/man/systemd-journald.service.html)是默认的日志聚合系统。随着v0.8.0版本发布，rkt现在能自动与主机日志集成了，如果检测到会提供一个systemd原生
日志管理体验。如果需要体验rkt产品的日志，你仅仅需要添加一个机器区分符如`-M rkt-$UUID`到主机的`journalctl`命令。

举个简单例子，我们来体验一下之前启动的etcd容器的日志。首先我们使用`machinectl`列出rkt已经注册到systemd的pods：

```shell
$ machinectl list
MACHINE                                  CLASS     SERVICE
rkt-bccc16ea-3e63-4a1f-80aa-4358777ce473 container nspawn
rkt-c3a7fabc-9eb8-4e06-be1d-21d57cdaf682 container nspawn

2 machines listed.
```

我们可以看到etcd的pod列出的第二台机器已经被systemd发现。现在我们使用jornal直接查看pod的日志：

```shell
$ sudo journalctl -M rkt-c3a7fabc-9eb8-4e06-be1d-21d57cdaf682
etcd[4]: 2015-08-18 07:04:24.362297 N | etcdserver: set the initial cluster version to 2.2.0
```

# 用户命名空间支持

这次的版本包括对[用户命名空间](http://man7.org/linux/man-pages/man7/user_namespaces.7.html)初步支持来改善容器隔离。通过使用用户命名空间，应用程序在容器内可以以root用户运行但是在容器外会被映射到非root用户。
通过从系统的root用户隔离容器增加了额外安全层。这个功能预览版本还是是实验性的并且使用拥有特权的用户命名空间，但是rkt的未来版本会在这个版本的基础上继续改进并提供更多规则控制。

为了打开用户命名空间，需要添加两个标签到我们最开始的例子：`--private-users`和`--no-overlay`。第一个是打开用户命名空间功能，第二个是关闭rkt的overlayfs子系统，因为现阶段它与用户命名空间不兼容。

```shell
$ ./rkt run --no-overlay --private-users \
  --private-net --port=client:2379 \
  --volume data-dir,kind=host,source=/tmp/etcd \
  coreos.com/etcd,version=v2.2.0-alpha.0 \
  -- --advertise-client-urls="http://127.0.0.1:2379" \
     --listen-client-urls="http://0.0.0.0:2379"`
```

我们通过使用`curl`来验证etcd的功能行并且检查etcd数据目录的权限来确认这个功能正常，注意，从主机的角度看etcd成员目录被一个id很高的用户拥有：

```shell
$ curl 172.16.28.19:2379/version
{"etcdserver":"2.2.0-alpha.0","etcdcluster":"2.2.0"}`

$ ls -la /tmp/etcd
total 0
drwxrwxrwx  3 core       core        60 Aug 18 07:31 .
drwxrwxrwt 10 root       root       200 Aug 18 07:31 ..
drwx------  4 1037893632 1037893632  80 Aug 18 07:31 member`
```

增加用户命名空间支持是对我们让rkt成为最安全的容器运行时目标迈出的重要一步，在接下来的版本我们会继续努力改进这一功能-你可以查看[roadmap in this issue](https://github.com/coreos/rkt/issues/986)。

# 开放平台项目进展

在rktv0.8.0版本我们进一步巩固在安全强化方面的成果，并向1.0稳定版本和生产版本推进。我们还致力于确保容器生态系统继续朝着大家发布容器到“构建一次，签名一次，到处运行。”路线前进。如今rkt是应用程序容器规范(appc)的实现，
在未来我们希望rkt成为开放容器平台（OCI）规范的实现。不管怎样，OCI还在起步阶段还有很多工作需要做。查看OCI和appc协调工作进展,你可以在
[OCI dev邮件列表](https://groups.google.com/a/opencontainers.org/forum/#!topic/dev/uo11avcWlQQ)阅读更多内容。

# 向rkt贡献

rkt的一个目标是使得rkt成为最安全的容器运行时，并且在我们向1.0版本演变中有还有许多令人兴奋的工作要做。加入我们的使命:欢迎您参与rkt的发展,通过[rkt-dev邮件列表](https://groups.google.com/forum/#!forum/rkt-dev)讨论,
提交[GitHub问题](https://github.com/coreos/rkt/issues),或直接向项目[贡献](https://github.com/coreos/rkt/issues?q=is%3Aopen+is%3Aissue+label%3A%22help+wanted%22)。

原文连接：[Using Virtual Machines to Improve Container Security with rkt v0.8.0](https://coreos.com/blog/rkt-0.8-with-new-vm-support/)（翻译：朱高校 审校：）
