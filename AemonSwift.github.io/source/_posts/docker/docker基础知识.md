---
title: docker基础知识
date: 2019-11-13 08:56:45
tags:
---

# 容器和虚拟化之间的关系

## 常用虚拟化方式
### 主机级别虚拟化——虚拟化整个完整物理硬件平台
两种类型实现：
1. TYPE-1: 有宿主机，在宿主机上安装宿主操作系统，在宿主操作系统系统上安装虚拟机管理器，在虚拟机管理器安装虚拟机
2. TYPE-2: 直接在硬件平台上安装虚拟机管理器，在虚拟机管理器安装虚拟机（没有宿主操作系统）。


运行内核目的：资源分配和管理
真正在用户空间跑的应用程序才是生产力，而现在的软件都是依赖内核而运行，例如：应用程序所依赖的库。
主机级别虚拟化缺陷：产生较多的开销
需要两级调度和资源分配
1. 虚拟机里面存在内核，存在cpu调度和io的调度
2. 虚拟及内核也是被宿主机或硬件平台上的虚拟机管理器进行调度

有时创建虚拟机目的是为了创建几个富有生产力的进程，故在主机级别虚拟化基础上减少中间层，来提高效率。

## 容器
例如：把虚拟机内核抽调，只保留进程。
虚拟机目的：环境隔离。若一台宿主机运行两个Nginx，他们都要监听80端口，而主机上只有一个80套接字。而虚拟机可以达到目的，而且一个nginx损坏了，不会影响其他Nginx的运行。
把虚拟机内核抽调，只保留进程，需要做的是：两个进程相互隔离，而且是不可达的，碰巧是他们共享硬件资源

隔离是隔离的用户空间，一般有一个用户空间是有特权的（一般是第一个），用它来管理其它用户空间。
隔离空间是用来放进程的，给进程提供运行环境，而且保护进程不受其它进程的干扰。即不管这个进程有任何异常行为，都不会影响此隔离空间之外的进程——程序的应用安全运行（防止其他人攻击）。

一个用户空间主要是实现隔离环境，任何进程运行在用户空间的时候，它就以为自己就是唯一一个运行在用户空间内核之上的进程。一个用户空间能看到的东西。
1. 主机名，主机域名 （UTS）
2. 根文件系统（每个用户空间都要有自己独立的根文件系统）（Mount）
3. IPC（信号量，消息队列，共享内存等）。两个用户空间的进程可以通过IPC进行通信，则隔离无任何意义（例如两个进程通过共享内存通信，则隔离无意义）。每个用户空间的IPC是独立的（即一个用户空间的进程可以通过IPC进行通信），但跨用户空间是不行的。而内核管理是期望任何两个进程可以通过IPC进行通信。
4. 进程树（NIT），PID。每个进程应该从属于某个进程（进程右父进程创建），一个系统运行无非就是两棵树——进程树，文件系统树，让进程认为自己是当前用户空间唯一运行的程序，故需要制造出假象：要么自己是一个NIT，要么自己从属于某个NIT（用户空间运行多个进程）。linux进程管理都是子进程的创建和回收都是由父进程来做的。
5. User用户组，运行的进程应该以某个用户身份来运行。第一个用户空间的用户和第二个用户空间的用户的id号一样，但名字不一样是存在。(每个用户空间每个主机上id为0的root，id为1-999的为系统用户，id>1000的为普通登录用户)，而每个用户空间都需要root，但在一个真正的内核空间只能有一个root。产生问题（一个用户空间的root可以管理别人用户空间的资源）。因此用户空间的root只能伪装出来，这个root用户只是一个普通用户，但针对这个用户空间来讲，可以让这个普通用户的id为0，只是让他在此用户空间为所欲为。
6. 网络。每个用户空间都以为自己是系统的唯一一个用户空间，能够看见自己的专用的ip地址，端口，已经tcp/ip协议栈等。而且两个容器间的用户可以通过网络进行通信。而在内核级专用的ip地址，端口，已经tcp/ip协议栈等都是只有一个。

上述资源在内核设计的时候只是针对单个用户空间；为了适应多个用户空间，在内核级直接对上面的资源虚拟化。每种资源只要在内核能够切分为多个互相隔离的环境，我们把它为名称空间。

例如：UTS可以以名称空间为单位进行隔离。在同个内核之上，创建多个名称空间，在名称空间之上，让uts资源的每个名称空间相互隔离。因此每个名称空间都有自己独立的名称，主机名和域名是内核的作用域，所以一个主机房主机名只有一个。但现在有多个用户空间，就必须要为每个用户空间视作这个系统上唯一的用户空间。因此每个名称空间都应有有自己的主机名而且与其他用户空间的主机名不一样。若都是用内核的主机名和域名；则内核混乱了。古内核在内核级进行隔离，让他们分别各自使用而不影响真正的内核空间的用户名和域名。

linux为了适应容器的实现，linux在内核级通过所谓的名称空间机制对上述资源进行了隔离支持。直接通过系统调用向外输出。
创建进程：clone()，把创建的进程放到名称空间去：setns()等

除了上述问题：主机级别隔离；还存在只能使用多少个cpu，以及使用的内存有多大。（超出了限制会直接虚拟的主机直接挂掉，不影响其他虚拟主机）。而容器虚拟化也需要做到这两点。即内核级还要限制名称空间所有的资源总量。（例如3个名称空间：cpu分配：1：2：1（这样分配很有弹性），例如后面两个不启动，第一个用户空间可以吃掉这个cpu的核数；另外一种分配：一个用户空间最多使用几核。而内存资源也是这样）。内核实现这需求，还需要CGroups（control groups）
CGroups（control groups）：把系统资源分为多个组，把每个组的资源量指派到对应的用户空间，而每个子组也可以被再次划分。
- blkio：块设备io
- cpu： cpu
- cpuacct：cpu资源使用报告
- cpuset： 多处理平台上的cpu集合
- devices： 设备访问
- freezer： 挂起或恢复任务
- memory： 内存用量及报告
- perf_event：对cgroup中的任务进行统一性能测试
- net——cls：cgroup中的任务创建的数据报文的类别标识符

容器的隔离能力比主机级虚拟化弱了很多，由于都是属于同一个内核，只不过在内核里强行设置边界。为了防止一个用户空间进程利用漏洞进入另外一个用户空间里，故利用selinx等各种安全加强机制来加固用户空间的边界。

## LXC——linux container
是一组工具，为了用户快速使用容器。
- lxc-create：创建容器（用户空间），用户空间应该有bin，sbin这样的目录结构，以及基本的应用程序，例如ls等，这种东西怎么创建——可以从宿主机进行copy。若创建的用户空间：根用户空间是centos，新用户空间是Ubuntu，则复制宿主空间的是不行的，因此需要一个机制来自动创建对于用户空间的应用程序，——即模板，是一种脚本，创建用户空间的时候回自动执行此脚本，此脚本指向了名称空间指向的linux发行版的仓库，在仓库中把对应需要的东西下载下来。

使用lxc就要学好lxc工具，以及会写定制脚本，更重是每个用户空间都是安装生成的，后来运行过程中生成了很多文件（数据），若此宿主机出现故障的时候，如何将此数据迁移到其他宿主机上？以及批量创建容器也不容易。于是后面出现了docker。

docker是一种容器易用的前端工具，用于简化使用容器。容器是linux内核中的技术，docker只是简化容器使用的工具。docker是在lxc上进行封装发行版。
 docker利用lxc做容器管理引擎，创建用户空间时不再是利用模板去现场安装。而是事先利用一种镜像技术（尝试把os在用户空间用到的所有组件事先准备编排好，打包成一个文件），这些镜像文件放在了互联网的一个集中统一仓库里面，把大家都会用到的镜像都放在仓库里面。为了使得容器使用更加易于管理，docker采用了在一个容器内只运行一个进程（这样可以最小化安装系统工具）。
 这里存在一个坏处：一个系统上的运行进程出现了故障，会使用一组跟踪工具来观察进程运行情况。而现在每个进程应用在自己的用户空间中，这些工具需要在此用户空间携带上（每个用户空间自带调试工具）。调试工具是用到时再启动。当主进程终止时，这个用户空间也没存在意义了。因此，先做调试进程有可能没有工具，以及要突破容器的边界（以前没有边界）。
 docker为开发做到了一次编辑到处运行，以及部署简单。开发以前是为了开发每种平台的软件。现在只是编辑完，打包成镜像。发布只需运行镜像即可。

发布操作可以靠编排工具来实现。
docker还可以批量创建创建容器：每个物理机只需下载一个docker即可，docker运行容器：与它的镜像构建有关。docker的镜像构建机制：分层构建，联合挂载。
分层构建：底层做一个纯净的centos镜像，在此镜像之上装一个nginx。把两个叠放在一起就是一个nginx镜像。使得镜像分发没有那么庞大了，例如运行三个容器，他们可以共同使用底层的镜像（centos镜像），上层就对应特定的服务，nginx，Tomcat等。能这样做的原因：每个镜像只是只读的。
若第一个容器更改了文件，将会影响其他容器。故联合挂载栈的最顶层额外附加新层，这个层是每个容器专有的。若删除文件，只需标记不可见；若改：则写时复制，把底层的东西copy上来，后面只看上面的东西。但存在问题：容器迁移存在问题：由于存在专用层（维持更改），因此真正使用容器时，不会在容器本地保持有效数据。需要在专有层挂载一个外部持久的共享存储。后面重新启动的时候，运行镜像，挂载此共享存储，达到正常运行。

启动的容器调度到集群某个dockerhost主机上面，以及启动有依赖关系的容器序列，移除有依赖的容器序列——这个工具叫容器编排工具。例如：kubernetes等。

现在docker采用另外的引擎：libcontainer，替换了lxc。由于docker要发展，故需要标准化因此从libcontainer发展成runC（OCF——开放容器标准）。
生成镜像要遵循OCI标准

## docker架构
docker daemon，docker client，docker registries三部分组成。
docker是一种C/S架构，遵循restful风格。
运行docker daemon进程表示，把此主机作为服务器，可以监听在某个套接字之上，为了安全，它只提供unix文件的套接字。（mysql有两种接入方法：tcp/ip标准的ip协议接入；使用socket文件接入）
docker host是真正运行容器的主机，关系两个组成：容器，镜像。镜像来自于registry（镜像仓库，用到那个下载那个到本地）。镜像是分层构建的，可以共享很多基础组件供多个镜像使用（由于镜像只可读）。启动容器时是基于镜像而启动的，是为了一个容器创建一个专用的可写仓库。

docker registries：docker repository只放一个应用程序，是为了此应用程序不断升级产生的多个版本。而每个仓库名就是应用程序名，例如nginx。但为了区分仓库中的镜像，故需要标签来区分。因此：只有仓库名+标签才能唯一确定一个镜像
镜像和容器的关系就是程序和进程之间的关系。镜像是静态的，容器是动态的。
任何restful的资源都是可以进行增删改查，docker中的例如：镜像，容器，网络，卷，插件等。

centos extras仓库中有docker，里面的docker版本较老，一般不用此仓库的docker，一般去互联的镜像网站去重新安装一个仓库
```shell
yum repolist
yum install docker-ce
```
docker默认配置文件为json格式的文件。配置文件位于/etc/docker/daemon.json文件，此文件默认不存在，自己需要定义，为了下载更快一点，可以在此文件设置镜像加速器(加速器有：docker-cn，阿里云，中科大镜像等)。
```shell
# daemon.json
"registry-mirrors":["https://registry.docker-cn.com"] #可以设置多个镜像
```
启动服务：`systemctl start docker.service`
docker使用方式：
```
分组管理命令 --新用法
docker 组 组中的子命令

原始命令
docker 命令

docker help 
docker version
docker info
```
存储驱动：docker的镜像是分层构建，联合挂载，做到这点必须要求特殊的文件系统才行。因此docker引入了存储驱动，来适应宿主机的文件系统格式。

```
docker search 
docker pull 
docker images

docker image pull nginx:1.14-alpine
```
容器名称：无斜线的表示顶级仓库，一般是dockerhub官方的（nginx），有斜线的表示用户仓库（项目仓库：一个人注册账号单独建立的）（jwilder/nginx-proxy）。
利于nginx镜像是基于centos构建的，比较大；因此有较小的构建方式——alpine标记（专门用于构建非常小的文件的微型发行版），能够为程序运行提供基础环境，但体积非常小。但里面少了调试工具，在生产环境中不能使用。尽量自己做镜像，自己带调试工具。而且dockerhub上面的镜像也是建立在centos最小版本之上，里面也缺少各种调试工具。因此需自己建立私有registry仓库。

```
busybox：自己是一个程序，可以创建符号链接，例如链接成ls，它就成为ls的功能。一个程序实现成linux百来种命令

```
doker ps：容器id，基于那个镜像创建的容器，在容器中运行的命令（docker容器只是为了运行单个进程，每个镜像都有它自己默认要运行的命令，但我们也可进行更改）。

```
-it进入交互式窗口
docker run --name b1 -it busybox:latest
docker inspect b1---查看容器信息
docker start -ai b1 ---ai主要是为了创建时有-it而进行的
docker kill 发了9信号 （强制终止），一般不要强制终止，防止数据丢失
docker run --name web -d nginx:lates-aplpine ----在后台运行（command中显示 daemon of...）
docker stop 发了15信号
```
一个容器是为了跑一个程序；若启动的程序跑在后台运行，则这个容器表示没有任何程序（以为此程序终止了）。则容器会终止。

```
docker exec ---绕过容器的边界，访问容器里面内容
docker exec -it redis /bin/sh  ---交互式使用shell
netstat -tnl
```
每个容器只是运行一个程序，而这个程序生成的日志。linux生成的日志在`/var/log/程序名`的文件中。而容器只运行一个进程，故所有日志都放在控制台上。用docker container logs去获取运行的日志。

docker镜像含有启动容器所需的文件系统及其内容，其用于创建并启动容器
采用分层构建机制：底层bootfs，上层为rootfs（构建用户空间，运行容器）
- bootfs：用于系统引导的文件系统，包括了bootloader和kernel，容器启动完成以后会被卸载以节约内存资源
- rootfs：位于bootfs之上，表现为docker容器的根文件系统。传统模式中，系统启动之时，内核挂载rootfs时会首先将其挂载为“只读”模式，完整性自检完成后将其重新挂载为读写模式。而在docker中，rootfs由内核挂载为“只读”模式，而后通过联合挂载技术挂载一个“可写”层。

启动apache，先挂载最底层debian，启动完成再挂应用工具层，最后挂载应用程序层，他们叠加在一起挂载的，这三层只是可读的。容器启动完成以后，若某些进程需要更改文件中的内容，以及创建临时文件时，都会在tmp目录里面，但镜像是不可写的。因此writeable是容器特有的。镜像是可共享的。容器所有写操作都是在容器writeable实现的。

分层构建，联合挂载依赖于专用的文件系统来支撑实现，常用可用的文件系统
- aufs：前身为unionFS，aufs不是内核自有的文件系统（代码量太长）。centos因此没有融合在内核中，而Ubuntu融合在内核系统中
- overlayfs：
- btrfs：
- devicemapper(简称dm):不实用，也不稳定


启动容器时，docker daemon会试图从本地获取相关的镜像，本地镜像不存在时，其将从registry中下载镜像并保存到本地。

docker registry分类
- sponsor registry：第三方registry，供客户和docker社区使用
- mirror registry：第三方registry，只让客户使用
- vendor registry：由发布docker镜像的供应商提供的registry
- private registry：通过设有防火墙和额外的安全层的私有实体提供的registry

registry组成
- repository 
- index：维护用户账号，镜像的校验以及公共命名空间的信息；相当于为registry提供了一个完成用户认证等功能的检索接口

registry的镜像由开发人员制作，然后推送至registry，供其他人员使用，例如：部署到生产环境中。存在问题：如何为不同环境配置不同的信息。——通过向容器启动时传环境变量来传信息。而容器启动时从环境变量中加载注入到程序的配置中。
我们做镜像都是在已有的base镜像而做的。例如纯净最小版的centos镜像，在此基础上加层次。而base镜像怎么而来？——由docker hub官方人员来做。
做镜像两种方式：
1. 基于容器去做，docker运行一段时间，把可写层也纳入到镜像中采用（即把可写层单独创建一个镜像层）。docker commit命令即可
2. 基于dockerfile去做，完成以后使用docker build或者上传到automated builds中自动创建镜像（gihub仓库一个项目就是做镜像的项目，dockerhub中监听这个仓库，监听dokerfile文件，会从github中拖过来，构建镜像）

```
docker pull 服务器名/用户名/仓库名:标签---针对不是在默认仓库中下载镜像，port默认使用443
```
基于容器做镜像：例如在busybox基础上，创建data目录创建网页目录以及内容，把新加的内容做成镜像。下次用此镜像启动时会有此内容
```
docker run --name b1 -it busybox

mkdir -p /data/html
vi /data/html/index.html 
<h1> BusyBox http server </h1>


docker commit -p b1 --做镜像时必须要容器暂停，否则制作过程中会产生数据
docker tag imageid号 aemon/httpd:v0.1-1 --给自己做的镜像打标签
docker tag  aemon/httpd:v0.1-1 aemon/httpd:latest
```
删镜像只是删引用，由于可能其他镜像名仍然指向此镜像。
上面制作的镜像并没有改变容器默认运行的命令。用docker inspect 镜像名 信息中cmd来看运行的命令

```
docker inspect aemon/httpd:v0.1-1
- a: 容器制作的作者
- c: 设置容器默认运行的命令
docker commit -a "111@aa.com" -c 'CMD ["/bin/httpd","-f","-h","/data/html"]' -p b1 aemon/httpd:v0.2-1

登录其他服务器，后面还要制定服务器名
docker login -u 用户名 
登录成功后可以往上推镜像
docker push 镜像名 --若没有更tag，会把此仓库的所有镜像都会push上去
若推导其他服务器时，镜像名也要带上服务器
```

镜像共享方式
1. push
2. 导入和导出 多台服务器，一台服务器做了一个镜像，另外一台想用这个镜像。若采用push方式很麻烦。简单方式将镜像打包文件，在另外一台服务器解包即可
```
docker save  -o --- 打包文件（可打包多个镜像文件）
docker load -i  ---解包
```
服务器上传文件：
```
scp /etc/docker/daemon.json node02:/etc/docker/daemon.json
```

# Docker网络虚拟化
若名空间太多，而网卡太少；所有名称空间中的进程也需要通过网络通信。可以使用虚拟网卡设备。用纯软件方式模拟设备的使用，linux内核级支持二层设备，三层设备（层次表示计算机网络层次）
OVS（open vSwith），SDN （soft definitive netowrk）

docker自动提供了四种种网络
docker network ls
- bridge :nat桥，在本机上创建一个软交换机（可以扮演交换机，网卡使用(给网卡号)），随后我们启动一个容器，会给这个容器分配一对网卡的地址，一半在容器上，一半在宿主机上，此网卡关联到桥上
- host：让容器使用宿主机的网络名称空间，就拥有了管理物理网络的特权
- none：不给容器设置网络，只有lo接口；相当于宿主机没有网卡。例如用于处理数据。
- joined：多个容器可以共享uts，net，ipc名称空间。

```
yum -y install bridge-utils
brctl show --看交换机关联的网卡
iptables -t nat -vnL

docker  network inspect bridge 
rpm -q ip iproute //查看包是否安装
```

几种通信方式
- 1. 多个容器在一个交换机上连着
- 2. 通过物理机来访问
- 3. 其他物理机的访问：使用桥接网络时，只能在宿主机上添加delate规则，以便于它可以外部的其他客户端访问。例如80端口转给web1；而存在web服务器时，宿主机80端口只有一个，其他web服务器怎么访问？——nat桥面临的问题。而overlay network就能满足。因为不需要做映射。而容器的nat桥能够满足，由于名称空间存在


当使用ip管理网络名称空间时，只有网络名称空间是被隔离的，其他名称空间都是共享的。
```
ip netns 网络名称空间信息

ip netns add r1
ip netns exec r1 ifconfig -a //无网卡
//创建虚拟网卡段方式
ip link add name veth1.1 type veth peer name veth1.2 --创建一对网卡
ip link set dev veth1.2 netns r1 
ip link show 

ip netns exec ri ip link set dev veth1.2 name eth0 --改网卡名

ifconfig veth1.1 10.1.0.1/24 up --激活网卡
```

```
docker run --name b1 -it --network bridge --rm busybox:latest

docker run --name b1 -it --network bridge -h aemon --rm busybox:latest //容器外注入进来的主机名
hostname
此容器通过主机名的方式访问其它主机或容器：
1. 这个容器能够通过主机名解析，dns服务器来解析主机名或host文件来解析（/etc/hosts）
2. /etc/resolv.conf配置dns解析

docker run --name b1 -it --network bridge -h aemon --dns 114.114.114.114 --rm busybox:latest
docker run --name b1 -it --network bridge -h aemon --dns 114.114.114.114 --dns-search ilinux.io --add-host www.baidu.con 123.32.3.4 --rm busybox:latest

```
为了和容器通信，容器需要主动暴露出来。创建容器时用-p命令。
```
-p 1232 指定容器端口1232映射到主机的某个随机端口
-p 111:1232 将容器端口1232映射到主机的111端口
-p 172.30.2.1::1232 将容器端口1232映射到主机ip地址的任意一个端口
-P 或--publish-all 将容器的所有计划要暴露端口全部映射至主机端口

docker run --name b2 --network container:b1 -it busybox //共享容器b1的名称空间
```

默认改docker的网络
```json
#/etc/docker/daemon.json
{
    "bip":"192.168.1.5/24", #桥所属网络
    "fixed-cidr":"10.20.0.0/16",
    "fixed-cidr-v6":"2001:db8::/64",
    "mtu":1500,
    "default-gateway":"10.20.1.1",
    "default-gateway-v6":"2001:db8:abcd::89",
    "dns":["10.20.1.2","10.20.1.3"]

}
最主要是bip，bip更改后可以进行自动计算（其余几个不用配置）。除开dns
```
两个容器之间是不互相通信默认是不可能的，由于docker守护进程的c/s，其默认仅监听unix socker格式的地址（/var/run/docker.sock文件），只支持本地通信。但若使用tcp套接字，通过其他宿主机来访问此容器，加入配置文件
```json
{
    "hosts":["tcp://0.0.0.0:2375","unix:///var/run/docker.sock"]
}
```
在其他宿主机中运行容器
docker -H 172.20.0.67:2375 ps 

怎么自定义创建桥
docker info 可以看见docker支持的网络
```
docker network create bridge --subnet "172.26.0.0/16" --gateway "172.26.0.1" mybr0
```

```
docker run --name b1 --network bridge -it busybox
docker run --name b2 --network mybr0 -it busybox:latest
两个容器通信，在宿主机上打开转发功能
cat /proc/sys/net/ipv4/ip_forward
```

文件的修改，删除效率非常低（涉及到多层）。而针对redis等这种io密集型，实现持久化存储的时候。这样存储系统势必对底层存储系统的性能要求较高。因此要想绕过镜像cow的限制（绕过容器的文件系统的限制），通过使用存储卷实现。把宿主机上的某路径与容器的某路径建立绑定关系，容器上写数据，就是写在宿主机的路径上。两者共享数据内容。宿主对应的目录称为volume。存在问题？容器关联哪些宿主机的目录，下次启动忘记了怎么办？——利用文件来保存启动容器并创建容器的相关配置（这是容器编排的内容）。把数据存储在nfs上面，可以让容器不局限在单机之上。
```
linux 中 mount --bind 能够实现
```
docker的存储卷使用的是宿主机上本地的文件目录，并不能直接使用nfs。docker有两种类型存储卷
- bind mont volume：在宿主机和容器中的路径都需要人工来指定。

- docker managed volume：只需指定容器的目录，宿主机的目录由docker deamon来自行维护。优点：不存在耦合；缺点：人工无法指定。

```
docker run -it --name bbox1 -v /data busybox ---指定容器的目录

docker run -i -v hostdir:volume busybox

docker inspect bbox1--来查看信息mount：source

来进行过滤信息——go 模板
docker inspect -f {{.Mounts}} b2 ---根下的Mounts信息
docker inspect -f {{.NetworSettings.IPAddress}} b2
```
一个宿主机上多个docker可以共享宿主机的同一个目录来挂载，达到了多个docker共享数据。一个docker可以复制其他docker存储卷的设置。————例如joined container（nginx，Tomcat），也需要共享存储卷。

```
docker run -it --name box1 --volume-from bbox1 busybox --复制一个已存在容器的存储卷
```

```
joined container制作：先制作一个基础容器，其他生产力的容器来拷贝上面的配置

docker run --name infracon -it -v /data/volumes/:/data/web/html busybox 

docker run --name nginx --network container:infracon --volumes-from infracon -it busybox

```

单机容器编排工具——docker-compose