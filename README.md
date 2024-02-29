# mprpc
## 项目介绍
该项目是一个基于muduo、Protobuf和Zookeeper实现的轻量级分布式RPC网络通信框架。

可以把任何单体架构系统的本地方法调用，重构成基于TCP网络通信的RPC远程方法调用，实现同一台机器的不同进程之间的服务调用，或者不同机器之间的服务调用，适用于把单体架构系统拆分成基于分布式微服务调用进行部署。



## 项目特点
- 基于muduo网络库实现高并发网络通信模块，作为RPC远程调用的基础。
- 基于Protobuf实现RPC方法调用中方法和参数的序列化和反序列化，并定义网络通信中数据的传输格式(header_size(4字节长度信息) + service_name method_name args_size(header,服务名、方法名、参数长度,参数长度用于解决粘包问题) + args(RPC方法调用所需的参数))。
- 基于ZooKeeper分布式协调服务中间件提供服务注册和服务发现功能。
- 基于生产者消费者模型，设计了线程安全的缓冲队列，实现了异步工作方式的日志模块。

## 项目工程目录
> bin: 可执行文件

> build: 项目编译文件

> lib: 项目库文件

> src: 源文件

> test: 测试代码

> example: 框架代码使用范例

> CMakeLists.txt: 顶层的cmake文件

> README.md: 项目自述文件

> autobuild.sh: 一键编译脚本

## 项目演示

项目构建，运行自动编译脚本，生成可执行程序。

```bash
chmod +x autobuild.sh #给自动编译脚本添加可执行权限
./autobuild.sh #启动自动编译脚本

# 执行完后，会在当前目录的bin目录下生成可执行文件
[Joy@VM-12-10-centos mprpc]$ tree ./bin/
./bin/
|-- consumer
|-- mprpc.conf
`-- provider

0 directories, 3 files

# mprpc.conf 是 rpc 节点和 Zookeeper 的配置信息
[Joy@VM-12-10-centos mprpc]$ cat ./bin/mprpc.conf 
# rpc节点的ip地址
rpcserverip=127.0.0.1
# rpc节点的端口号
rpcserverport=8080
# zookeeper的ip地址
zookeeperip=127.0.0.1
# zookeeper的端口号
```

启动 Zookeeper，我们需要再 Zookeeper 上获取注册的服务消息，因此先需要保证启动了 Zookeeper。进入到 Zookeeper 的目录，我的 Zookeeper 目录是在`/home/Joy/install/zookeeper-3.4.10`。在里面的`bin`目录下有客户端和服务端的启动脚本，先启动 Zookeeper 服务端，再启动客户端。启动客户端主要是为了看我们的 RPC 框架是否插入新服务信息。

```bash
./zkServer.sh start #启动 Zookeeper 服务端
./zkCli.sh #启动 Zookeeper 客户端
```

![1](https://gitee.com/DreamyHorizon/mprpc/raw/master/image/1_20230821225819.png)

启动服务端

```bash
./provider -i mprpc.conf
```

我们观察下打印的信息，可以看到它打印了 `mprpc.conf` 配置文件的信息，至少说明 RPC 框架读取配置是成功的。

![1](https://gitee.com/DreamyHorizon/mprpc/raw/master/image/2_20230821231918.png)

重点是下面的信息，其显示在 Zookeeper 上注册了服务。

![3_20230821232424](https://gitee.com/DreamyHorizon/mprpc/raw/master/image/3_20230821232424.png)

启动客户端

```bash
./consumer -i mprpc.conf
```

![4_20230821233410](https://gitee.com/DreamyHorizon/mprpc/raw/master/image/4_20230821233410.png)

有很多的提示信息，也是解析了配置文件，并且有许多 Zookeeper 相关日志信息。这里注意最重要的几个地方，它打印显示了 `rpc Login response : 1`。响应为 1，RPC 方法调用成功！

## RPC框架的设计

这个项目是基于 Protobuf 和 Zookeeper 的 RPC 框架，那么 Protobuf 和 Zookeeper 又在整个框架中扮演怎样的角色呢？

## Protobuf的作用

Protobuf 主要是作为整个框架的传输协议。我们可以看一下整个框架对于传输信息的格式定义：

```protobuf
syntax = "proto3";

package mprpc;

message RpcHeader 
{
    bytes serviceName = 1;  // 类名
    bytes methodName = 2;   // 方法名
    uint32 argsSize = 3;    // 参数长度（参数序列化后的长度）
}
```

可以看到，它**定义了要调用方法是属于哪个类的哪个方法以及这个方法所需要的的参数长度**。

那怎么进行使用呢？我们来看一下我们框架内传输的数据是什么：**4字节标识头部长度 headerSize + headerStr + argsStr。其中，headSize 描述的是 headStr 的长度，headerStr 包括 serviceName、methodName 和 argsSize，argsSize 描述的是 agrsStr 的长度。**

举个例子来帮助大家理解一下，假如现在有人调用 UserService 的 Login 方法，Login 方法的参数是用户名`zhang san`和密码`123456`。

那么经过 Protobuf 的序列化后，就会得到`18UserServiceLogin15zhang san123456`。

- 18 等于 UserService + Login + 15，分别对应着 serviceName、methodName 和 argsSize。
- 15 等于户名`zhang san`和密码`123456`的长度之和。
- 因此拿到序列化后的字符串，我们先读取前四个字节的数据，拿到 headerSize 的大小。然后再读取 headerSize 大小的数据，对这个数据进行反序列化就能够拿到serviceName、methodName 和 agrsSize了，再根据 argsSize 读取出 rpc 方法所需要的参数。

## Zookeeper的作用

Zookeeper 在本项目中主要是起到了一个**配置中心**的作用。

什么意思呢？

Zookeeper上面我们标识了每个类的方法所对应的**分布式节点地址**，当我们其他服务器或客户端想要 RPC 的时候，就去 Zookeeper 上面查询对应要调用的服务在哪个节点上。如果该服务在 Zookeeper 上注册过，那么 Zookeeper 将会返回该方法所在的 IP 地址和端口号。

Zookeeper 所起的作用就是**服务注册和服务发现**。

## 从框架的使用来认识

框架的使用一般都是在 example 目录下的 `callee/UserService.cpp` 里面

```c++
// 调用框架的初始化操作 provider -i config.conf
MprpcApplication::Init(argc, argv);

// provider是一个rpc网络服务对象，把UserServcie对象发布到rpc节点上
RpcProvider provider; 
provider.NotifyService(new UserService());

// 启动一个rpc服务发布节点，Run以后，进程进入阻塞状态，等待远程的rpc调用请求
provider.Run();
```

可以看到，主要去做了三个事情：

- 首先 RPC 框架肯定是部署到一台服务器上的，所以我们需要对这个服务器的 ip 和 port 对 RPC 框架进行初始化。
- 然后创建一个 porvider（也就是 server）对象，将当前 UserService 这个对象传递给他，也就是对 UserService 这个服务的所有的 RPC 方法都保存到 provider 对象中的 Map 表中记录起来。
- 最后就是让这个 provider 去 Run 起来，也就是将 RPC 方法发布到 Zookeeper 上，并启动 RPC 服务。

```C++
// 启动rpc服务节点，开始提供rpc远程网络调用服务
void RpcProvider::Run()
{
    std::string ip = MprpcApplication::getInstance().getConfig().Load("rpcserverip");
    uint16_t port = atoi(MprpcApplication::getInstance().getConfig().Load("rpcserverport").c_str());
    muduo::net::InetAddress address(ip, port);

    // 创建TcpServer对象
    muduo::net::TcpServer server(&_eventLoop, address, "RpcProvider");
    // 绑定连接回调和消息读写回调方法(分离了网络代码和业务代码)
    server.setConnectionCallback(std::bind(&RpcProvider::onConnection, this, std::placeholders::_1));
    server.setMessageCallback(std::bind(&RpcProvider::onMessage, this, std::placeholders::_1, 
            std::placeholders::_2, std::placeholders::_3));

    // 设置muduo库的线程数量
    server.setThreadNum(3);

    // 把当前rpc节点上要发布的服务全部注册到zookeeper上，让rpc client可以从zookeeper上发现服务
    // session timeout超时时间是30秒，zkclient的网络I/O线程会以三分之一的timeout超时时间发送心跳消息
    ZkClient zkCli;
    zkCli.Start(); // 连接zookeeper server
    // serviceName为永久性节点  methodName为临时性节点
    for(auto& servicePair : _serviceMap)
    {
        /****** /serviceName（服务节点 /UserServiceRpc） ******/   
        std::string servicePath = "/" + servicePair.first;
        zkCli.Create(servicePath.c_str(), nullptr, 0);
        for(auto& methodPair : servicePair.second._methodMap)
        {
            /********  /serviceName/methodName（方法节点 /UserServicePrc/Login） *********/ 
            // 存储的是当前这个rpc服务节点主机的ip地址和端口号
            std::string methodPath = servicePath + "/" + methodPair.first;
            char methodPathData[128] = {0};
            sprintf(methodPathData, "%s:%d", ip.c_str(), port);
            // ZOO_EPHEMERAL表示znode是一个临时性节点
            zkCli.Create(methodPath.c_str(), methodPathData, strlen(methodPathData), ZOO_EPHEMERAL); 
        }
    }

    std::cout << "RpcProvider start service at ip: " << ip << " port: " << port << std::endl;

    // 启动网络服务
    server.start();
    _eventLoop.loop();
}
```

可以看到，整个 Run 其实就是干了这么几件事情：

- 因为底层调用的是muduo网络库，所以这里会获取ip地址和端口号，然后初始化网络层。
- 然后去设置一个连接回调以及发生读写事件时候的回调函数。
- 然后设置整个 muduo 网络库工作的线程数量。
- 然后创建 Zookeeper 客户端，与 Zookeeper 服务端建立连接，将这些方法的信息以及本机的 IP 地址注册到 Zookeeper。
- 然后开启本机服务器的事件循环，等待其他服务器或客户端的连接。

## 其他服务器或客户端是怎样调用的呢

我们看一下 example 目录下的 caller/CallUserService.cpp 里面是怎样调用的。

```C++
// 整个程序启动后，想使用mprpc框架来享受rpc服务调用
// 一定要先调用框架的初始化函数，只初始化一次
MprpcApplication::Init(argc, argv);

// 调用远程发布的rpc方法Login
Joy::UserServiceRpc_Stub stub(new MprpcChannel());
// rpc方法的请求参数
Joy::LoginRequest request;
request.set_name("zhang san");
request.set_pwd("123456");
// rpc方法的响应
Joy::LoginResponse response;

// RpcChannel->callMethod 集中来做所有rpc方法调用的参数序列化和网络发送
MprpcController controller;
stub.Login(&controller, &request, &response, nullptr); 

// 读取rpc方法的响应结果
if(controller.Failed()) // rpc方法调用失败
{
    std::cout << controller.ErrorText() << std::endl;
}
else
{
    if(response.result().errcode() == 0)
    {
        std::cout << "rpc login response: " << response.success() << std::endl;
    }
    else
    {
        std::cout << "rpc login error: " << response.result().errmsg() << std::endl;
    }
}
```

同样，也是有以下几个步骤：

- 初始化 RPC 框架
- 定义一个 UserSeervice 的 stub 桩类，由这个桩类去调用Login方法，这个 Login 方法可以去看一下源码的定义：

```C++
// 处理业务的方法
bool Login(std::string name, std::string pwd)
{
    std::cout << "do local service Login" << std::endl;
    std::cout << "name: " << name << " pwd: " << pwd << std::endl;

    return true;
}

/*
* 重写基类UserServiceRpc的虚函数，下面的这些方法都是框架直接调用的
* 1. caller(调用者) ===> Login(LoginRequest) ===> muduo ===> callee
* 2. callee(提供者) ===> Login(LoginRequest) ===> 交到下面重写的这个Login方法了
*/ 
void Login(::google::protobuf::RpcController* controller,
        const ::Joy::LoginRequest* request,
        ::Joy::LoginResponse* response,
        ::google::protobuf::Closure* done)
{
    // 框架给业务上报了请求参数LoginRequest，应用获取相应的数据做本地业务
    std::string name = request->name();
    std::string pwd = request->pwd();

    // 做本地业务
    bool ret = Login(name, pwd); 

    // 把响应写入 包括错误码、错误消息、返回值
    Joy::ResultCode* code = response->mutable_result();
    code->set_errcode(0);
    code->set_errmsg("");
    response->set_success(ret);

    // 执行回调操作 执行响应对象数据的序列化和网络发送，这些都是由框架完成的
    done->Run();
}
```

可以看到，Login 的 RPC 重载函数有四个参数：**controller（表示函数是否出错）、request（参数）、response（返回值）、done（回调函数）** 其主要做的也是去围绕着解析参数，将参数放入本地调用的方法，将结果返回并执行回调函数。至于这个回调函数则是在服务端执行读写事件回调函数绑定的。

绑定的方法如下：

```C++
// Closure的回调操作，用于序列化rpc的响应和网络发送
void RpcProvider::sendRpcResponse(const muduo::net::TcpConnectionPtr& conn, google::protobuf::Message* response)
{
    // response进行序列化
    std::string responseStr;
    if(response->SerializeToString(&responseStr))
    {
        // 序列化成功，通过网络将rpc方法执行的结果发送给rpc的调用方
        conn->send(responseStr);
    }
    else
    {
        std::cout << "serialize response error!" << std::endl;
    }
    conn->shutdown(); // 模拟http的短连接服务，由RpcProvider主动断开连接
}
```

它是将 RPC 方法调用的结果 responseStr 发送给 RPC 调用方。

## 桩类是干嘛的

桩类 UserServiceRpc_Stub 是 Protobuf 帮我们自动生成的一个类。

```C++
class UserServiceRpc_Stub : public UserServiceRpc {
 public:
  UserServiceRpc_Stub(::google::protobuf::RpcChannel* channel);
  UserServiceRpc_Stub(::google::protobuf::RpcChannel* channel,
                   ::google::protobuf::Service::ChannelOwnership ownership);
  ~UserServiceRpc_Stub();

  inline ::google::protobuf::RpcChannel* channel() { return channel_; }

  // implements UserServiceRpc ------------------------------------------

  void Login(::google::protobuf::RpcController* controller,
                       const ::ik::LoginRequest* request,
                       ::ik::LoginResponse* response,
                       ::google::protobuf::Closure* done);
  void Register(::google::protobuf::RpcController* controller,
                       const ::ik::RegisterRequest* request,
                       ::ik::RegisterResponse* response,
                       ::google::protobuf::Closure* done);
 private:
  ::google::protobuf::RpcChannel* channel_;
  bool owns_channel_;
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(UserServiceRpc_Stub);
};
```

其实 UserServiceRpc_Stub 是一个代理类，我们需要传入 RpcChannel 指针进行初始化。但由于 RpcChannel 是一个抽象类，所以我们需要实现一个继承抽象类 RpcChannel 的子类 MprpcChannel，重写抽象类的虚函数。将 MprpcChannel 的指针传入给桩类 UserServiceRpc_Stub，当我们去调用这个桩类的`Login`方法的时候，会去调用传递进来的 channel 的`CallMethod`方法：

```C++
void UserServiceRpc_Stub::Login(::google::protobuf::RpcController* controller,
                              const ::ik::LoginRequest* request,
                              ::ik::LoginResponse* response,
                              ::google::protobuf::Closure* done) {
  channel_->CallMethod(descriptor()->method(0),
                       controller, request, response, done);
}
```

所以，我们的 RPC 请求的发送以及响应的接收都是在 CallMethod 这个方法里面进行的。

```C++
/**
 * headerSize + headerStr(serviceName methodName argsSize) + agrsStr
*/
// 所有通过stub代理对象调用的rpc的方法，都会来到这里，
// 统一做rpc方法调用的数据序列化和网络发送
void MprpcChannel::CallMethod(const google::protobuf::MethodDescriptor* method,
                        google::protobuf::RpcController* controller, 
                        const google::protobuf::Message* request,
                        google::protobuf::Message* response, 
                        google::protobuf::Closure* done)
{
    const google::protobuf::ServiceDescriptor* sd = method->service();
    std::string serviceName = sd->name();
    std::string methodName = method->name();

    // 获取参数的序列化字符串长度
    uint32_t argsSize = 0;
    std::string argsStr;
    if(request->SerializeToString(&argsStr))
    {
        argsSize = argsStr.size();
    }
    else
    {
        controller->SetFailed("serialize request error!");
        return;
    }

    // 定义rpc请求的header
    mprpc::RpcHeader rpcHeader;
    rpcHeader.set_servicename(serviceName);
    rpcHeader.set_methodname(methodName);
    rpcHeader.set_argssize(argsSize);
    
    uint32_t headerSize = 0;
    std::string headerStr;
    if(rpcHeader.SerializeToString(&headerStr))
    {
        headerSize = headerStr.size();
    }
    else
    {
        controller->SetFailed("serialize rpcHeader error!");
        return;
    }

    // 组织待发送的rpc请求的字符串
    std::string sendRpcStr;
    sendRpcStr.insert(0, std::string((char*)&headerSize, 4)); // 前4个字节以二进制的形成存储
    sendRpcStr += headerStr;
    sendRpcStr += argsStr;

    // 使用TCP编程，完成rpc方法的远程调用
    int clientFd = socket(AF_INET, SOCK_STREAM, 0);
    if(clientFd == -1)
    {
        controller->SetFailed("create clientFd error: " + std::string(strerror(errno)));
        return;
    }

    // rpc调用方想要调用serviceName的methodName的服务，需要查询在zookeeper上该服务的host信息
    ZkClient zkCli;
    zkCli.Start(); // 与zookeeper server 建立连接
    /******   /UserService/Login   ******/
    std::string methodPath = "/" + serviceName + "/" + methodName;
    /* hostData ==> 127.0.0.1:8080 */ 
    std::string hostData = zkCli.getData(methodPath.c_str());
    if(hostData == "")
    {
        controller->SetFailed(methodPath + " does not exist!");
        return;
    }
    size_t index = hostData.find(":");
    if(index == std::string::npos)
    {
        controller->SetFailed(methodPath + " address is invalid!");
        return;
    }
    std::string ip = hostData.substr(0, index);
    uint16_t port = atoi(hostData.substr(index + 1, hostData.size() - index).c_str());

    struct sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(port);
    serverAddr.sin_addr.s_addr = inet_addr(ip.c_str());

    if(connect(clientFd, (struct sockaddr*)&serverAddr, sizeof(serverAddr)) == -1)
    {
        controller->SetFailed("connect error: " + std::string(strerror(errno)));
        close(clientFd);
        return;
    }

    // 发送rpc请求
    if(send(clientFd, sendRpcStr.c_str(), sendRpcStr.size(), 0) == -1)
    {
        controller->SetFailed("send error: " + std::string(strerror(errno)));
        close(clientFd);
        return;
    }

    // 接收rpc请求的响应
    char recvBuf[1024] = {0};
    uint32_t recvSize = 0;
    if((recvSize = recv(clientFd, recvBuf, 1024, 0)) == -1)
    {
        controller->SetFailed("recv error: " + std::string(strerror(errno)));
        close(clientFd);
        return;
    }

    // 反序列化rpc调用的响应数据
    if(!response->ParseFromArray(recvBuf, recvSize))
    {
        controller->SetFailed("parse responseStr error: " + std::string(recvBuf));
        close(clientFd);
        return;
    }

    close(clientFd);
}
```

- 组织要发送的 sendRpcStr 字符串。
- 从 Zookeeper 中拿到服务端的 ip 和 port，连接服务端。
- 发送 sendRpcStr 字符串。
- 接收服务端返回过来的 response 对象并进行反序列化得到相应。

## RPC方法调用总体流程

![image](https://github.com/IceHowe/mpRPC/blob/main/%E9%A1%B9%E7%9B%AE%E4%BB%A3%E7%A0%81%E4%BA%A4%E4%BA%92%E5%9B%BE-%E7%94%A8%E7%94%BB%E5%9B%BE%E6%9D%BF%E6%89%93%E5%BC%80.png)

