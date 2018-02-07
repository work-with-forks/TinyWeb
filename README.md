```
 _____ _           __        __   _     
|_   _(_)_ __  _   \ \      / /__| |__  
  | | | | '_ \| | | \ \ /\ / / _ \ '_ \ 
  | | | | | | | |_| |\ V  V /  __/ |_) |
  |_| |_|_| |_|\__, | \_/\_/ \___|_.__/ 
               |___/                    
```

TinyWeb是一款基于C++11开发的高性能Web服务器（性能还需要测试），基本方式进程池+事件驱动。在开发的时候参考Nginx的设计原理，后添加了自己的想法进行设计。他的优点主要是：高性能，配置方便，占用内存小，可方便进行热部署。实现了Web服务器的基本功能：代理，负载均衡，高性能日志。
本项目在设计的时候注重模块化设计，其中日志功能、配置功能、网络部分均可抽出为其他开发者使用。详情见下文分析。
其中网络部分被抽出作为一个时间驱动式网络库，可以自定义通信逻辑，被应用在另一个独立项目：[Sigil（一个分布式键值数据库）](https://www.github.com/GeneralSandman/Sigil)

- 参考muduo实现高性能事件驱动式网络库
- 参考nginx实现高效进程通信模型
- 参考twisted实现异步式编程


> # 一、如何安装TinyWeb
```
git clone https://www.github.com/GeneralSandman/TinyWeb
cd TinyWeb
make TinyWeb
make clean
```

> # 二、TinyWeb配置方法
名称不区分大小写，数值区分大小写;#开头为注释行
|名称|含义|
|-|-|
|***基本配置***|
|listen|监听端口|
|processpoll|进程池数量|
|Docs|html路径|
|HostName|网址|
|***日志配置***|
|LogLevel|日志等级|
|LogPath|日志路径|
|DebugFile|Debug等级日志的文件名|
|InfoFile|Info等级日志的文件名|
|WarnFile|Warn等级日志的文件名|
|ErrorFile|Error等级日志的文件名|
|FatalFile|Fatal等级日志的文件名|
|***负载均衡***|
|||
|||
|||



> # 三、启动TinyWeb

- 直接启动，默认配置文件为```/TinyWeb.conf```，该文件不存在，或格式错误时均返回错误。

```
sudo ./TinyWeb
```

- 启动时指定配置文件路径,该文件不存在，或格式错误时返回错误。

```
sudo ./TinyWeb -c /home/li/TinyWeb.conf
```

- 命令行控制以Debug模式运行,忽略配置中的等级。

```
sudo ./TinyWeb -d -c /home/li/TinyWeb.conf
```

- 如果日志等级为```Debug```,不会切换至daemon进程，可由终端控制



> # 四.如何进行测试test下的相关例程（主要是单元测试）


- ```server_test``` 监听80端口，600s后终止程序
```
make lib
make server_test
sudo ./server_test
```

- ```client.py``` 作为客户端连接
```
python client.py 9090 139.199.13.50 80
```


------------------



> # 设计难点及解决之道
|难点|解决办法|
|-|-|
|进程间通信机制||
|||
|||
|||



> # 各种成熟与不成熟想法
- 实现一个高效的内存池，为每个TCP连接维持内存池。参考STL实现。
    - 通过成员函数allocate和deallocate分配释放内存。
    - 当需要的内存大于128Bytes时：直接分配内存（第一级内存配置器）。
        - 直接调用malloc，free，
        - 处理内存不足的情况
    - 当小于128Bytes时：使用内存池分配配（第二级内存配置器）。
        - 16种内存大小，链表维护他们，
        - 如果内存不足时转向第一级配置器。
    - 要考虑内存不足时的处理，
    - 考虑造成的内存碎片
    - 效率
- 增加异步读磁盘
- nginx定义了结构体描述接受到信号的行为，定义一个数组注册收到信号的行为。
- 提升程序为demon进程（demon程序标准输入输出如何处理?）
- Semphore,SharedMemory,Cache
- 优化Parser
- 通过list维护一批请求头，响应头
- 如何处理url中包含```../```的问题
- add MIME type
- 为Connection添加close()功能来作为对shutdownWrite()的补充
- 如何支持中文url??????
- 如何使用进程池来处理多个连接
    - master进程负责EventLoop.loop()循环，负责分配连接
    - slaver进程监听与master的socket，接受请求，工作
    - 一个Connection只能属于一个进程，不能多个进程同时处理
    - Connection的读，写，关闭事件均有master进程负责，
        并把相应的事件传给slaver
- EventLoop与ProcessPoll的逻辑关系如何设计？？？？？
- 每个Process均有一个EventLoop，不过监听的event不同
    - master监听网络事件
    - slaver监听master的通信事件，和每个Connection的定时器事件
- 残留的Connection应该由Server释放
- 残留的Protocol应该由Factory释放
- Connection不知道Protocol的存在，Server不知道Factory的存在
- processpoll和process通过底层的信号控制
- 进程间通信用信号和共享内存
    - 信号控制进程的工作，停止
    - 共享内存更偏重于业务逻辑
- fork 之后复制的内存如何销毁的问题！！！！！！！！
- 优化创建进程的方式：通过vfork和exec
- master和worker实际上是对processpoll和process的高层封装
    - 通过对他们的信号进行控制，进而实现对master对worker的控制，
        - 如：平滑启动，强制关闭所有进程，重启的一系列高层决策，
        - processpoll只负责接受命令，分发信号，处理信号
- Nginx封装的锁都不会直接使用信号量,因为一旦获取信号量互斥锁失败,进程
就会进入睡眠状态,这会导致其他请求“饿死”。
- master进程不需要处理网络事件,它不负责业务的执行,只会通过管理worker等子进程
来实现重启服务、平滑升级、更换日志文件、配置文件实时生效等功能。
- 计划完成Client，Connector，Proxy,memorypool类。
- 重读代码，配置各个模块的日志结构。
- 为Factory添加writeCompleteCallback
- Accepter 只有一个，负责所有到来的连接
- Connector 与Accepter不同，只负责一个Connection，
- 因为listen-Socket是可以复用的，而已经创建起连接的Socket是不可复用的，下一个新连接必定要使用新的connect-Socket,他们决定了Accepter和Connector在细节上略有不同。
- 三种工作状态：Server和Proxy
- 为Protocol添加makeConnection()功能，作为客户端使用
- Connector只监听读事件，错误事件
- Connection监听读、写、写完、错误、关闭事件
- Connection和Connector的Channel不同，也就决定了监听对象的不同，Connector监听的是连接建立的时候的事件，而Connection要对连接成功之后的事件进行负责。注意事件范围。
- ConnectSocket 和已经建立连接后的Connection中的socket是相同的
如何处理两者都关闭file descriptor 出错的问题。


> # TinyWeb特点
1. 性能
    - 网络性能：能否处理高并发连接
    - 单次请求延迟：在高并发的情况下维持较低的请求延迟
    - 网络效率：如长连接相比与短连接减少握手次数，提高网络效率
2. 可配置性
3. 可见性：某些关键组件可以被监控

> # 在宏观上需要改进的意向

- defer
- 进程池机制（平滑启动）
- cache
- proxyer
- 负载均衡
- 进程间的消息传递
    - 信号   1 
    - 共享内存  1
    - socketpair    1
- 进程间的同步
    - 原子操作（基于原子变量实现的自旋锁）1
    - 信号量   (会导致睡眠) 实现互斥量 
    - 文件锁   (会导致睡眠)
    - 基于以上三种实现mutex

```
不要随意地使用信号量互斥锁,这会使得
worker进程在得不到锁时进入睡眠状态,从而导致这个worker进程上的其他请求被“饿死”。
鉴于此,Nginx使用原子操作、信号量和文件锁实现了一套ngx_shmtx_t互斥锁,当操作系统
支持原子操作时ngx_shmtx_t就由原子变量实现,否则将由文件锁来实现。顾名思义,
ngx_shmtx_t锁是可以在共享内存上使用的,它是Nginx中最常见的锁。
```

- Nginx各进程间共享数据的主要方式就是使用共享内存(在使用共享内存时,Nginx一般是由master进程创建,在master进程fork出worker子进程后,所有的进程开始使用这块内存中
的数据)。

- 这个消息的格式似乎过于简单了,没错,因为Nginx仅用这个频道同步master进程与worker进程间的状态,这点从针对command成员已经定义的命令就可以看出来,如下所示。
- master又
是如何启动、停止worker子进程的呢?正是通过socketpair产生的套接字发送命令的,即每次要派生一个子进程之前,都会先调用socketpair方法。
- 接收频道消息的套接字添加到epoll中
- 使用信号量作为互斥锁有可能导致进程
睡眠,因此,要谨慎使用,特别是对于Nginx这种每一个进程同时处理着数以万计请求的服
务器来说,这种导致睡眠的操作将有可能造成性能大幅降低。
- 信号量提供的用法非常多,但Nginx仅把它作为简单的互斥锁来使用
- 但Nginx封装的文件锁仅用于保护代码段的顺序执行(例如,在进行负
载均衡时,使用互斥锁保证同一时刻仅有一个worker进程可以处理新的TCP连接),使用方
式要简单得多:一个lock_file文件对应一个全局互斥锁,而且它对master进程或者worker进程
都生效。因此,对于l_start、l_len、l_pid,都填为0,而l_whence则填为SEEK_SET,只需要
这个文件提供一个锁。l_type的值则取决于用户是想实现阻塞睡眠锁还是想实现非阻塞不会
睡眠的锁。

- nginx进程间的通信机制
    -

> # 更新进度（2.7-2.11）
- 对进程间通信的基本类进行设计
    - 信号量
    - 文件锁
    - 共享内存
- 对进程池进行重新设计，利用IPC基本类进行通信，控制

> # 更新进度
- 参考nginx内存池，list,rbtree,string,
- 思考mime.type的设计，如何配置 include /home/li/TinyWeb/mime.type
- 重新写Makefile
> # 模块的开发文档

> # 需要处理的信号：
- SIGCHLD 子进程终止通知父进程
- SIGPIPE 管道异常
- SIGINT Ctrl-c
- SIGTERM　
- SIGHUP 终端退出信号

各种类模块的说明：
- distingush Protocol & Connection
    - Protocol 用来处理用户逻辑，简单的维护用户上下文，不负责数据传输，
        通过简单的重载basic Protocol的共有函数来实现处理Connection读到
        Buffer中的数据。
    - Connection 用来与底层的IO-Multiplexing 进行交互，负责数据的传输，
        维护者数据缓冲，监听着各种连接事件
    - 简单来说：Connection负责数据传输，不处理用户逻辑；Protocol不处理数据传输，
     负责处理逻辑



> # test目录下相关测试的说明

> # multiwebserver目录下web服务器各种版本的说明。
|目录|版本|
|-|-|
||单进程+阻塞IO|
||多进程+阻塞IO|
||单进程多线程+阻塞IO|
||单进程+epoll+非阻塞IO|
||进程池+epoll+非阻塞IO|
||单进程多线程+epoll+非阻塞IO|


> # 用户可重用或可更改部分的操作方法

> ## 1.Configer的使用方法

- 获得Configer实例（配置项还未加载，需调用loadConfig()）
```
Configer &getConfigerInstance(const std::string file = "")
```

- 调整confige文件
```
setConfigerFile(const std::string &file)
```

- 装载配置文件（不需调用，setConfigerFile会装载配置文件）
```
loadConfig()
```

- 获取配置项数值
```
std::string getConfigValue(const std::string &);
```

- 注意

这样会出错：
```
Configer configer = Configer::getConfigerInstance();
```

必须用引用接受：
```
Configer &configer = Configer::getConfigerInstance();
```

推荐用法 1：
```
std::string a = "../TinyWeb.conf";
setConfigerFile(a);
if (loadConfig())
    std::cout << "load config successfull\n";
else
    std::cout << "load config failed\n";
Configer::getConfigerInstance().test();
cout << getConfigValue("loglevel");
```

使用方法 2：
```
Configer &configer = Configer::getConfigerInstance();
configer.setConfigerFile("../TinyWeb.conf");
if (configer.loadConfig())
    std::cout << "load config successfull\n";
else
    std::cout << "load config failed\n";
configer.test();
cout << configer.getConfigValue("loglevel");
```

```

```


> ## 2. Protocol 使用方法

- 如果想要自定义协议类，只需要继承Protocol,并重载相关函数（见WebProtocol定义）


--------------


# 参考文献：

1.APUE

2.UNP卷1

3.UNP卷2

4.C++ Primer

5.[Linux 多线程服务端编程：使用 muduo C++ 网络库](https://github.com/chenshuo/documents)

6.[C++ 官方参考手册](http://en.cppreference.com/w/cpp)

7.[高性能服务器编程](http://blog.csdn.net/column/details/high-perf-network.html)
