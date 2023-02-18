# clickhouse源码阅读 - server如何启动

`Clickhouse`作为`Olap`的核武器，学习优秀的代码设计，从`clickhosue`服务如何启动开始。

## 程序启动

首先进入`progrms/main.cpp`，`clickhouse_applications`会注册启用的`application`的`main`函数，其中可以看到`mainEntryClickHouseServer`，这就是`clickhouse`服务端的主函数。

```c++
using MainFunc = int (*)(int, char**);

/// Add an item here to register new application
std::pair<const char *, MainFunc> clickhouse_applications[] =
{
#if ENABLE_CLICKHOUSE_LOCAL
    {"local", mainEntryClickHouseLocal},
#endif
#if ENABLE_CLICKHOUSE_CLIENT
    {"client", mainEntryClickHouseClient},
#endif
#if ENABLE_CLICKHOUSE_BENCHMARK
    {"benchmark", mainEntryClickHouseBenchmark},
#endif
#if ENABLE_CLICKHOUSE_SERVER
    {"server", mainEntryClickHouseServer}, 
#endif
#if ENABLE_CLICKHOUSE_EXTRACT_FROM_CONFIG
    {"extract-from-config", mainEntryClickHouseExtractFromConfig},
#endif
#if ENABLE_CLICKHOUSE_COMPRESSOR
    {"compressor", mainEntryClickHouseCompressor},
#endif
#if ENABLE_CLICKHOUSE_FORMAT
    {"format", mainEntryClickHouseFormat},
#endif
#if ENABLE_CLICKHOUSE_COPIER
    {"copier", mainEntryClickHouseClusterCopier},
#endif
#if ENABLE_CLICKHOUSE_OBFUSCATOR
    {"obfuscator", mainEntryClickHouseObfuscator},
#endif
#if ENABLE_CLICKHOUSE_INSTALL
    {"install", mainEntryClickHouseInstall},
    {"start", mainEntryClickHouseStart},
    {"stop", mainEntryClickHouseStop},
    {"status", mainEntryClickHouseStatus},
    {"restart", mainEntryClickHouseRestart},
#endif
};

// 开始执行
int main(int argc_, char ** argv_)
{
    std::vector<char *> argv(argv_, argv_ + argc_);
  // 找到需要启动服务的主函数
    for (auto & application : clickhouse_applications)
    {
        if (isClickhouseApp(application.first, argv))
        {
            main_func = application.second;
            break;
        }
    }
  // 执行mainEntryClickHouseServer
    return main_func(static_cast<int>(argv.size()), argv.data());
}
```

## 启动 Server app

根据`mainEntryClickHouseServer`找到`programs/server/Server.cpp`

```c++
int mainEntryClickHouseServer(int argc, char ** argv)
{
    DB::Server app;
    try
    {
      // 启动app
        return app.run(argc, argv);
    }
    catch (...)
    {
        std::cerr << DB::getCurrentExceptionMessage(true) << "\n";
        auto code = DB::getCurrentExceptionCode();
        return code ? code : 1;
    }
}

int ServerApplication::run(int argc, char** argv)
{
	try
	{
    // 初始化参数
		init(argc, argv);
	}
	catch (Exception& exc)
	{
		logger().log(exc);
		return EXIT_CONFIG;
	}
  // 跟踪run()
	return run();
}

int ServerApplication::run()
{
	return Application::run();
}

int Application::run()
{
	rc = EXIT_SOFTWARE;
  // 核心main方法
	rc = main(_unprocessedArgs);
	return rc;
}

int Application::main(const ArgVec& args)
{
  // 。。。咦怎么就退出了
	return EXIT_OK;
}
```

会发现跟到`Application`的`main`方法后，这条链路就断了，这时候就要回到最开是调用的地方，创建的对象是`DB::Server app`，查看继承关系，`Server->BaseDaemon->ServerApplication->Application`。到这里就很清楚了，`Application`的`main`方法一定会在这个继承链上被覆盖了。所以找到`Server::main`。

这里其实出现了很经典的设计模式---模板方法设计模式，所有的app都会走这样的一条启动链路。最终调用到我们自己实现的启动逻辑。这也是我最喜欢最常用的设计模式之一。

## Server 启动干了些什么

```c++
// 需要做的事情很多，摘抄一些核心的点
int Server::main(const std::vector<std::string> & /*args*/)
{
  // 看方法名就很清晰，在注册一些核心功能
    registerFunctions();
    registerAggregateFunctions();
    registerTableFunctions();
    registerStorages();
    registerDictionaries();
    registerDisks();
  
    /** Context contains all that query execution is dependent:
    *  settings, available functions, data types, aggregate functions, databases, ...
    */
    auto shared_context = Context::createShared();
    auto global_context = std::make_unique<Context>(Context::createGlobal(shared_context.get()));
    global_context_ptr = global_context.get();
  
  // 接来很大的篇幅都是在初始化配置，将配置设置到全局的上下文环境中
    global_context->getProcessList().setMaxSize(config().getInt("max_concurrent_queries", 0));
  
  // 设置最大使用的内存
    size_t max_server_memory_usage = config().getUInt64("max_server_memory_usage", 0);
    double max_server_memory_usage_to_ram_ratio = config().getDouble("max_server_memory_usage_to_ram_ratio", 0.9);
  // memory_amount是物理内存的大小
    size_t default_max_server_memory_usage = memory_amount * max_server_memory_usage_to_ram_ratio;

    if (max_server_memory_usage == 0)
    {
        max_server_memory_usage = default_max_server_memory_usage;
    }
    else if (max_server_memory_usage > default_max_server_memory_usage)
    {
        max_server_memory_usage = default_max_server_memory_usage;
    }
    total_memory_tracker.setOrRaiseHardLimit(max_server_memory_usage);
    total_memory_tracker.setDescription("(total)");
    total_memory_tracker.setMetric(CurrentMetrics::MemoryTracking);
  
  // 连接池
  Poco::ThreadPool server_pool(3, config().getUInt("max_connections", 1024));
  // TCP服务
  std::vector<std::unique_ptr<Poco::Net::TCPServer>> servers; 
  // 启动http、https、Tcp等服务端口
  for (const auto & listen_host : listen_hosts)
        {
            auto create_server = [&](const char * port_name, auto && func)
            {
                /// For testing purposes, user may omit tcp_port or http_port or https_port in configuration file.
                if (!config().has(port_name))
                    return;

                auto port = config().getInt(port_name);
                try
                {
                  // 回调
                    func(port);
                }
                catch (const Poco::Exception &)
                {
                    ...
                }
            };
             /// 核心了解TCP服务的创建过程 
            create_server("tcp_port", [&](UInt16 port)
            {
                Poco::Net::ServerSocket socket;
                auto address = socket_bind_listen(socket, listen_host, port);
                socket.setReceiveTimeout(settings.receive_timeout);
                socket.setSendTimeout(settings.send_timeout);
              // 创建TCPServer对象加入servers集合中
                servers.emplace_back(std::make_unique<Poco::Net::TCPServer>(
                    new TCPHandlerFactory(*this),
                    server_pool,
                    socket,
                    new Poco::Net::TCPServerParams));
            });
    				// 同样的方式创建http、https等服务省略
   					 ...
  			}
	}
	// 依次启动TCP服务
  for (auto & server : servers)
     server->start();
	...
}
```

在创建各种`TCP`服务的时候，这里采用了一个回调技术。先创建一个`create_server`函数对象，调用`create_server`的时候传入获取端口的`key`和具体服务创建逻辑。`create_server`内部调用创建逻辑，传入获取的端口。很经典的回调技术，不只这里用到了回调，后面很多地方都使用到了。如果对设计模式比较了解的话，其实回调就是模板方法模式的一种实现方式。

`server->start()`  看似简单的启动，其实背后蕴含着策略模式的设计意图。`TCPServer`继承`Runnable`接口，`HTTPServer`和`HTTPServer`都继承了`TCPServer`，而start方法是属于`TCPServer`的。但是所有的实现类都必须去`run`方法。即`run`方法实现的就是不同的策略。但是这里还有一个很特别的地方就是他的策略模式是异步的。看下面的代码就会明白的。

```c++
void TCPServer::start()
{
	poco_assert (_stopped);

	_stopped = false;
	_thread.start(*this);
}
```

## TCPServer怎么启动的

找到TCPServer的run方法。

```c++
void TCPServer::run()
{
	while (!_stopped)
	{
		Poco::Timespan timeout(250000);
		try
		{
			if (_socket.poll(timeout, Socket::SELECT_READ))
			{
				try
				{
          // 等待连接
					StreamSocket ss = _socket.acceptConnection();
					
					if (!_pConnectionFilter || _pConnectionFilter->accept(ss))
					{
						// enable nodelay per default: OSX really needs that
#if defined(POCO_OS_FAMILY_UNIX)
						if (ss.address().family() != AddressFamily::UNIX_LOCAL)
#endif
						{
							ss.setNoDelay(true);
						}
            // 将请求放入队列中  TCPServerDispatcher* _pDispatcher
						_pDispatcher->enqueue(ss);
					}
				}
				catch (Poco::Exception& exc)
			...
	}
}

void TCPServerDispatcher::enqueue(const StreamSocket& socket)
{
	FastMutex::ScopedLock lock(_mutex);
	// Poco::NotificationQueue         _queue;
	if (_queue.size() < _pParams->getMaxQueued())
	{
		if (!_queue.hasIdleThreads() && _currentThreads < _pParams->getMaxThreads())
		{
			try
			{// 没有空闲线程，但是未达到最大线程数，创建新的线程
				_threadPool.startWithPriority(_pParams->getThreadPriority(), *this, threadName);
				++_currentThreads;
			}
			catch (Poco::Exception&)
			{
				++_refusedConnections;
				return;
			}
		}
    // 封装成一个Notification对象添加到通知队列中
		_queue.enqueueNotification(new TCPConnectionNotification(socket));
	}
	else
	{
		++_refusedConnections; // 队列满了直接拒绝
	}
} 
    
void NotificationQueue::enqueueNotification(Notification::Ptr pNotification)
{
	poco_check_ptr (pNotification);
	FastMutex::ScopedLock lock(_mutex);
	if (_waitQueue.empty())
	{
    // 入队
		_nfQueue.push_back(pNotification);
	}
	else
	{
		WaitInfo* pWI = _waitQueue.front();
		_waitQueue.pop_front();
		pWI->pNf = pNotification;
		pWI->nfAvailable.set();
	}	
}    
```

到这里一个请求会丢到一个通知队列中，等待调度。

谁来调度? `TCPServerDispatcher* _pDispatcher` ，他是什么时候初始化的

```c++
TCPServer::TCPServer(TCPServerConnectionFactory::Ptr pFactory, Poco::ThreadPool& threadPool, const ServerSocket& socket, TCPServerParams::Ptr pParams):
	_socket(socket),
//	当创建TCP服务的时候会初始化TCPServerDispatcher调度器
	_pDispatcher(new TCPServerDispatcher(pFactory, threadPool, pParams)),
	_thread(threadName(socket)),
	_stopped(true)
{
}

// 调度器也实现了Runable接口，说明会有线程运行run方法
class Net_API TCPServerDispatcher: public Poco::Runnable{}

void TCPServerDispatcher::run()
{
	AutoPtr<TCPServerDispatcher> guard(this, true); // ensure object stays alive

	int idleTime = (int) _pParams->getThreadIdleTime().totalMilliseconds();

	for (;;)	// 死循环，一直在监听消息
	{
		{
			ThreadCountWatcher tcw(this);
			try
			{
        // 等待消息
				AutoPtr<Notification> pNf = _queue.waitDequeueNotification(idleTime);
				if (pNf)
				{
					TCPConnectionNotification* pCNf = dynamic_cast<TCPConnectionNotification*>(pNf.get());
					if (pCNf)
					{
            // 创建连接 _pConnectionFactory是创建TCP服务的时候传进来的，上面也可以找到TCPHandlerFactory
						std::unique_ptr<TCPServerConnection> pConnection(_pConnectionFactory->createConnection(pCNf->socket()));
						poco_check_ptr(pConnection.get());
						beginConnection();
            // 运行的是TCPHandler的start方法，处理具体的请求
						pConnection->start();
						endConnection();
					}
				}
			}
			catch (Poco::Exception &exc) { ErrorHandler::handle(exc); }
			catch (std::exception &exc)  { ErrorHandler::handle(exc); }
			catch (...)                  { ErrorHandler::handle();    }
		}
		if (_stopped || (_currentThreads > 1 && _queue.empty())) break;
	}
}

// TCPHandlerFactory
 Poco::Net::TCPServerConnection * createConnection(const Poco::Net::StreamSocket & socket) override
 {
     try
     {
       // 实际创建了一个TCPHandler
         return new TCPHandler(server, socket);
     }
     catch (const Poco::Net::NetException &)
     {
			...
     }
 }
```

总结一下，`TCPServer`等待连接，将连接请求放入`TCPServerDispatcher`通知队列中，同时`TCPServerDispatcher`会监听这个队列，有连接了则会创建`TCPHandler`来处理。

## TCPHander 开始处理请求

```c++
void TCPServerConnection::start()
{
   try
   {
     // 很显然这里又使用了模板方法模式，由于当前对象是TCPHander，所以去找他的run方法
      run();	
   }
   catch (Exception& exc)
   ...
}

void TCPHandler::run()
{
    try
    {
        runImpl();
    }
    catch (Poco::Exception & e)
    {
      ...
    }
}

void TCPHandler::runImpl()
{
    setThreadName("TCPHandler");
    ThreadStatus thread_status;
  	while (true){
    	/// Processing Query 开始处理sql
    	state.io = executeQuery(state.query, *query_context, false, state.stage, may_have_embedded_data);
    }
}
```

`executeQuery`方式是执行具体的`Sql`，这一偏就不继续往下跟了。下一篇继续。



从`clikchouse-server`启动到接收一个`query`，把具体的流程代码给拿过来，可以发现`clickhouse`的代码设计很优秀的，得多揣摩，方便日后运用到自己的代码上。













