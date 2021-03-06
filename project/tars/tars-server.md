# server端代码分析

## 主要类简介

- Application

    - 我们实现的helloServer需要继承自这个类。主要承载配置的解析、epollser的初始化、消息的处理和分发、servant的创建和管理。
    - 其中最主要的两个成员： EpollServer、ServantHelperManager

- TC_EpollServer

    - 接收新的连接
    - 负载所有io
    - 消息分发到servant
  
- ServantHelperManager

  - helloServer初始化的时候会通过addServant添加特例话的helloServant。servantHelper的主要工作就是提供create接口方便创建helloServant实例。ServantHelperManager就是通过ServantName管理这些ServantHelper。

- BindAdapter

  - 监听适配器。每个bindAdaper对应一个监听的端口，并且维护一个消息队列。用于支撑不同的端口不同的协议。
  - 提供Handler的初始化能力。

- NetThread

  - 承载分配到本线程的connection的io。内部依靠epoller来驱动。所谓的one sthread one looper。

- Connection

  - 维护连接的属性，是udp还是tcp

- TC_Transceiver

  - 处理数据的收发和socket的链接，这样可以排除平台的差异。具体的协议协议介系需要通过回调支持。

- Handle

  - 负责对消息的处理，分发到不同的servatn。servant在分发到不同的method。


## 初始化流程

- application初始化

  - 配置解析::parseConfig

    这里没什么需要特别说明的，就是从服务器的配置文件中解析配置。初始化单利配置实例，并进行托管。

  - 客户端部分初始化::initializeClient

    由于我们的服务节点需要同nodeServer进行通信，回报自己的状态。所以，需要初始化客户端部分，也就是Communicator（详情见**tars源码分析-客户端部分**）。

  - 服务器部分初始化::initializeServer

    - 初始化epollServer实例，详见**epollServer初始化**。
    - 实例化bindAdapter，创建bindAdapter的同时会根据bindAdatperName创建ServantHandle，servantHandle数量由配置决定，默认为1。
    - 初始化与nodeServer、notifyServer、logServer、configServer通信的各种远程通信客户端部分的实例， 填充配置。
    - 初始化AdminServant用于接受远程控制指令。例如，tars通过后台修改配置并进行分发的功能就是通过adminServant和baseNotify实现的。

  - bindAdapter初始化::bindAdapter

    - 根据配置创建对应数量的bindAdapter实例，配置消息队列长度、连接数量上限、协议解析回调、超时时间等。
    - 创建配置数量的ServantHandler。对应的就是配置中XXX.obj

  - 用户服务实例的初始化::initialize

    - helloServer中的initialize接口会被调用。主要讲用户实现的helloimpl添加到application的ServantHelperManager中进行管理，需要的时候再实例话。
    - application::waitForShutdown中，会完成servant实例的创建。

  - 绑定控制命令

    - 绑定了一些控制命令，这些命令可以用来支撑通过后台控制服务节点的目的。关联的类为AdminServant。
    - adminSservant接受到远程控制命令以后，介系common和参数，然后交给notifyObserver，最后会调用实际的控制命令函数
    - 控制命令函数通过TARS_ADD_ADMIN_CMD_PREFIX注册到baseNotify的_procFunctors中。application继承与这个类。

- epollServer初始化

    - epellServer的初始化分：一部分在initializeServer；另一部分在waitForShutdonw。
    - 第一部分
      - 配置io线程数量
      - 绑定bindAdapter
      - 绑定accept回调
    - 第二部分
      - 创建并且启动io线程，详见**handle初始化**
      - 创建并且启动handle线程，详见**handle初始化**
      - 关于handle线程的创建需要根据模式决定，只有非merge模式的才会创建handle线程。merge模式会将handle的处理放到io线程中。

- NetThread初始化
  - inithandle
  
    - 如果是merge模式线程数量根据servantHandle的配置数量和bindAdapter的数量共同决定。如果不是merge模式，根绝threadNum决定。
    - 另一个差异为merge模式除了会为每个bindAdapter绑定对应数量的io线程组，还会为每个servantHanle都会绑定一个线程；
    - 相同之处是所有的servantHandler都会绑定bindAdapter的dataBuffer。这里需要注意每个bindAdapter都会对应一组属于自己的ServantHandler。

  - starthandle

    - 没什么特别的，启动上面创建的所有io线程。非merge模式，出了要启动io线程，还要启动handle线程。
    - 启动handle线程需要说明一点。如果是merge模式，handle的实际执行函数为handleOnceThread或handleOnceCoroutine。通过setInitializeHandle接口添加到io线程中，并且会在epoll每一个loop末尾执行。而非merge模式会将handleLoopThread加入到handlePool中（线程池）。并且绑定handleLoopThread或handleLoopCoroutine作为线程入口函数。

- handle初始化

    - handle的初始化根据模式不同也不太一样。如果是merge模式，需要通过setInitializeHandle设置到NetThread中，在线程的run函数中执行。如果是非merge模式，则会在handleLoopThread或handleLoopCoroutine的开始处执行。
    - 通过servantHelperManage找到对应的ServantHelper，然后调用create接口创建helloServant实例。看代码是和adapter一一对应的，可能以后会扩充同一个bindAdapter绑定多个Servant

- Servant初始化

    没有什么特别的会调用helloServant的initialize

## 驱动模型

- Accept

  - 在初始化appilication的服务器部分时，epollServer初始化的部分会在创建io线程和handle线程之后绑定accept回调acceptCallback。
  - accecptCallback会调用，然后获得socket创建connecto，轮询选择NetThread，NetThreads是initHanle中创建的，与bindAdapter绑定的。
  - 讲新创建的connection添加的NetThread中，并且创建epollinfo绑定output、output事件，加入到poller事件循环中中。

- C2S

  - 调用流程：Epoller::done()->Epoller::fireEvent()->Connection::handleInputImp()->TC_TCPTransceiver::doResponse()->recv()->TC_Transceiver::doProtocolAnalysis()->Connection::onParserCallback()->BindAdapter::getProtocol()->BindAdapter::insertRecvQueue()->DataBuffer::insertRecvQueue()->Handle::handleOnceThread()->DataBuffer::pop()->ServantHandle::handle()->ServantHandle::handleTarsProtocol()->Servant::dispatch()->Servant::onDispatch()

  - Epoller::done()

    服务端的io和客户端部分的结构一样。都存在一饿TC_CoroutineScheduler，scheduler中管理epooler。scheduler::run轮训执行poller的done。关于poler的done的具体逻辑可以自行百度epoll的源码。

  - Epoller::fireEvent()

    当epollwait结束，将激活的事件通过fireEvent触发。

  - Connection::handleInputImp()

    - 新的链接建立后，会创建connection绑定socketFd。然后添加到NetThread的ConnectionList后会创建epollInfo，然后注册innput、output等事件并添加到epoller的事件循环中。
    - tars采用的et模式，当读缓存发生变化后会出发input事件，然后出发handleInputImp回调。

  - TC_TCPTransceiver::doResponse()

    - tars为了缩小平台以及tcp udp的差异带来的代码冗余提供了Transceiver类。
    - 这个类提供了重连、收发数据的功能。doResponse调用socket recv接口接收数据。
    - 读取的数据存入_recvBuffer，并且调用doProtocolAnalysis进行协议的解析。

  - Connection::onParserCallback()

    - connection本身不具备协议解析的能力，tars服务端协议解析的能力由BindAdapter提供。BindAdapter的协议解析接口又Appliction初始化以后透传。
    - 调用BindAdapter的协议解析函数的到数据以后会组装RecvContext
    - 将RecvContext添加到DataBuffer中。

  - BindAdapter::insertRecvQueue()

    透传给dataBuffer

  - DataBuffer::insertRecvQueue()

    databuffer提供了wait和notify的能力。如果是merge模式需要通过epollInfo的notify通知poller。

  - Handle::handleOnceThread()

    - 前面在NetThread和handle初始化的部分有提到，handle创建时会讲BindAdapter的dataBuffer绑定到handle上面。当RecvContext添加到DataBuffer中后会通过notify通知handle线程或者通知io线程。
    - handleOnceThread会每次从队列中获取一个RecvContext进行消费。

  - ServantHandle::handle()

    - 根据是否是tars协议决定调用ServantHandle::handleTarsProtocol()或者ServantHandle::handleNoTarsProtocol()
    - 创建current绑定recvContex和sendContext

  - ServantHandle::handleTarsProtocol()

    调用servant的dispath，然后调用ondispath

  - Servant::onDispatch()

    - helloServant的代码是通过tars2Cpp生成的。所有方法基本上都是解析tart和tup协议然后反序列化参数。
    - 打印调用信息
    - 调用hello业务实现的接口
    - 组装sendContext
    - 每个接口都会存入数组中，根据名字可以求出偏移。最后在onDispath中通过序号进行分发。
  
- S2C

  - 调用过程：Current::sendResponse()->Handle::send()->TC_EpollServer::send()->NetThread::send()->Connection::send()->TC_Transceiver::sendRequest()->TC_NetWorkBuffer::addBuffer()->socket::send()

  - Current::sendResponse()

    - 通过current接口绑定recvContex和sendContex
    - 将request中的信息复制给response。例如，RequestID、vertion、调用类型等
    - response结构的序列化tup协议头大小为4字节

  - Handle::send()

    - 透传给EpollServer
    - 传递的是sendContex，这个时候response已经序列化为buff了

  - TC_EpollServer::send()

    - 从response中取出接受消息时绑定的bindAdapter实例。
    - 将response数据放入到接受消息时处理数据的线程中。
    - 会根据当前线程是否是接受消息时的线程有所差异，如果是则直接选取connection执行send。如果不是则需要先添加到发送队列中，然后，激活output事件，从而出发onOuputiml回调。

  - Connection::send

    - 调用TC_Transceiver::sendRequest，
    - 如果待发送数据较多，需要先入发送队列

  - socket::send()

    发送数据啦

