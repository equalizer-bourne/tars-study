# Tars简介

在腾讯内部Taf是一个使用非常广泛的服务治理框架，但对于外部开发者来说关于Tars的资料比较少，主要还是基于[官方仓库](https://github.com/TarsCloud/Tars/blob/master/README.zh.md)的内容来进行学习。
下面我们对Tars整体概况和安装部署进行简单的记录(方便起见部分内容直接摘抄自官方文档)。

## 项目整体结构
从仓库的结构来看，TarsFramework是Tars的核心能力层，节点管理、服务路由等功能都是在这里实现的，TarsWeb实现了基于web界面的服务控制台，用户貌似可以在这里发布和管理服务，剩下比较核心的模块就是针对各个语言封装好的开发环境，比如TarsCpp、TarsJava等等。具体可以看这里[https://github.com/TarsCloud/Tars/blob/master/README.zh.md#%E5%AD%90%E9%A1%B9%E7%9B%AE](https://github.com/TarsCloud/Tars/blob/master/README.zh.md#%E5%AD%90%E9%A1%B9%E7%9B%AE)

## 系统设计介绍
[https://github.com/TarsCloud/Tars/blob/master/Introduction.md](https://github.com/TarsCloud/Tars/blob/master/Introduction.md)

### 顶层设计
![aa](https://github.com/TarsCloud/Tars/raw/master/docs/images/tars.png)

### 系统架构
![aa](https://github.com/TarsCloud/Tars/raw/master/docs/images/tars_tuopu.png)

整体架构的拓扑图主要分为2个部分：服务节点与公共框架节点。
服务节点:
服务节点可以认为是服务所实际运行的一个具体的操作系统实例，可以是物理主机或者虚拟主机、云主机。随着服务的种类扩展和规模扩大，服务节点可能成千上万甚至数以十万计。 每台服务节点上均有一个Node服务节点和N(N>=0)个业务服务节点，Node服务节点会对业务服务节点进行统一管理，提供启停、发布、监控等功能，同时接收业务服务节点上报过来的心跳。

公共框架节点:
除了服务节点以外的服务，其他服务节点均归为一类。
公共框架节点，数量不定，为了自身的容错容灾，一般也要求在在多个机房的多个服务器上进行部署，具体的节点数量，与服务节点的规模有关，比如，如果某些服务需要打较多的日志，就需要部署更多的日志服务节点。
又可细分为如下几个部分：
Web管理系统：在Web上可以看到服务运行的各种实时数据情况，以及对服务进行发布、启停、部署等操作；
Registry（路由+管理服务）：提供服务节点的地址查询、发布、启停、管理等操作，以及对服务上报心跳的管理，通过它实现服务的注册与发现；
Patch（发布管理）：提供服务的发布功能；
Config（配置中心）：提供服务配置文件的统一管理功能；
Log（远程日志）：提供服务打日志到远程的功能；
Stat（调用统计）：统计业务服务上报的各种调用信息，比如总流量、平均耗时、超时率等，以便对服务出现异常时进行告警；
Property（业务属性）：统计业务自定义上报的属性信息，比如内存使用大小、队列大小、cache命中率等，以便对服务出现异常时进行告警；
Notify（异常信息）：统计业务上报的各种异常信息，比如服务状态变更信息、访问db失败信息等，以便对服务出现异常时进行告警；


> 物理机上运行着业务封装的服务节点(Server)，并且还有NodeServer用来对业务服务进行管理;
> Tars全局的核心功能是被封装到公共框架模块中

![](https://github.com/TarsCloud/Tars/raw/master/docs/images/tars_jiaohu.png)

框架服务在整个系统中运行时，服务之间的交互分：业务服务之间的交互、业务服务与框架基础服务之间的交互。

服务发布流程：在Web系统上传server的发布包到patch，上传成功后，在web上提交发布server请求，由registry服务传达到node，然后node拉取server的发布包到本地，拉起server服务。

管理命令流程：Web系统上的可以提交管理server服务命令请求，由registry服务传达到node服务，然后由node向server发送管理命令。

心跳上报流程：server服务运行后，会定期上报心跳到node，node然后把服务心跳信息上报到registry服务，由registry进行统一管理。

信息上报流程：server服务运行后，会定期上报统计信息到stat，打印远程日志到log，定期上报属性信息到property、上报异常信息到notify、从config拉取服务配置信息。

Client访问Server流程：client可以通过server的对象名Obj间接访问server，Client会从registry上拉取server的路由信息（如ip、port信息），然后根据具体的业务特性（同步或者异步，tcp或者udp方式）访问server(当然client也可以通过ip/port直接访问server)。

> 也就是说服务节点会注册其监听的端口等信息，调用方从registry拿到路由之后直接走网络调用

### 客户端-服务端交互过程
![](https://github.com/TarsCloud/Tars/raw/master/docs/images/tars_server_client.png)

服务端：
NetThread： 收发包，连接管理，多线程(可配置），采用epoll ET触发实现，支持tcp/udp；
BindAdapter： 绑定端口类，用于管理Servant对应的绑定端口的信息操作；
ServantHandle：业务线程类，根据对象名分派Servant的对象和接口调用；
AdminServant： 管理端口的对象；
ServantImp： 继承Servant的业务处理基类（Servant：服务端接口对象的基类）；

客户端：
NetThread： 收发包，连接管理，多线程(可配置），采用epoll ET触发实现，支持tcp/udp；
AdapterProxy： 具体服务器某个节点的本地代理，管理到服务器的连接，以及请求超时处理；
ObjectProxy： 远程对象代理，负责路由分发、负载均衡、容错，支持轮询/hash/权重；
ServantProxy： 远程对象调用的本地代理，支持同步/异步/单向，Tars协议和非Tars协议；
AsyncThread： 异步请求的回应包处理线程；
Callback： 具体业务Callback的处理基类对象；

> 对于业务开发来说，理论上只需实现ServantImp(服务端业务逻辑)和Callback(客户端接收到响应之后的业务逻辑)即可接入Tars框架

### tars协议

tars协议采用接口描述语言（Interface description language，缩写IDL）来实现，它是一种二进制、可扩展、代码自动生成、支持多平台的协议，使得在不同平台上运行的对象和用不同语言编写的程序可以用RPC远程调用的方式相互通信交流， 主要应用在后台服务之间的网络传输协议，以及对象的序列化和反序列化等方面。

协议支持的类型分两种，基本类型和复杂类型。
基本类型包括：void、bool、byte、short、int、long、float、double、string、unsigned byte、unsigned short、unsigned int；
复杂类型包括：enum、const、struct、vector、map，以及struct、vector、map的嵌套。

> 服务端通过IDL来声明服务;
> 网络对接通信的代码是由框架自动实现的;
> 貌似客户端通过stringToProxy()方法调用来从Registry获得服务端接口的代理

## 框架安装&部署

### 运行环境配置及框架部署
step by step安装流程: [https://github.com/TarsCloud/Tars/blob/master/Install.zh.md](https://github.com/TarsCloud/Tars/blob/master/Install.zh.md)

一键部署方式: [https://github.com/TarsCloud/Tars/tree/master/deploy](https://github.com/TarsCloud/Tars/tree/master/deploy)

最终我们采用第二种方案成功拉起框架核心服务，简单来说:

1. 安装mysql, 本人使用的是5.6.43版本
2. 安装开发依赖，简单起见直接install开发全家桶即可
```
yum -y groupinstall "Development tools"
```
3. 把Tars的deploy模块代码[https://github.com/TarsCloud/Tars/tree/master/deploy](https://github.com/TarsCloud/Tars/tree/master/deploy) 拉到本地
4. 修改comm.properties文件，把本机安装好的MySQL的root账号密码填进去
5. 然后运行以下命令即可
```
	cd /data
	git clone https://github.com/TarsCloud/Tars.git --recursive
	cd /data/Tars/deploy
	python ./deploy.py all
```

执行成功后可以访问 [http://localhost:3000](http://localhost:3000) ，此时应该能进入到Tars的console界面

![](https://github.com/TarsCloud/Tars/raw/master/docs/images/tars_web_system_index.png)

> 说实话在安装过程中遇到了一些麻烦，单单clone源码的过程就很艰辛，好几次尝试之后才成功。最开始参考了那篇install说明文档，需要手动下载一大坨依赖，略显蛋疼，后来读到一键部署说明文档，明显能减轻开发者的负担，但在尝试过程中也遇到一些小问题，比如mysql账号权限不足等等情况，可能是跟本机配置有关，我编译过程中有部分模块花费了巨长的时间，好像要将近两个小时，简直崩溃。

### 开发环境搭建
这里以C++语言为例一起看下怎么开发并发布上线我们的业务Server节点。
一键部署脚本仅安装了Tars各项核心服务(registry、node等)，没有安装c++开发环境，为了编写tars服务，我们需要自己把它安装到本地。
过程相对简单一些，先把代码clone到本地，然后编译安装即可:
```
git clone https://github.com/TarsCloud/TarsCpp.git --recursive
cd TarsCpp
cmake .
make
make install
```

install结束之后c++开发环境就会被安装到/usr/local/tars/cpp目录中

#### 服务命名规则
使用Tars框架的服务，其的服务名称有三个部分：

APP： 应用名，标识一组服务的一个小集合，在Tars系统中，应用名必须唯一。例如：TestApp；

Server： 服务名，提供服务的进程名称，Server名字根据业务服务功能命名，一般命名为：XXServer，例如HelloServer；

Servant：服务者，提供具体服务的接口或实例。例如:HelloImp；

说明：

一个Server可以包含多个Servant，系统会使用服务的App + Server + Servant，进行组合，来定义服务在系统中的路由名称，称为路由Obj，其名称在整个系统中必须是唯一的，以便在对外服务时，能唯一标识自身。

因此在定义APP时，需要注意APP的唯一性。

例如：TestApp.HelloServer.HelloObj。


