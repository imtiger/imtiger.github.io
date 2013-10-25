---
layout: post
title: "Tomcat关闭过程（Tomcat源代码阅读系列之四）"
date: 2013-10-21 14:45
comments: true
categories: 
- Java
- Tomcat
- 技术
---
{% img center /images/2013/10/14/apache_tomcat_bag.jpg%}
本文是[Tomcat源代码阅读系列](/blog/2013/10/08/tomcat-source-code-study/)的第四篇文章，在阅读此文之前，建议先读前三篇：  
[在IntelliJ IDEA 和 Eclipse运行tomcat 7源代码（Tomcat源代码阅读系列之一）](/blog/2013/10/14/run-tomcat-in-idea-or-eclipse/)   
[Tomcat总体结构                             （Tomcat源代码阅读系列之二）](/blog/2013/10/16/tomcat-architecture/)  
[Tomcat启动过程（Tomcat源代码阅读系列之三）](/blog/2013/10/17/tomcat-start-process/)

我们在[Tomcat启动过程（Tomcat源代码阅读系列之三）](/blog/2013/10/17/tomcat-start-process/)一文中已经知道Tomcat启动以后，会启动6条线程，他们分别如下：
<!-- more -->
```java Tomcat threads
"ajp-bio-8009-AsyncTimeout" daemon prio=5 tid=7f8738afe000 nid=0x115ad6000 waiting on condition [115ad5000]

"ajp-bio-8009-Acceptor-0" daemon prio=5 tid=7f8738b05800 nid=0x1159d3000 runnable [1159d2000]

"http-bio-8080-AsyncTimeout" daemon prio=5 tid=7f8735acb800 nid=0x1158d0000 waiting on condition [1158cf000]

"http-bio-8080-Acceptor-0" daemon prio=5 tid=7f8735acd000 nid=0x1157cd000 runnable [1157cc000]

"ContainerBackgroundProcessor[StandardEngine[Catalina]]" daemon prio=5 tid=7f8732850800 nid=0x111203000 waiting on condition [111202000]

"main" prio=5 tid=7f8735000800 nid=0x10843e000 runnable [10843c000]
```

其中5条是Dameon线程，而对于Java程序来说，当所有非Dameon程序都终止的时候，Jvm就会退出，因此要想终止Tomcat就只需要将main这一条非Dameon线程终止了即可。

>Dameon线程又叫后台或者守护线程，它负责在程序运行期提供一种通用服务的线程，比如垃圾收集线程，非Dameon线程和Dameon线程的区别就在于当程序中所有的非Daemon线程都终止的时候，Jvm会杀死余下的Dameon线程，然后退出。

接下来，我们就来一步步的分析如何来让main线程终止，要想终止它，我们还是得从Tomcat的启动中来寻找答案，我们在分析Tomcat容器启动的时候，在Catalina#start中有一段代码，我们没有关注，接下来就来看看这段代码：
```java org.apache.catalina.startup.Catalina#start
public void start() {
    // ignore other code 
   
    // Register shutdown hook
    // 1 
    if (useShutdownHook) {
        if (shutdownHook == null) {
            shutdownHook = new CatalinaShutdownHook();
        }
        Runtime.getRuntime().addShutdownHook(shutdownHook);

        // If JULI is being used, disable JULI's shutdown hook since
        // shutdown hooks run in parallel and log messages may be lost
        // if JULI's hook completes before the CatalinaShutdownHook()
        LogManager logManager = LogManager.getLogManager();
        if (logManager instanceof ClassLoaderLogManager) {
            ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                    false);
        }
    }

    // 2
    if (await) {
        await();
        stop();
    }
}

```

这里就是Tomcat关闭流程的入口代码了。我在代码中标注了两处，我们首先来看标注1的地方。标注1的代码，我们用到了Jvm的shutdownHook机制。对于shutdownHook大家可以参考[这个连接](http://docs.oracle.com/javase/7/docs/api/java/lang/Runtime.html#addShutdownHook%28java.lang.Thread%29),我这里做一个简单的介绍，shutdown hook是一个已经初始化但是还没有启动的线程，当Jvm关闭的时候，它会启动并并发的运行所有已经注册过的shutdown hooks，知道了这点，我们就来看看`CatalinaShutdownHook`线程做了什么事情？它的代码如下：
```java org.apache.catalina.startup.Catalina.CatalinaShutdownHook#run

public void run() {
    try {
        if (getServer() != null) {
            Catalina.this.stop();
        }
    } catch (Throwable ex) {
        ExceptionUtils.handleThrowable(ex);
        log.error(sm.getString("catalina.shutdownHookFail"), ex);
    } finally {
        // If JULI is used, shut JULI down *after* the server shuts down
        // so log messages aren't lost
        LogManager logManager = LogManager.getLogManager();
        if (logManager instanceof ClassLoaderLogManager) {
            ((ClassLoaderLogManager) logManager).shutdown();
        }
    }
}
```
通过上面的代码，我们可以清楚的看到调用了Catalina的stop方法。而Catalina#stop方法最终又是调用了StandardServer#stop方法和destroy方法。通过这里，我们知道Tomcat利用了shutdown hook机制来在Jvm关闭的时候关闭各个组件。但是Jvm又是何时退出的呢？这就要来看标注为2的代码了。

接下来我们再来看看标注2的地方：在标注为2的代码中，首先判断了await属性是否为true,如果为true就调用await()，调用完以后，再调用stop方法，接下来我们就来看await()方法,而catalina的awit方法又调用了StandardServer#awit方法，它的代码如下：
```java org.apache.catalina.core.StandardServer#await
public void await() {

    // Set up a server socket to wait on
    try {
        awaitSocket = new ServerSocket(port, 1,
                InetAddress.getByName(address));
    } catch (IOException e) {
        log.error("StandardServer.await: create[" + address
                           + ":" + port
                           + "]: ", e);
        return;
    }

    try {
        awaitThread = Thread.currentThread();

        // Loop waiting for a connection and a valid command
        while (!stopAwait) {
            ServerSocket serverSocket = awaitSocket;
            if (serverSocket == null) {
                break;
            }

            // Wait for the next connection
            Socket socket = null;
            StringBuilder command = new StringBuilder();
            try {
                InputStream stream;
                try {
                    socket = serverSocket.accept();
                    socket.setSoTimeout(10 * 1000);  // Ten seconds
                    stream = socket.getInputStream();
                } catch (AccessControlException ace) {
                    log.warn("StandardServer.accept security exception: "
                            + ace.getMessage(), ace);
                    continue;
                } catch (IOException e) {
                    if (stopAwait) {
                        // Wait was aborted with socket.close()
                        break;
                    }
                    log.error("StandardServer.await: accept: ", e);
                    break;
                }

                // Read a set of characters from the socket
                int expected = 1024; // Cut off to avoid DoS attack
                while (expected < shutdown.length()) {
                    if (random == null)
                        random = new Random();
                    expected += (random.nextInt() % 1024);
                }
                while (expected > 0) {
                    int ch = -1;
                    try {
                        ch = stream.read();
                    } catch (IOException e) {
                        log.warn("StandardServer.await: read: ", e);
                        ch = -1;
                    }
                    if (ch < 32)  // Control character or EOF terminates loop
                        break;
                    command.append((char) ch);
                    expected--;
                }
            } finally {
                // Close the socket now that we are done with it
                try {
                    if (socket != null) {
                        socket.close();
                    }
                } catch (IOException e) {
                    // Ignore
                }
            }

            // Match against our command string
            boolean match = command.toString().equals(shutdown);
            if (match) {
                log.info(sm.getString("standardServer.shutdownViaPort"));
                break;
            } else
                log.warn("StandardServer.await: Invalid command '"
                        + command.toString() + "' received");
        }
    } finally {
        ServerSocket serverSocket = awaitSocket;
        awaitThread = null;
        awaitSocket = null;

        // Close the server socket and return
        if (serverSocket != null) {
            try {
                serverSocket.close();
            } catch (IOException e) {
                // Ignore
            }
        }
    }
}
```

通过上面的代码，我们可以看出在配置的端口上通过ServerSocket来监听一个请求的到来，如果请求的字符串和配置的字符串相同的话即跳出循环，这样的话就会运行stop方法，运行完了以后，main线程就退出了。

>这里ServerSocket监听的端口，以及对比的字符串都是在conf/server.xml中配置的，缺省情况下，配置如下：<Server port="8005" shutdown="SHUTDOWN"></Server>,从这里可以看出监听端口为8005,关闭请求发送的字符串为SHUTDOWN.

看到这里，我们基本上已经清楚了Tomcat的关闭就是通过在8005端口，发送一个SHUTDOWN字符串。那么我们就来实验一下。首先启动Tomcat，然后在终端运行如下指令：
```bash telnet
telnet 127.0.0.1 8005
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
SHUTDOWN
Connection closed by foreign host.
```
运行telnet命令，并发送SHUTDOWN字符串以后，我们发现Tomcat就会退出await方法，然后执行stop方法最终停止。

但是一般情况下，我们停止tomcat都不会像上面那种方式来关闭，我们一般有两种方式来关闭：

1. ps aux | grep java ,kill -9 <pid>  
对于这种方式，比较简单粗暴会直接干掉进程，不过这种简单粗暴的方式我也经常用。
2. 运行shutdown.sh  
这种方式其实最终也是向server发送了一个SHUTDOWN字符串，我们接下来分析下第二种情况。  
查看shutdown.sh最终是调用了 catalina.sh，并传递了stop参数。查看catalina.sh脚本，最终其实是调用了 `org.apache.catalina.startup.Bootstrap#main`,并传递参数stop.我们查看Bootstrap#main方法，发现会调用`org.apache.catalina.startup.Bootstrap#stopServer`,而Bootstrap#stopServer通过反射调用了`org.apache.catalina.startup.Catalina#stopServer`,我们来看看Catalina#stopServer方法，代码如下：
```java org.apache.catalina.startup.Catalina#stopServer

public void stopServer(String[] arguments) {

    if (arguments != null) {
        arguments(arguments);
    }

    Server s = getServer();
    // 1 
    if( s == null ) {
        // Create and execute our Digester
        Digester digester = createStopDigester();
        digester.setClassLoader(Thread.currentThread().getContextClassLoader());
        File file = configFile();
        FileInputStream fis = null;
        try {
            InputSource is =
                new InputSource(file.toURI().toURL().toString());
            fis = new FileInputStream(file);
            is.setByteStream(fis);
            digester.push(this);
            digester.parse(is);
        } catch (Exception e) {
            log.error("Catalina.stop: ", e);
            System.exit(1);
        } finally {
            if (fis != null) {
                try {
                    fis.close();
                } catch (IOException e) {
                    // Ignore
                }
            }
        }
    } else {
        // Server object already present. Must be running as a service
        try {
            s.stop();
        } catch (LifecycleException e) {
            log.error("Catalina.stop: ", e);
        }
        return;
    }

    // Stop the existing server
    s = getServer();
    // 2
    if (s.getPort()>0) {
        Socket socket = null;
        OutputStream stream = null;
        try {
            socket = new Socket(s.getAddress(), s.getPort());
            stream = socket.getOutputStream();
            String shutdown = s.getShutdown();
            for (int i = 0; i < shutdown.length(); i++) {
                stream.write(shutdown.charAt(i));
            }
            stream.flush();
        } catch (ConnectException ce) {
            log.error(sm.getString("catalina.stopServer.connectException",
                                   s.getAddress(),
                                   String.valueOf(s.getPort())));
            log.error("Catalina.stop: ", ce);
            System.exit(1);
        } catch (IOException e) {
            log.error("Catalina.stop: ", e);
            System.exit(1);
        } finally {
            if (stream != null) {
                try {
                    stream.close();
                } catch (IOException e) {
                    // Ignore
                }
            }
            if (socket != null) {
                try {
                    socket.close();
                } catch (IOException e) {
                    // Ignore
                }
            }
        }
    } else {
        log.error(sm.getString("catalina.stopServer"));
        System.exit(1);
    }
}
```
我们来分析一下标注的地方：

1. 标注1的代码，此时因为是新开了一个进程,并且conf/server.xml还没有解析，因此s是NULL，通过Digester解析conf/server.xml，最终生成了未初始化的StandardServer对象。
2. 标注2的代码，向standardServer.getPort返回的端口（其实这里面返回即是conf/server.xml中Server根节点配置的port和shutdown属性）发送了standardServer.getShutdown()返回的字符串，而默认情况下这个字符串就是SHUTDOWN.
 
分析到这里，我想大家已经清楚了Tomcat的关闭流程，我们再来总结一下：
Tomcat启动的时候的主线程会在8005端口（默认配置，可以更改）上建立socket监听，当关闭的时候，最终其实就是新起了一个进程然后向Tomcat主线程监听的8005端口发送了一个SHUTDOWN字符串，这样主线程就会结束了，主线程结束了以后，因为其它的线程都是dameon线程，这样依赖Jvm就会退出了。





