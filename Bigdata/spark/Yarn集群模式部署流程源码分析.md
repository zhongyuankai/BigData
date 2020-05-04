# Yarn集群模式部署流程源码分析

在实际工厂环境下使用的绝大多数的集群管理器是`Hadoop YARN`，因此我们关注的重点是`Hadoop YARN`模式下的`Spark`集群部署。

先熟悉一下`Yarn`集群模式下`spark`部署流程，然后再跟踪源码进行分析。
![](img\yarn.png)

>1. 任务提交后会和ResourceManager通讯申请启动ApplicationMaster，随后ResourceManager分配container；在合适的NodeManager上**启动ApplicationMaster**；
>2. ApplicationMaster启动之后会**启动一个Driver线程**；
>3. Driver启动后，ApplicationMaster**会向ResourceManager注册**，并**申请资源**。
>4. ResourceManager接到ApplicationMaster的资源申请后会**分配container**；
>5. ApplicationMaster在合适的NodeManager上**启动Executor**；
>6. Executor启动后会**向Driver反向注册**；
>7. Executor全部注册完成后**Driver开始执行main函数**，之后执行到Action算子时，触发一个job，并根据宽依赖开始划分stage，**每个stage生成对应的taskSet**，之后**将task分发到各个Executor上执行**；



熟悉了`spark`的部署流程后，源码撸起来。
万事开头难，从哪开始呢？没错从我们提交`spark`应用程序开始。

>bin/spark-submit \
>--class org.apache.spark.examples.SparkPi \
>--num-executors 2 \
>--master yarn \
>--deploy-mode cluster \
>./examples/jars/spark-examples_2.11-2.1.1.jar \
>100

通过提交这个`spark-submit`一提交我们的程序就启动起来了，所以从`spark-submit`开刀。

```java
vim spark-submit
```


查看脚本，我们可以发现其实里面其实就是通过`java`命令运行了一个`java`程序；

```java
// java命令运行这个类
spark-submit-> /bin/java org.apache.spark.deploy.SparkSubmit
```

很显然运行了**org.apache.spark.deploy.SparkSubmit**这个类，那么我们只需要去找类的`main`方法，接着就开始了源码之旅。

整个源码我就不沾过来了，只看核心部分，写的是伪代码。

```java
org.apache.spark.deploy.SparkSubmit
main{
    // main方法里面掉了submit方法
	submit(){
		// 解析脚本参数
		prepareSubmitEnvironment(args){
        
			isYarnCluster -> 
            // yarn集群模式的话运行的是这个类
				childMainClass = "org.apache.spark.deploy.yarn.Client"
                
			Client ->   // 即验证了client模式，Driver直接在客户端上运行
				childMainClass = args.mainClass	// 即用户程序的主类
			
		} 
		
		doRunMain(){
			runMain(){
				//var mainClass: Class[_]; 通过反射拿到childMainClass的Class对象
				mainClass = Utils.classForName(childMainClass)
		
				// 获取Class对像的main方法
				val mainMethod = mainClass.getMethod("main", new Array[String](0).getClass)
		
				// 调用运行main
				mainMethod.invoke(null, childArgs.toArray)
			}
		}	
	}
}
```

看到这里我们可以知道，`submint`方法里面运行了**org.apache.spark.deploy.yarn.Client**这个类的`main`方法。所以我们接下来就去找这个类的**main方法**。

如果没找到的话，在`pom`文件中加入这个依赖。
><dependency>
><groupId>org.apache.spark</groupId>
><artifactId>spark-yarn_2.11</artifactId>
><version>2.1.1</version>
></dependency>

```java
org.apache.spark.deploy.yarn.Client
main(){
	
	new Client(args, sparkConf).run()
	
	// 提交一个application到ResourceManager
	run(){
        // 提交application返回appid
		this.appId = submitApplication()
		{
			//LauncherBackend 是跟LauncherServer通信的客户端，向LauncherServer发送状态变化的通信端点
			launcherBackend.connect()	// 连接RM
			
			// Get a new application from our RM
			val newApp = yarnClient.createApplication()	// 让RM创建一个application
			val newAppResponse = newApp.getNewApplicationResponse() // 获取新application的响应
			appId = newAppResponse.getApplicationId()	// 获取appid
			
			// 创建一个amContainer容器
			val containerContext = createContainerLaunchContext(newAppResponse)
			{
				// 设置java虚拟机的运行参数
				
				// 获取am运行的主类(Cluster模式)
				val amClass = Utils.classForName("org.apache.spark.deploy.yarn.ApplicationMaster").getName
				
				// 封装指令 command = bin/java amClass ...
			}
			// 主要进行sparkConf的配置
			val appContext = createApplicationSubmissionContext(newApp, containerContext)
			
			// 向Yarn提交应用，实际上提交的是指令
			yarnClient.submitApplication(appContext)
		}
	}
}
```

提交`application`实际上就是提交封装的一个`command`，里面就是一个启动`java`进程的一个命令，启动的类是
**org.apache.spark.deploy.yarn.ApplicationMaster**，所以我们就去找这个类的**main方法**。

去找这个`mian`方法之前我们需要明白，提交`application`之后，`ResourceManager`会将`application`打包成一个任务放入任务队列中，`NodeManger`就会来领取任务，运行`ApplicationMaster`，即下面这个类。

```java
org.apache.spark.deploy.yarn.ApplicationMaster
main(){
	// 封装用户提交的参数，即 java --jar ... --class ...等参数
	val amArgs = new ApplicationMasterArguments(args)
	// 创建ApplicationMaster，同时传入用户参数和与RM交互的客户端
	master = new ApplicationMaster(amArgs, new YarnRMClient)
	// 启动创建ApplicationMaster
	master.run(){
	// isClusterMode 运行Driver
	runDriver(securityMgr)
	{
		// 启动用户的应用
		userClassThread = startUserApplication()
		{
		// 获取用户的参数
		var userArgs = args.userArgs
		// 获取用户程序的主方法	，args.userClass即拿到了用户主类路径
		val mainMethod = userClassLoader.loadClass(args.userClass).getMethod("main", classOf[Array[String]])
		
		// 创建一个Driver线程执行用户程序的main方法
		val userThread = new Thread {
			run(){
				mainMethod.invoke(null, userArgs.toArray)
			}
		}
		userThread.setName("Driver")
		userThread.start()
		}
		
		// 向RM注册AM
		registerAM(sc.getConf, rpcEnv, driverRef, sc.ui.map(_.appUIAddress).getOrElse("")
		{
		// client是YarnRMClient， 向RM注册自己并获取资源
		allocator = client.register(driverUrl,driverRef,yarnConf,_sparkConf,uiAddress,historyAddress,securityMgr,localResources)
		
		// 分配的资源
		allocator.allocateResources()
		{
			// 处理可分配的资源
			handleAllocatedContainers(allocatedContainers.asScala)
			{	
			// 这里会涉及到本地化策略，如进程本地化、节点本地化、机架本地化...
			
			// 运行可分配的Container
			runAllocatedContainers(containersToUse)
			{
				// 遍历containers
				for (container <- containersToUse) 
				{
				// 正在运行的Executor的数量还要小于总共需要的Executors的数量，则继续创建运行
				if (numExecutorsRunning < targetNumExecutors) {
				
					// launcherPool线程池来执行
					launcherPool.execute(new Runnable {
					run(){
					
						new ExecutorRunnable(Some(container),conf,sparkConf,driverUrl,executorId...).run()
						{
						// 创建与NodeManager交互的客户端并启动
						nmClient = NMClient.createNMClient()
						nmClient.init(conf)
						nmClient.start()
						
						// 启动容器
						startContainer()
						{
							
						val ctx = Records.newRecord(classOf[ContainerLaunchContext]).asInstanceOf[ContainerLaunchContext]
						
						// 封装命令，commands = /bin/java org.apache.spark.executor.CoarseGrainedExecutorBackend ...
						val commands = prepareCommand()
						ctx.setCommands(commands.asJava)
						
						// 让客户端去对应的NM上启动容器
						nmClient.startContainer(container.get, ctx)
						}
						}
					}
					}
				}
				}
			}
			}
			
		}
		}
		// Driver线程没执行完，当前线程不能继续往下执行，也就是说ApplicationMaster不能结束
		userClassThread.join()
	}
	}
}

```

`ApplicationMaster`会通过`NodeManager`的客户端和`NodeManager`通信启动容器，同时将`commands`发送过去，在`commands`里面要运行的类是**org.apache.spark.executor.CoarseGrainedExecutorBackend**，所以接下来就要去找这个类的`main`方法。

```java
org.apache.spark.executor.CoarseGrainedExecutorBackend
main(){
	
	run(driverUrl, executorId, hostname, cores, appId, workerUrl, userClassPath)
	{
		// 创建一个Executor后台程序
		env.rpcEnv.setupEndpoint("Executor", new CoarseGrainedExecutorBackend(env.rpcEnv, driverUrl, executorId, hostname, cores, userClassPath, env))
	}
}
```

这个`mian`方法很简单就是创建了`CoarseGrainedExecutorBackend`对象，很显然这个对象就是`Executor`后台程序，所以接下来就来看看这个对象里面在干嘛。

```java
private[spark] class CoarseGrainedExecutorBackend() extends ThreadSafeRpcEndpoint{
	// 由于该类继承了Rpc端点，所以该对象的生命周期是 constructor(创建) -> onStart(启动) -> receive*(接收消息) -> onStop(停止)

	// 我们所说的Executor就是CoarseGrainedExecutorBackend中的一个属性对象
	var executor: Executor = null
	
	override def onStart() {
		//向Driver反向注册
		driver = Some(ref)
		ref.ask[Boolean](RegisterExecutor(executorId, self, hostname, cores, extractLogUrls))
	}
	
	override def receive: PartialFunction[Any, Unit] = {
		// 收到Driver注册成功的消息
		case RegisteredExecutor =>
			// 创建计算对象Executor
			executor = new Executor(executorId, hostname, env, userClassPath, isLocal = false)
		
		// 收到Driver端发送过来的task
		case LaunchTask(data) =>
			// 由executor对象调用方法运行
			executor.launchTask(this, taskId = taskDesc.taskId, attemptNumber = taskDesc.attemptNumber,taskDesc.name, taskDesc.serializedTask)
	}
}
```

通过对`Yarn`集群模式源码解读，应该对这个`spark`的部署流程应该更加熟悉，中间涉及到哪些类，是怎样相互调用的。
