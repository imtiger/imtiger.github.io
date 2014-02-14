---
layout: post
title: "Tomcat启动过程（Tomcat源代码阅读系列之三）"
date: 2013-10-17 11:03
comments: true
keywords: tomcat源代码，java
categories: 
- Tomcat
- Java
- 技术
---
{% img center /images/2013/10/14/apache_tomcat_bag.jpg%}
本文是[Tomcat源代码阅读系列](/blog/2013/10/08/tomcat-source-code-study/)的第三篇文章，在阅读此文之前，建议先读前面的两篇：  
[在IntelliJ IDEA 和 Eclipse运行tomcat 7源代码（Tomcat源代码阅读系列之一）](/blog/2013/10/14/run-tomcat-in-idea-or-eclipse/)   
[Tomcat总体结构                             （Tomcat源代码阅读系列之二）](/blog/2013/10/16/tomcat-architecture/)

本篇我们来一起分析一下Tomcat的启动过程，启动过程涉及到了Tomcat组件的生命周期管理，本文将从**Tomcat组件生命周期管理**,**Tomcat启动的总过程**，**Tomcat启动过程关键步骤分析**三个方面来进行描述。
<!-- more -->
#Tomcat组件生命周期管理
在[Tomcat总体结构                             （Tomcat源代码阅读系列之二）](/blog/2013/10/16/tomcat-architecture/)中，我们列出了Tomcat中Server,Service,Connector,Engine,Host,Context的继承关系图，你会发现它们都实现了`org.apache.catalina.Lifecycle`接口，而`org.apache.catalina.util.LifecycleBase`采用了`模板方法模式`来对所有支持生命周期管理的组件的生命周期各个阶段进行了总体管理，每个需要生命周期管理的组件只需要继承这个基类，然后覆盖对应的钩子方法即可完成相应的生命周期阶段性的管理工作。
下面我们首先来看看`org.apache.catalina.Lifecycle`接口的定义，它的类图如下图所示：
{% img center /images/2013/10/17/LifeCycle.png %}
从上图我们可以清楚的看到LifeCycle中主要有四个生命周期阶段，它们分别是init(初始化)，start(启动),stop(停止)，destory(销毁)。知道了这四个生命周期阶段以后，咋们就来看看`org.apache.catalina.util.LifecycleBase`是如何实现`模板方法模式`的。
那接下来我们就来看看`org.apache.catalina.util.LifecycleBase`类的定义，它的类图如下所示：
{% img center /images/2013/10/17/LifeCycleBase.png%}
上图中用红色标注的四个方法就是`模板方法模式`中的钩子方法，子类可以通过实现钩子方法来纳入到基类已经流程化好的生命周期管理中。  
上面我们对LifeCycle和LifeCycleBase有了一个总体的认识，接下来，我们通过查看`org.apache.catalina.util.LifecycleBase`的源代码来具体的分析一下。
咋们首先来看`org.apache.catalina.util.LifecycleBase`的init方法的实现。
```java org.apache.catalina.util.LifecycleBase#init
@Override
public final synchronized void init() throws LifecycleException {
        // 1
        if (!state.equals(LifecycleState.NEW)) {
            invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
        }
        setStateInternal(LifecycleState.INITIALIZING, null, false);

        try {
            // 2 
            initInternal();
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            setStateInternal(LifecycleState.FAILED, null, false);
            throw new LifecycleException(
                    sm.getString("lifecycleBase.initFail",toString()), t);
        }
        // 3 
        setStateInternal(LifecycleState.INITIALIZED, null, false);
}
```
下面我们逐一来分析一下上述代码中标注了数字的地方：  

1. 标注1的代码首先检测当前组件的状态是不是`NEW`(新建)，如果不是就调用`org.apache.catalina.util.LifecycleBase#invalidTransition`方法来将当前的状态转换过程终止，而`invalidTransition`的实现是抛出了`org.apache.catalina.LifecycleException`异常。接着调用了`setStateInternal`方法将状态设置为INITIALIZING（正在初始化）
2. 标注2的代码就是init模板方法的钩子，子类可以通过实现`protected abstract void initInternal() throws LifecycleException;`方法来纳入初始化的流程。
3. 标注3的代码将组件的状态改为`INITIALIZED`(已初始化)。

上面我们分析了init模板方法，接下来我们再看看start方法具体做了什么事情。start的代码如下：
```java org.apache.catalina.util.LifecycleBase#start
    public final synchronized void start() throws LifecycleException {
        // 1
        if (LifecycleState.STARTING_PREP.equals(state) ||
                LifecycleState.STARTING.equals(state) ||
                LifecycleState.STARTED.equals(state)) {
            
            if (log.isDebugEnabled()) {
                Exception e = new LifecycleException();
                log.debug(sm.getString("lifecycleBase.alreadyStarted",
                        toString()), e);
            } else if (log.isInfoEnabled()) {
                log.info(sm.getString("lifecycleBase.alreadyStarted",
                        toString()));
            }
            
            return;
        }
        
        // 2
        if (state.equals(LifecycleState.NEW)) {
            init();
        } else if (state.equals(LifecycleState.FAILED)){
            stop();
        } else if (!state.equals(LifecycleState.INITIALIZED) &&
                !state.equals(LifecycleState.STOPPED)) {
            invalidTransition(Lifecycle.BEFORE_START_EVENT);
        }
        // 3
        setStateInternal(LifecycleState.STARTING_PREP, null, false);

        try {
        //4   
            startInternal();
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            setStateInternal(LifecycleState.FAILED, null, false);
            throw new LifecycleException(
                    sm.getString("lifecycleBase.startFail",toString()), t);
        }
        
        // 5
        if (state.equals(LifecycleState.FAILED) ||
                state.equals(LifecycleState.MUST_STOP)) {
            stop();
        } else {
            // Shouldn't be necessary but acts as a check that sub-classes are
            // doing what they are supposed to.
            if (!state.equals(LifecycleState.STARTING)) {
                invalidTransition(Lifecycle.AFTER_START_EVENT);
            }
            
            setStateInternal(LifecycleState.STARTED, null, false);
        }
    }

```
下面我们逐一来分析一下上述代码中标注了数字的地方：  

1. 标注1的代码检测当前组件的状态是不是`STARTING_PREP`(准备启动),`STARTING`（正在启动）,`STARTED`（已启动）.如果是这三个状态中的任何一个，则抛出`LifecycleException`。
2. 标注2的代码的检查其实主要是为了保证组件状态的完整性，在正常启动的流程中，应该是不会出现没有初始化就启动，或者还没启动就已经失败的情况。
3. 标注3的代码设置组件的状态为`STARTING_PREP`（准备启动状态）
4. 标注4的代码是start模板方法的钩子方法，子类通过实现`org.apache.catalina.util.LifecycleBase#startInternal`这个方法来纳入到组件启动的流程中来。
5. 标注5的代码做了一些状态检查，然后最终将组件的状态设置为`STARTED`(已启动)

上面我们分析了init和start方法的流程，对于stop和destroy方法的总体过程是类似的，大家可以自己阅读一下，但是通过上面的分析，我们可以得出生命周期方法的总体的骨架，如果用伪代码来表示可以简化为如下：
```java org.apache.catalina.util.LifecycleBase#lifeCycleMethod

    public final synchronized void lfieCycleMethod() throws LifecycleException {
        stateCheck();//状态检查
        //设置为进入相应的生命周期之前的状态
        setStateInternal(LifecycleState.BEFORE_STATE, null, false);
        lfieCycleMethodInternal();//钩子方法
        //进入相应的生命周期之后的状态
        setStateInternal(LifecycleState.AFTER_STATE, null, false);
    }
```		
#Tomcat启动的总过程

通过上面的介绍，我们总体上清楚了各个组件的生命周期的各个阶段具体都是如何运作的。接下来我们就来看看，Tomcat具体是如何一步步启动起来的。我们都知道任何Java程序都有一个main函数入口，Tomcat中的main入口是`org.apache.catalina.startup.Bootstrap#main`,下面我们就来分析一下它的代码：
```java org.apache.catalina.startup.Bootstrap#main
public static void main(String args[]) {

    if (daemon == null) {
        // Don't set daemon until init() has completed
        // 1 
        Bootstrap bootstrap = new Bootstrap();
        try {
            // 2
            bootstrap.init();
        } catch (Throwable t) {
            handleThrowable(t);
            t.printStackTrace();
            return;
        }
        // 3
        daemon = bootstrap;
    } else {
        // When running as a service the call to stop will be on a new
        // thread so make sure the correct class loader is used to prevent
        // a range of class not found exceptions.
        Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
    }

    try {
        String command = "start";
        if (args.length > 0) {
            command = args[args.length - 1];
        }

        if (command.equals("startd")) {
            args[args.length - 1] = "start";
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stopd")) {
            args[args.length - 1] = "stop";
            daemon.stop();
        } else if (command.equals("start")) {
            // 4
            daemon.setAwait(true);
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stop")) {
            daemon.stopServer(args);
        } else if (command.equals("configtest")) {
            daemon.load(args);
            if (null==daemon.getServer()) {
                System.exit(1);
            }
            System.exit(0);
        } else {
            log.warn("Bootstrap: command \"" + command + "\" does not exist.");
        }
    } catch (Throwable t) {
        // Unwrap the Exception for clearer error reporting
        if (t instanceof InvocationTargetException &&
                t.getCause() != null) {
            t = t.getCause();
        }
        handleThrowable(t);
        t.printStackTrace();
        System.exit(1);
    }

}

```

下面我们逐一来分析一下上述代码中标注了数字的地方： 
 
1. 标注1的代码初始化了自举类的实例，标注2的代码对BootStrap实例进行了初始化，标注3的代码将实例赋值给了daemon。
2. 标注4的代码首先调用了BootStrap的load方法，然后调用了start方法。

接下来我们分别分析一下BootStrap的init,load，start方法具体做了哪些工作。
##BootStrap#init方法
首先来看`org.apache.catalina.startup.Bootstrap#init`方法，它的代码如下：
```java org.apache.catalina.startup.Bootstrap#init
public void init()throws Exception{

    // Set Catalina path
    setCatalinaHome();
    setCatalinaBase();

    initClassLoaders();

    Thread.currentThread().setContextClassLoader(catalinaLoader);

    SecurityClassLoad.securityClassLoad(catalinaLoader);

    // Load our startup class and call its process() method
    if (log.isDebugEnabled())
        log.debug("Loading startup class");
    // 1
    Class<?> startupClass =
        catalinaLoader.loadClass
        ("org.apache.catalina.startup.Catalina");
    Object startupInstance = startupClass.newInstance();

    // Set the shared extensions class loader
    if (log.isDebugEnabled())
        log.debug("Setting startup class properties");
    String methodName = "setParentClassLoader";
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Class.forName("java.lang.ClassLoader");
    Object paramValues[] = new Object[1];
    paramValues[0] = sharedLoader;
    Method method =
        startupInstance.getClass().getMethod(methodName, paramTypes);
    // 2
    method.invoke(startupInstance, paramValues);
    // 3
    catalinaDaemon = startupInstance;

}

```	
下面我们重点逐一来分析一下上述代码中标注了数字的地方：

1. 标注1的代码通过反射实例化了`org.apache.catalina.startup.Catalina`类的实例;
2. 标注2的代码调用了Catalina实例的setParentClassLoader方法设置了父亲ClassLoader，对于ClassLoader方面的内容，我们在本系列的后续文章再来看看。标注3的代码将Catalina实例赋值给了Bootstrap实例的catalinaDaemon.

##BootStrap#load
接下来我们再来看看`org.apache.catalina.startup.Bootstrap#load`方法，通过查看源代码，我们知道此方法通过反射调用了`org.apache.catalina.startup.Catalina#load`方法，那我们就来看看Catalina的load方法，Catalina#load方法代码如下：
```java org.apache.catalina.startup.Catalina#load
public void load() {

    // 1 
    Digester digester = createStartDigester();

    InputSource inputSource = null;
    InputStream inputStream = null;
    File file = null;
    try {
        file = configFile();
        inputStream = new FileInputStream(file);
        inputSource = new InputSource(file.toURI().toURL().toString());
    } catch (Exception e) {
        if (log.isDebugEnabled()) {
            log.debug(sm.getString("catalina.configFail", file), e);
        }
    }
   

    
    try {
        inputSource.setByteStream(inputStream);
        digester.push(this);
        digester.parse(inputSource);
    } catch (SAXParseException spe) {
        log.warn("Catalina.start using " + getConfigFile() + ": " +
                spe.getMessage());
        return;
    } catch (Exception e) {
        log.warn("Catalina.start using " + getConfigFile() + ": " , e);
        return;
    } finally {
        try {
            inputStream.close();
        } catch (IOException e) {
            // Ignore
        }
    }
    
    
    getServer().setCatalina(this);

    // Stream redirection
    initStreams();

    // Start the new server
    try {
        // 2
        getServer().init();
    } catch (LifecycleException e) {
        if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
            throw new java.lang.Error(e);
        } else {
            log.error("Catalina.start", e);
        }

    }
    
}
```
上面的代码，我只保留了主流程核心的代码，下面我们重点逐一来分析一下上述代码中标注了数字的地方：

1. 标注1的代码创建Digester实例解析"conf/server.xml"文件
2. 标注2的代码最终调用了StandardServer的init方法。

大家可以自行查看下源代码，我们会发现如下的一个调用流程：
```java init call stack  
org.apache.catalina.core.StandardServer#init   
->org.apache.catalina.core.StandardService#init  
-->org.apache.catalina.connector.Connector#init  
-->org.apache.catalina.core.StandardEngine#init  
```


>因为StandardService，Connector，StandardEngine实现了LifeCycle接口，因此符合我们上文所获的生命周期的管理，最终都是通过他们自己实现的initInternal方法进行初始化

读到这里的时候，我想大家应该和我一样，以为StandardEngine#init方法会调用StandardHost#init方法，但是当我们查看StandardEngine#init方法的时候，发现并没有进行StandardHost的初始化，它到底做了什么呢？让我们来具体分析一下,我们首先拿StanderEngine的继承关系图来看下：
{% img center /images/2013/10/16/StandardEngine.png %}
通过上图以及前面说的LifeCyecle的模板方法模式，我们知道StandardEngine的初始化钩子方法initInternal方法最终调用了ContainerBase的initInternal方法，那我们拿ContainerBase#initInternal方法的代码看看：  
```java org.apache.catalina.core.ContainerBase#initInternal
protected void initInternal() throws LifecycleException {
    BlockingQueue<Runnable> startStopQueue =
        new LinkedBlockingQueue<Runnable>();
    startStopExecutor = new ThreadPoolExecutor(
            getStartStopThreadsInternal(),
            getStartStopThreadsInternal(), 10, TimeUnit.SECONDS,
            startStopQueue,
            new StartStopThreadFactory(getName() + "-startStop-"));
    startStopExecutor.allowCoreThreadTimeOut(true);
    super.initInternal();
}
```
我们可以看到StandardEngine的初始化仅仅是创建了一个ThreadPoolExecutor，当看到这里的时候，笔者当时也纳闷了，StandardEngine#init竟然没有调用StandardHost#init方法，那么StandardHost的init方法是什么时候被调用的呢？遇到这种不知道到底方法怎么调用的时候怎么办呢？笔者介绍个方法给大家。我们现在需要知道StandardHost#init方法何时被调用的，而我们知道init最终会调用钩子的initInternal方法，因此这个时候，我们可以在StandardHost中override initInternal方法，增加了实现方法以后，有两种方法可以用，一种就是设置个断点debug一下就可以看出线程调用栈了，另外一种就是在新增的方法中打印出调用栈。笔者这里采用第二种方法，我们增加如下的initInternal方法到StandardHost中：
```java org.apache.catalina.core.StandardHost#initInternal
protected void initInternal() throws LifecycleException {
    Throwable ex = new Throwable();
    StackTraceElement[] stackElements = ex.getStackTrace();
    if (stackElements != null) {
        for (int i = stackElements.length - 1; i >= 0; i--) {
            System.out.print(stackElements[i].getClassName() + "\t");
            System.out.print(stackElements[i].getMethodName() + "\t");
            System.out.print(stackElements[i].getFileName() + "\t");
            System.out.println(stackElements[i].getLineNumber());
        }
    }
    super.initInternal();   
}
```
上面的代码将会打印出方法调用堆栈，对于调试非常有用，上面的方法运行以后在控制台打印出了如下的堆栈信息：
```java stack info
java.lang.Thread	run	Thread.java	680
java.util.concurrent.ThreadPoolExecutor$Worker	run	ThreadPoolExecutor.java	918
java.util.concurrent.ThreadPoolExecutor$Worker	runTask	ThreadPoolExecutor.java	895
java.util.concurrent.FutureTask	run	FutureTask.java	138
java.util.concurrent.FutureTask$Sync	innerRun	FutureTask.java	303
org.apache.catalina.core.ContainerBase$StartChild	call	ContainerBase.java	1549
org.apache.catalina.core.ContainerBase$StartChild	call	ContainerBase.java	1559
org.apache.catalina.util.LifecycleBase	start	LifecycleBase.java	139
org.apache.catalina.util.LifecycleBase	init	LifecycleBase.java	102
org.apache.catalina.core.StandardHost	initInternal	StandardHost.java	794
```
通过控制台的信息，我们看到是StartChild#call方法调用的，而我们查看StartChild#call方法其实是在StandardEngine的startInternal方法中通过异步线程池去初始化子容器。因此到这里我们就理清楚了，StarndardHost的init方法是在调用start方法的时候被初始化。那么接下来我们就来看看，start方法的整体调用流程。

##BootStrap#start
采用分析load方法一样的方法，经过对BootStrap#start的分析，我们最终可以得到得到如下的调用链：
```java org.apache.catalina.startup.Bootstrap#start call stack
org.apache.catalina.startup.Bootstrap#start
->org.apache.catalina.startup.Catalina#start 通过反射调用
-->org.apache.catalina.core.StandardServer#start
--->org.apache.catalina.core.StandardService#start
---->org.apache.catalina.core.StandardEngine#start
---->org.apache.catalina.Executor#start
---->org.apache.catalina.connector.Connector#start
```

综合上文的描述我们总体得到如下的调用链：
```java org.apache.catalina.startup.Bootstrap#main call stack 
org.apache.catalina.startup.Bootstrap#main
->org.apache.catalina.startup.Bootstrap#init
->org.apache.catalina.startup.Bootstrap#load
-->org.apache.catalina.startup.Catalina#load
--->org.apache.catalina.core.StandardServer#init
---->org.apache.catalina.core.StandardService#init
----->org.apache.catalina.connector.Connector#init
----->org.apache.catalina.core.StandardEngine#init
->org.apache.catalina.startup.Bootstrap#start
-->org.apache.catalina.startup.Catalina#start 通过反射调用
--->org.apache.catalina.core.StandardServer#start
---->org.apache.catalina.core.StandardService#start
----->org.apache.catalina.core.StandardEngine#start
----->org.apache.catalina.Executor#start
----->org.apache.catalina.connector.Connector#start

```
通过上面的分析我们已经搞清楚了Tomcat启动的总体的过程，但是有一些关键的步骤，我们还需要进行进一步的深入探究。let's do it.
#Tomcat启动过程关键步骤分析
##Connector#init
我们首先来看一下**org.apache.catalina.connector.Connector#init**,我们知道Connector的生命周期也是通过LifeCycle的模板方法模式来管理的，那么我们只需要查看一下它的initInternal方法即可知道它是如何初始化的。接下来我们就来看一下initInternal方法，代码如下：

```java org.apache.catalina.connector.Connector#initInternal
protected void initInternal() throws LifecycleException {

        super.initInternal();

        // Initialize adapter
        adapter = new CoyoteAdapter(this);
        protocolHandler.setAdapter(adapter);

        // Make sure parseBodyMethodsSet has a default
        if( null == parseBodyMethodsSet ) {
            setParseBodyMethods(getParseBodyMethods());
        }

        if (protocolHandler.isAprRequired() &&
                !AprLifecycleListener.isAprAvailable()) {
            throw new LifecycleException(
                    sm.getString("coyoteConnector.protocolHandlerNoApr",
                            getProtocolHandlerClassName()));
        }

        try {
            // 1 
            protocolHandler.init();
        } catch (Exception e) {
            throw new LifecycleException
                (sm.getString
                 ("coyoteConnector.protocolHandlerInitializationFailed"), e);
        }

        // Initialize mapper listener
        mapperListener.init();
    }

```

上面代码中，本文最关心的是标注了1的地方，这个地方调用了`org.apache.coyote.ProtocolHandler#init`方法，而ProtocolHandler是在Connector的构造函数中初始化，而Connector的构造函数又是Digester类解析conf/server.xml的时候调用的，明白了这点，我们在来具体看看Connector构造函数中调用的一个核心的方法setProtocol方法，下面是其代码：

```java org.apache.catalina.connector.Connector#setProtocol
public void setProtocol(String protocol) {

        if (AprLifecycleListener.isAprAvailable()) {
            //这里统一使用AprEndpoint
            if ("HTTP/1.1".equals(protocol)) {
                setProtocolHandlerClassName
                    ("org.apache.coyote.http11.Http11AprProtocol");   //Http11AprProtocol$Http11ConnectionHandler
            } else if ("AJP/1.3".equals(protocol)) {
                setProtocolHandlerClassName
                    ("org.apache.coyote.ajp.AjpAprProtocol");     //AjpAprProtocol$AjpConnectionHandler
            } else if (protocol != null) {
                setProtocolHandlerClassName(protocol);
            } else {
                setProtocolHandlerClassName
                    ("org.apache.coyote.http11.Http11AprProtocol");
            }
        } else {
            // 1 
            if ("HTTP/1.1".equals(protocol)) {
                setProtocolHandlerClassName
                    ("org.apache.coyote.http11.Http11Protocol");  //Http11Protocol$Http11ConnectionHandler
            } else if ("AJP/1.3".equals(protocol)) {
                setProtocolHandlerClassName
                    ("org.apache.coyote.ajp.AjpProtocol");    //AjpProtocol$AjpConnectionHandler
            } else if (protocol != null) {
                setProtocolHandlerClassName(protocol);
            }
        }
}

```

从setProtocol的代码中，我们可以看出主要逻辑分为了两块，一种情况是使用[APR(Apache Portable Runtime)](http://tomcat.apache.org/tomcat-7.0-doc/apr.html)，另外一种是不使用APR的情况。缺省情况下不采用APR库，这样的话，代码会走到标注1的代码分支，这里通过协议的不同，最终初始化了不同的类。如果是http1.1协议就采用`org.apache.coyote.http11.Http11Protocol`,如果是AJP(Apache Jserv Protocol)协议，就采用`org.apache.coyote.ajp.AjpProtocol`类，下面我们来看一下Http11Protocol和AjpProtocol的继承关系图如下：
{% img center /images/2013/10/17/Http11Protocol.png %}  
{% img center /images/2013/10/17/AJPProtocol.png %}
通过上图我们可以看到它们都继承了公共的基类`org.apache.coyote.AbstractProtocol`,而它们自己的init方法最终其实都是调用了AbstractProtocol的init方法，通过查看AbstractProtocol#init代码，我们可以看到最终是调用了`org.apache.tomcat.util.net.AbstractEndpoint#init`,而AbstractEndpoint的实例化操作是在实例化AjpProtocol和Http11Protocol的时候在其构造函数中实例化的，而AjpProtocol和Http11Protocol构造函数中，其实都是初始化了`org.apache.tomcat.util.net.JIoEndpoint`类，只不过根据是http协议还是AJP协议，它们具有不同的连接处理类。其中Http11Protocol的连接处理类为`org.apache.coyote.http11.Http11Protocol.Http11ConnectionHandler`,而连接处理类为`org.apache.coyote.ajp.AjpProtocol.AjpConnectionHandler`，因此到这里我们基本清楚了Connector的初始化流程，总结如下：
```java Connect init 采用APR的情况
//1 HTTP/1.1协议连接器
org.apache.catalina.connector.Connector#init
->org.apache.coyote.http11.Http11AprProtocol#init
-->org.apache.tomcat.util.net.AprEndpoint#init
(org.apache.coyote.http11.Http11AprProtocol.Http11ConnectionHandler)


// 2 AJP/1.3协议连接器
org.apache.catalina.connector.Connector#init
->org.apache.coyote.ajp.AjpAprProtocol#init
-->org.apache.tomcat.util.net.AprEndpoint#init
(org.apache.coyote.ajp.AjpAprProtocol.AjpConnectionHandler)

```
```java Connector init 不采用APR的情况
// 1 HTTP/1.1协议连接器
org.apache.catalina.connector.Connector#init
->org.apache.coyote.http11.Http11Protocol#init
-->org.apache.tomcat.util.net.JIoEndpoint#init
(org.apache.coyote.http11.Http11Protocol.Http11ConnectionHandler)

// 2 AJP/1.3协议连接器
org.apache.catalina.connector.Connector#init
->org.apache.coyote.ajp.AjpProtocol#init
-->org.apache.tomcat.util.net.JIoEndpoint#init
(org.apache.coyote.ajp.AjpProtocol.AjpConnectionHandler)

```
>这里需要注意，除了JIoEndpoint外，还有NIoEndpoint，对于Tomcat7.0.24的代码，并没有采用NIOEndPoint，NIOEndpoint采用了NIO的方式进行Socket的处理。

最后，咋们再来看看org.apache.tomcat.util.net.JIoEndpoint#init的初始化过程，我们首先来看一下JIoEndpoint的继承关系图如下：
{% img center /images/2013/10/17/JIoEndpoint.png %}
通过上图我们知道JIoEndpoint继承了AbstractEndpoint，而通过查看源码可知，JIoEndpoint没有实现自己的init方法，它默认采用了父类的init方法，那么我们就来看看AbstractEndpoint的init，它的代码如下：
```java org.apache.tomcat.util.net.AbstractEndpoint#init
 public final void init() throws Exception {
        if (bindOnInit) {
            bind();
            bindState = BindState.BOUND_ON_INIT;
        }
}
```
通过查看上面的代码可知，因为bindOnInit默认是true,所以init调用了bind方法，而bind方法是抽象方法，最终由JIoEndpoint来实现，代码如下：
```java org.apache.tomcat.util.net.JIoEndpoint#bind
@Override
public void bind() throws Exception {

        // Initialize thread count defaults for acceptor
        if (acceptorThreadCount == 0) {
            acceptorThreadCount = 1;
        }
        // Initialize maxConnections
        if (getMaxConnections() == 0) {
            // User hasn't set a value - use the default
            setMaxConnections(getMaxThreadsExecutor(true));
        }

        if (serverSocketFactory == null) {
            if (isSSLEnabled()) {
                serverSocketFactory =
                    handler.getSslImplementation().getServerSocketFactory(this);
            } else {
                serverSocketFactory = new DefaultServerSocketFactory(this);
            }
        }

        if (serverSocket == null) {
            try {
                if (getAddress() == null) {
                    serverSocket = serverSocketFactory.createSocket(getPort(),
                            getBacklog());
                } else {
                    serverSocket = serverSocketFactory.createSocket(getPort(),
                            getBacklog(), getAddress());
                }
            } catch (BindException orig) {
                String msg;
                if (getAddress() == null)
                    msg = orig.getMessage() + " <null>:" + getPort();
                else
                    msg = orig.getMessage() + " " +
                            getAddress().toString() + ":" + getPort();
                BindException be = new BindException(msg);
                be.initCause(orig);
                throw be;
            }
        }

}
```
通过上面代码可以看出，最终是调用了`org.apache.tomcat.util.net.ServerSocketFactory#createSocket`方法创建一个`java.net.ServerSocket`，并绑定在conf/server.xml中Connector中配置的端口。

综上我们可以得出如下结论：
>Connector#init的时候，无论是AJP还是HTTP最终其实是调用了JioEndpoint的初始化，默认情况在初始化的时候就会创建java.net.ServerSocket绑到到配置的端口上。

##Connector#start
接着我们再来分析一下Connector#start，因为Connector符合LifeCycle模板方法生命周期管理的机制，因此它的start最终会调用startInternal,org.apache.catalina.connector.Connector#startInternal代码如下：
```java org.apache.catalina.connector.Connector#startInternal
    protected void startInternal() throws LifecycleException {

        // Validate settings before starting
        if (getPort() < 0) {
            throw new LifecycleException(sm.getString(
                    "coyoteConnector.invalidPort", Integer.valueOf(getPort())));
        }

        setState(LifecycleState.STARTING);

        try {
            protocolHandler.start();
        } catch (Exception e) {
            String errPrefix = "";
            if(this.service != null) {
                errPrefix += "service.getName(): \"" + this.service.getName() + "\"; ";
            }

            throw new LifecycleException
                (errPrefix + " " + sm.getString
                 ("coyoteConnector.protocolHandlerStartFailed"), e);
        }

        mapperListener.start();
    }

```
通过上面的代码，我们可以清晰的看到最终调用了protocolHandler.start()，而根据Connector#init流程的分析，这里会分是否采用APR，默认是不采用APR的，这里会根据不同的协议（AJP，HTTP）来调用对应的org.apache.coyote.ProtocolHandler#start.
其中AJP会采用org.apache.coyote.ajp.AjpProtocol，HTTP协议采用org.apache.coyote.http11.Http11Protocol,而无论是AjpProtocol还是Http11Protocol都会调用JIoEndpoint的方法，那么接下来我们就来看看JioEndpoint的start方法，它的代码如下：

```java org.apache.tomcat.util.net.JIoEndpoint#startInternal
public void startInternal() throws Exception {

        if (!running) {
            running = true;
            paused = false;

            // Create worker collection
            if (getExecutor() == null) {
                createExecutor();
            }

            initializeConnectionLatch();

            startAcceptorThreads();

            // Start async timeout thread
            Thread timeoutThread = new Thread(new AsyncTimeout(),
                    getName() + "-AsyncTimeout");
            timeoutThread.setPriority(threadPriority);
            timeoutThread.setDaemon(true);
            timeoutThread.start();
        }
}
```
从上面的代码可以看出，启动了Acceptor线程和AsyncTimeout线程，首先来看看Acceptor线程，我们再来看看startAcceptorThreads方法，代码如下：

```java  org.apache.tomcat.util.net.AbstractEndpoint#startAcceptorThreads
protected final void startAcceptorThreads() {
        int count = getAcceptorThreadCount();
        acceptors = new Acceptor[count];

        for (int i = 0; i < count; i++) {
            acceptors[i] = createAcceptor();
            String threadName = getName() + "-Acceptor-" + i;
            acceptors[i].setThreadName(threadName);
            Thread t = new Thread(acceptors[i], threadName);
            t.setPriority(getAcceptorThreadPriority());
            t.setDaemon(getDaemon());
            t.start();
        }
}
```
通过上面的代码，我们可以看出其实是通过`org.apache.tomcat.util.net.AbstractEndpoint.Acceptor`这个Runable接口的实现类来启动线程，接下来我们就来看看Acceptor#run方法，通过查看run方法，它里面其实就是调用了`java.net.ServerSocket#accept`的方法来接受一个Socket连接。

启动完了Acceptor线程以后，接着就会启动AsyncTimeout线程，而这里面需要注意的时候，无论是Acceptor还是AsyncTimeout线程，它们都是Daemon线程，而设置为Daemon的原因，我们会在下篇[Tomcat的关闭]()中进行说明。

##StandardEngine#start
从本文上面的分析中，我们得知StandardEngine继承了ContainerBase，而StandardEngine的startInternal钩子方法也仅仅是调用了父类ContainerBase的startInternal方法，那接下来我们分析一下ContainerBase的startInternal方法，代码如下：

```java org.apache.catalina.core.ContainerBase#startInternal 
protected synchronized void startInternal() throws LifecycleException {

    // Start our child containers, if any
    Container children[] = findChildren();
    List<Future<Void>> results = new ArrayList<Future<Void>>();
    for (int i = 0; i < children.length; i++) {
        results.add(startStopExecutor.submit(new StartChild(children[i])));
    }

    boolean fail = false;
    for (Future<Void> result : results) {
        try {
            result.get();
        } catch (Exception e) {
            log.error(sm.getString("containerBase.threadedStartFailed"), e);
            fail = true;
        }

    }
    if (fail) {
        throw new LifecycleException(
                sm.getString("containerBase.threadedStartFailed"));
    }

   
    setState(LifecycleState.STARTING);

}
```

我们删除了对本文的分析不相关的代码，只留下一些核心的代码，我们可以看到通过startStopExecutor异步的对子容器进行了启动，然后设置状态为`STARTING`的状态。而startStopExecutor是在容器的initInternal方法中进行初始化好的，接下来我们就来看看StartChild,StardChild的代码如下：
``` java org.apache.catalina.core.ContainerBase.StartChild
private static class StartChild implements Callable<Void> {

        private Container child;

        public StartChild(Container child) {
            this.child = child;
        }

        @Override
        public Void call() throws LifecycleException {
            child.start();
            return null;
        }
}
```

通过上面的代码，我们可以看到StartChild实现了Callable接口，实现这个接口的类可以将其放到对应的executor中执行（对于executor不熟悉的童鞋可以去看一下相关的文章，本文不做介绍），StartChild在运行的时候就会调用到子容器的start方法，而此时的父容器是StandardEngine，子容器就是StandardHost,接下来我们就来看看StandardHost的启动过程。通过前面对于init流程的分析，我们知道StandardHost不是在StandardEngine#init的时候初始化，因此在执行StandardHost#start的时候，要首先进行init方法的调用，具体的代码如下：
```java org.apache.catalina.util.LifecycleBase#start
public final synchronized void start() throws LifecycleException {
    
    
    if (state.equals(LifecycleState.NEW)) {
        init(); //因为此时的StandardHost还没有初始化，因此会走到这一步代码
    } else if (state.equals(LifecycleState.FAILED)){
        stop();
    } else if (!state.equals(LifecycleState.INITIALIZED) &&
            !state.equals(LifecycleState.STOPPED)) {
        invalidTransition(Lifecycle.BEFORE_START_EVENT);
    }

    setStateInternal(LifecycleState.STARTING_PREP, null, false);

    try {
        startInternal();
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        setStateInternal(LifecycleState.FAILED, null, false);
        throw new LifecycleException(
                sm.getString("lifecycleBase.startFail",toString()), t);
    }  
        
   setStateInternal(LifecycleState.STARTED, null, false);
   
}
```
上面省略了部分不相关的代码，在调用中我们可以清楚的看到，对于StandardHost的初始化，是在start的时候进行的。那接下来我们在来看一下StandardHost的init方法，通过查看代码，我们发现StandardHost本身没有实现initInternal的钩子方法，也就意味着最终初始化会调用ContainerBase#initInternal方法，而通过上文的描述，我们已经清楚ContainerBase#initInternal主要是初始化了一个startStopExecutor，这个线程池主要是为了异步的初始化子容器来用的。

>我们知道StandardEngine初始化的时候，也是初始化了一个线程池，而StandardHost也初始化了一个线程池，他们的不同点在与创建线程的工厂方法不同，在采用缺省配置的情况下，StandardEngine的线程池中的线程是以`Catalina-startStop`的形式命名的，而StandardHost是以`localhost-startStop`的方式进行命名的。大家注意区分。

StandardHost#start调用init方法初始化完StandardHost以后，会调用钩子的startInternal方法，而startInternal方法又是调用了ContainerBased#startInternal方法，而ContainerBase#startInternal方法最终又会去启动子容器的，对于StandardHost来说，子容器就是StandardContext。
因此分析到这里我们可以得出如下结论：
> 对于StandardEngine，StandardHost的启动，父容器在init的时候创建一个启动和停止子容器的线程池，然后父容器启动的时候首先通过异步的方式将子容器的启动通过`org.apache.catalina.core.ContainerBase.StartChild`提交到父容器中对应的线程池中进行启动，而子容器启动的时候首先会初始化，然后再启动。

另外这里还需要注意一点就是，StandEngine#start的时候，最终调用了ContainerBase#startInternal方法，而ContainerBase#startInternal的最后，调用了threadStart(),我们来看看它的代码如下：
```java org.apache.catalina.core.ContainerBase#threadStart
protected void threadStart() {

        if (thread != null)
            return;
        if (backgroundProcessorDelay <= 0)
            return;

        threadDone = false;
        String threadName = "ContainerBackgroundProcessor[" + toString() + "]";
        thread = new Thread(new ContainerBackgroundProcessor(), threadName);
        thread.setDaemon(true);
        thread.start();

}
```
上面的代码，首先会判断backgroundProcessorDelay是否小于0，而这个值默认情况下是-1，也就意味这后面的代码不会运行，而对于StandardEngine来说，它将backgroundProcessorDelay的值在构造函数中赋值为了10，这样的话，当StandardEngine启动的时候，就会启动名称为“ContainerBackgroundProcessor[StandardEngine[Catalina]]”的线程。

经过上面的分析，我们已经清楚了StandardEngine启动的过程了，但是我们还有一个地方需要进一步的分析。因为上面的分析我们仅仅只是分析了容器通过conf/server.xml配置文件的配置结构进行的启动，而我们都知道`CATALINA-HOME/webapps/`中的应用也是需要启动的，那么webapps目录的应用又是如何启动的呢？我们下面来分析一下，通过[Tomcat总体结构](/blog/2013/10/16/tomcat-architecture/)的描述，我们已经知道，webapps目录下面的应用其实是属于Context的，而Context对应Tomcat中的StandardContext类，因此我们就知道应该对谁下手了，知道了目标以后，咋们还是采用之前的那种方式，要么debug,要么打印调用栈，这里我们还是通过打印调用栈的方式进行，我们在`org.apache.catalina.core.StandardContext#initInternal`中增加打印调用栈的方法，具体代码如下：
```java org.apache.catalina.core.StandardContext#initInternal 
protected void initInternal() throws LifecycleException {
        super.initInternal();
        Throwable ex = new Throwable();
        StackTraceElement[] stackElements = ex.getStackTrace();
        if (stackElements != null) {
            for (int i = stackElements.length - 1; i >= 0; i--) {
                System.out.print(stackElements[i].getClassName() + "\t");
                System.out.print(stackElements[i].getMethodName() + "\t");
                System.out.print(stackElements[i].getFileName() + "\t");
                System.out.println(stackElements[i].getLineNumber());
            }
        }
        if (processTlds) {
            this.addLifecycleListener(new TldConfig());
        }

        // Register the naming resources
        if (namingResources != null) {
            namingResources.init();
        }

        // Send j2ee.object.created notification 
        if (this.getObjectName() != null) {
            Notification notification = new Notification("j2ee.object.created",
                    this.getObjectName(), sequenceNumber.getAndIncrement());
            broadcaster.sendNotification(notification);
        }
}
```
运行代码，可以看到控制台有如下的输出：
```java terminal info
java.util.concurrent.ThreadPoolExecutor$Worker	run	ThreadPoolExecutor.java	918
java.util.concurrent.ThreadPoolExecutor$Worker	runTask	ThreadPoolExecutor.java	895
java.util.concurrent.FutureTask	run	FutureTask.java	138
java.util.concurrent.FutureTask$Sync	innerRun	FutureTask.java	303
java.util.concurrent.Executors$RunnableAdapter	call	Executors.java	439
org.apache.catalina.startup.HostConfig$DeployDirectory	run	HostConfig.java	1671
org.apache.catalina.startup.HostConfig	deployDirectory	HostConfig.java	1113
org.apache.catalina.core.StandardHost	addChild	StandardHost.java	622
org.apache.catalina.core.ContainerBase	addChild	ContainerBase.java	877
org.apache.catalina.core.ContainerBase	addChildInternal	ContainerBase.java	901
org.apache.catalina.util.LifecycleBase	start	LifecycleBase.java	139
org.apache.catalina.util.LifecycleBase	init	LifecycleBase.java	102
org.apache.catalina.core.StandardContext	initInternal	StandardContext.java	6449
```

通过查看控制台的输出，我们可以看到有一个`org.apache.catalina.startup.HostConfig$DeployDirectory`类，于是乎找到这个类去看看呗。打开一看它是一个Runable接口的实现类，因此我们推断它也是放到某个线程池中进行异步运行的，最终通过IntellIJ IDEA提供的类调用栈分析工具（ctrl+alt+h）得到DeployDirectory构造器方法的调用栈如下图所示：
{% img center /images/2013/10/17/DeployDirectory-call-stack.png %}
通过上图我们可以清楚的看到，最终的调用方是`org.apache.catalina.startup.HostConfig#lifecycleEvent`,到这里我们就知道了Context的启动是通过某个组件的生命周期事件的监听器来启动的，而HostConfig到底是谁的监听器呢？通过名称我们应该可以猜测出它是StandardHost的监听器,那么它到底监听哪个事件呢？我们查看下org.apache.catalina.startup.HostConfig#lifecycleEvent的代码如下：
```java org.apache.catalina.startup.HostConfig#lifecycleEvent
public void lifecycleEvent(LifecycleEvent event) {

        // Identify the host we are associated with
        try {
            host = (Host) event.getLifecycle();
            if (host instanceof StandardHost) {
                setCopyXML(((StandardHost) host).isCopyXML());
                setDeployXML(((StandardHost) host).isDeployXML());
                setUnpackWARs(((StandardHost) host).isUnpackWARs());
            }
        } catch (ClassCastException e) {
            log.error(sm.getString("hostConfig.cce", event.getLifecycle()), e);
            return;
        }

        // Process the event that has occurred
        if (event.getType().equals(Lifecycle.PERIODIC_EVENT)) {
            check();
        } else if (event.getType().equals(Lifecycle.START_EVENT)) {
            start();
        } else if (event.getType().equals(Lifecycle.STOP_EVENT)) {
            stop();
        }
}
```

通过上面的代码，我们可以看出监听的事件是`Lifecycle.START_EVENT`,而通过查看`org.apache.catalina.LifecycleState`的代码`STARTING(true,Lifecycle.START_EVENT)`就可以得知，此时生命周期状态应该是STARTING,到这里我们应该已经猜到了，HostConfig是在StandardHost#start的时候通过监听器调用，为了验证我们的猜测，我们debug一下代码，我们可以在HostConfig#start方法中打个断点，运行以后得到如下内存结构：
{% img center  /images/2013/10/17/HostConfig.png %}
通过上图也就验证了我们刚才的猜测。

通过上面的分析我们清楚了webapps目录中context的启动，总结如下：
>webapps目录中应用的启动在StandardHost#start的时候，通过`Lifecycle.START_EVENT`这个事件的监听器HostConfig进行进一步的启动。

综合上面的文章所述,最后我们再来一下总结，我们知道Java程序启动以后，最终会以进程的形式存在，而Java进程中又会有很多条线程存在，因此最后我们就来看看Tomcat启动以后，到底启动了哪些线程，通过这些我们可以反过来验证我们对源代码的理解是否正确。接下来我们启动Tomcat，然后运行`jstack -l <pid>`来看看，在笔者的机器上面，jstack的输入如下所示：
```java Tomcat threads
Full thread dump Java HotSpot(TM) 64-Bit Server VM (20.51-b01-457 mixed mode):

"ajp-bio-8009-AsyncTimeout" daemon prio=5 tid=7f8738afe000 nid=0x115ad6000 waiting on condition [115ad5000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at org.apache.tomcat.util.net.JIoEndpoint$AsyncTimeout.run(JIoEndpoint.java:148)
        at java.lang.Thread.run(Thread.java:680)

   Locked ownable synchronizers:
        - None

"ajp-bio-8009-Acceptor-0" daemon prio=5 tid=7f8738b05800 nid=0x1159d3000 runnable [1159d2000]
   java.lang.Thread.State: RUNNABLE
        at java.net.PlainSocketImpl.socketAccept(Native Method)
        at java.net.PlainSocketImpl.accept(PlainSocketImpl.java:439)
        - locked <7f46a8710> (a java.net.SocksSocketImpl)
        at java.net.ServerSocket.implAccept(ServerSocket.java:468)
        at java.net.ServerSocket.accept(ServerSocket.java:436)
        at org.apache.tomcat.util.net.DefaultServerSocketFactory.acceptSocket(DefaultServerSocketFactory.java:60)
        at org.apache.tomcat.util.net.JIoEndpoint$Acceptor.run(JIoEndpoint.java:216)
        at java.lang.Thread.run(Thread.java:680)

   Locked ownable synchronizers:
        - None

"http-bio-8080-AsyncTimeout" daemon prio=5 tid=7f8735acb800 nid=0x1158d0000 waiting on condition [1158cf000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at org.apache.tomcat.util.net.JIoEndpoint$AsyncTimeout.run(JIoEndpoint.java:148)
        at java.lang.Thread.run(Thread.java:680)

   Locked ownable synchronizers:
        - None

"http-bio-8080-Acceptor-0" daemon prio=5 tid=7f8735acd000 nid=0x1157cd000 runnable [1157cc000]
   java.lang.Thread.State: RUNNABLE
        at java.net.PlainSocketImpl.socketAccept(Native Method)
        at java.net.PlainSocketImpl.accept(PlainSocketImpl.java:439)
        - locked <7f46a8690> (a java.net.SocksSocketImpl)
        at java.net.ServerSocket.implAccept(ServerSocket.java:468)
        at java.net.ServerSocket.accept(ServerSocket.java:436)
        at org.apache.tomcat.util.net.DefaultServerSocketFactory.acceptSocket(DefaultServerSocketFactory.java:60)
        at org.apache.tomcat.util.net.JIoEndpoint$Acceptor.run(JIoEndpoint.java:216)
        at java.lang.Thread.run(Thread.java:680)

   Locked ownable synchronizers:
        - None

"ContainerBackgroundProcessor[StandardEngine[Catalina]]" daemon prio=5 tid=7f8732850800 nid=0x111203000 waiting on condition [111202000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at org.apache.catalina.core.ContainerBase$ContainerBackgroundProcessor.run(ContainerBase.java:1508)
        at java.lang.Thread.run(Thread.java:680)

   Locked ownable synchronizers:
        - None


"main" prio=5 tid=7f8735000800 nid=0x10843e000 runnable [10843c000]
   java.lang.Thread.State: RUNNABLE
        at java.net.PlainSocketImpl.socketAccept(Native Method)
        at java.net.PlainSocketImpl.accept(PlainSocketImpl.java:439)
        - locked <7f32ea7c8> (a java.net.SocksSocketImpl)
        at java.net.ServerSocket.implAccept(ServerSocket.java:468)
        at java.net.ServerSocket.accept(ServerSocket.java:436)
        at org.apache.catalina.core.StandardServer.await(StandardServer.java:452)
        at org.apache.catalina.startup.Catalina.await(Catalina.java:779)
        at org.apache.catalina.startup.Catalina.start(Catalina.java:725)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
        at java.lang.reflect.Method.invoke(Method.java:597)
        at org.apache.catalina.startup.Bootstrap.start(Bootstrap.java:322)
        at org.apache.catalina.startup.Bootstrap.main(Bootstrap.java:456)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
        at java.lang.reflect.Method.invoke(Method.java:597)
        at com.intellij.rt.execution.application.AppMain.main(AppMain.java:120)
```

上面的截图我已经取消了JVM自己本生的线程，从上图中我们可以清楚的看到，有6条线程，其中`ajp-bio-8009-AsyncTimeout`和`ajp-bio-8009-Acceptor-0`是在Ajp的Connector启动的时候启动的，`http-bio-8080-AsyncTimeout`和`http-bio-8080-Acceptor-0`是http的Connector启动的时候启动的，`ContainerBackgroundProcessor[StandardEngine[Catalina]]`是在StandardEngine启动的时候启动的，而main线程就是我们的主线程。这里还需要注意一点就是除了Main线程以外，其它的线程都是Dameon线程，相关的内容在下篇[Tomcat的关闭](/blog/2013/10/21/tomcat-shutdown/)我们再来详细说明。
