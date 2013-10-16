---
layout: post
title: "Tomcat总体结构（Tomcat源代码阅读系列之二）"
date: 2013-10-16 14:33
comments: true
keywords: tomcat源代码，java,tomcat
categories: 
- Tomcat
- Java
- 技术
---

我们在[在IntelliJ IDEA 和 Eclipse运行tomcat 7源代码](/blog/2013/10/14/run-tomcat-in-idea-or-eclipse/)一文中介绍了如何在intelliJ IDEA 和 Eclipse中运行Tomcat源代码，本文介绍一下Tomcat的总体结构。 
>本文没有特别指明的地方，源代码都是针对tomcat7.0.42来说。

#Tomcat的总体结构
Tomcat即是一个Http服务器也是一个Servlet容器，它的总体结构我们可以用下图来描述：
{% img center /images/2013/10/16/TomcatArchitecture.png %}
<!-- more -->
通过上图我们可以看出Tomcat中主要涉及Server,Service,Engine,Connector,Host,Context组件，之前用过Tomcat的童鞋是不是觉得这些组件的名称有点似曾相识的赶脚，没赶脚？！您再想想。好吧，不用你想了，我来告诉你吧。其实在Tomcat二进制分发包解压后,在conf目录中有一个server.xml文件，你打开它瞄两眼看看，是不是发现server.xml文件中已经包含了上述的几个名称。我拿我本地Tomcat7.0.42分发包中的server.xml来具体分析一下，它的内容如下：
```xml conf/server.xml(Tomcat 7.0.42)
<?xml version='1.0' encoding='utf-8'?>
<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<!-- Note:  A "Server" is not itself a "Container", so you may not
     define subcomponents such as "Valves" at this level.
     Documentation at /docs/config/server.html
 -->
<Server port="8005" shutdown="SHUTDOWN">
  <!-- Security listener. Documentation at /docs/config/listeners.html
  <Listener className="org.apache.catalina.security.SecurityListener" />
  -->
  <!--APR library loader. Documentation at /docs/apr.html -->
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <!--Initialize Jasper prior to webapps are loaded. Documentation at /docs/jasper-howto.html -->
  <Listener className="org.apache.catalina.core.JasperListener" />
  <!-- Prevent memory leaks due to use of particular java/javax APIs-->
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <!-- Global JNDI resources
       Documentation at /docs/jndi-resources-howto.html
  -->
  <GlobalNamingResources>
    <!-- Editable user database that can also be used by
         UserDatabaseRealm to authenticate users
    -->
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <!-- A "Service" is a collection of one or more "Connectors" that share
       a single "Container" Note:  A "Service" is not itself a "Container",
       so you may not define subcomponents such as "Valves" at this level.
       Documentation at /docs/config/service.html
   -->
  <Service name="Catalina">

    <!--The connectors can use a shared executor, you can define one or more named thread pools-->
    <!--
    <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
        maxThreads="150" minSpareThreads="4"/>
    -->


    <!-- A "Connector" represents an endpoint by which requests are received
         and responses are returned. Documentation at :
         Java HTTP Connector: /docs/config/http.html (blocking & non-blocking)
         Java AJP  Connector: /docs/config/ajp.html
         APR (HTTP/AJP) Connector: /docs/apr.html
         Define a non-SSL HTTP/1.1 Connector on port 8080
    -->
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    <!-- A "Connector" using the shared thread pool-->
    <!--
    <Connector executor="tomcatThreadPool"
               port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    -->
    <!-- Define a SSL HTTP/1.1 Connector on port 8443
         This connector uses the JSSE configuration, when using APR, the
         connector should be using the OpenSSL style configuration
         described in the APR documentation -->
    <!--
    <Connector port="8443" protocol="HTTP/1.1" SSLEnabled="true"
               maxThreads="150" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS" />
    -->

    <!-- Define an AJP 1.3 Connector on port 8009 -->
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />


    <!-- An Engine represents the entry point (within Catalina) that processes
         every request.  The Engine implementation for Tomcat stand alone
         analyzes the HTTP headers included with the request, and passes them
         on to the appropriate Host (virtual host).
         Documentation at /docs/config/engine.html -->

    <!-- You should set jvmRoute to support load-balancing via AJP ie :
    <Engine name="Catalina" defaultHost="localhost" jvmRoute="jvm1">
    -->
    <Engine name="Catalina" defaultHost="localhost">

      <!--For clustering, please take a look at documentation at:
          /docs/cluster-howto.html  (simple how to)
          /docs/config/cluster.html (reference documentation) -->
      <!--
      <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>
      -->

      <!-- Use the LockOutRealm to prevent attempts to guess user passwords
           via a brute-force attack -->
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <!-- This Realm uses the UserDatabase configured in the global JNDI
             resources under the key "UserDatabase".  Any edits
             that are performed against this UserDatabase are immediately
             available for use by the Realm.  -->
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

        <!-- SingleSignOn valve, share authentication between web applications
             Documentation at: /docs/config/valve.html -->
        <!--
        <Valve className="org.apache.catalina.authenticator.SingleSignOn" />
        -->

        <!-- Access log processes all example.
             Documentation at: /docs/config/valve.html
             Note: The pattern used is equivalent to using pattern="common" -->
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log." suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />

      </Host>
    </Engine>
  </Service>
</Server>

```
接下来我们就根据上图以及conf/server.xml的内容来一步步描述一下上面所说的各种组件吧。

##Server
首先闪联登场的是咋们的Server大哥（大家能给点掌声吗？），Server是Tomcat中最顶层的组件，它可以包含多个Service组件。在Tomcat源代码中Server组件对应源码中的`org.apache.catalina.core.StandardServer`类。StandardServer的继承关系图如下图所示：
{% img center /images/2013/10/16/StandardServer.jpg %}

##Service
接下来咋们来看看Service组件，Service组件相当于Connetor和Engine组件的包装器，它将一个或者多个Connector组件和一个Engine建立关联。缺省的的配置文件中，定义一个叫`Catalina`的服务，并将Http,AJP这两个Connector关联到了一个名为`Catalina`的Engine.Service组件对应Tomcat源代码中的`org.apache.catalina.core.StandardService`,StandardService的继承关系图如下图所示：
{% img center /images/2013/10/16/StandardService.png %}

##Connector
既然Tomcat需要提供http服务，而我们知道http应用层协议最终都是需要通过TCP层的协议进行包传递的，而Connector正是Tomcat中监听TCP网络连接的组件，一个Connector会监听一个独立的端口来处理来自客户端的连接。缺省的情况下Tomcat提供了如下两个Connector。我们分别描述一下：

1. HTTP/1.1  
`<Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />`
上面定义了一个Connector，它缺省监听端口8080,这个端口我们可以根据具体情况进行改动。`connectionTimeout`定义了连接超时时间，单位是毫秒，`redirectPort`定义了ssl的重定向接口，根据缺省的配置，Connector会将ssl请求重定向到8443端口。
2. AJP/1.3  
AJP表示`Apache Jserv Protocol`,此连接器将处理Tomcat和Aapache http服务器之间的交互，这个连机器是用来处理我们将Tomcat和Apache http服务器结合使用的情况。假如在同样的一台物理Server上面部署了一台Apache http服务器和多台Tomcat服务器，通过Apache服务器来处理静态资源以及负载均衡的时候，针对不同的Tomcat实例需要AJP监听不同的端口。

Connector对应源代码中的`org.apache.catalina.connector.Connector`,它的继承关系图如下所示：
{% img center /images/2013/10/16/Connector.png %}

##Engine
Tomcat中有一个容器的概念，而Engine,Host,Context都属于Contanier，我们先来说说最顶层的容器Engine.  
一个Engine可以包含一个或者多个Host,也就是说我们一个Tomcat的实例可以配置多个虚拟主机。  
缺省的情况下`<Engine name="Catalina" defaultHost="localhost">`定义了一个名称为Cataline的Engine.Engine对应源代码中的`org.apache.catalina.core.StandardEngine`，它的继承关系图如下图所示：
{% img center /images/2013/10/16/StandardEngine.png %}

##Host
Host定义了一个虚拟主机，一个虚拟主机可以有多个Context，缺省的配置如下：  
`<Host name="localhost"  appBase="webapps" unpackWARs="true" autoDeploy="true">….</Host>` 其中`appBase`为webapps，也就是`<CATALINA_HOME>\webapps`目录，`unpackingWARS`属性指定在appBase指定的目录中的war包都自动的解压，缺省配置为true,`autoDeploy`属性指定是否对加入到appBase目录的war包进行自动的部署，缺省为true.  
Host对应源代码中的`org.apache.catalina.core.StandardHost`,它的继承关系图如下所示：
{% img center /images/2013/10/16/StandardHost.png %}

##Context
在Tomcat中，每一个运行的webapp其实最终都是以Context的形成存在，每个Context都有一个根路径和请求URL路径，Context对应源代码中的`org.apache.catalina.core.StandardContext`,它的继承关系图如下图所示：
{% img center /images/2013/10/16/StandardContext.png %}
在Tomcat中我们通常采用如下的两种方式创建一个Context.下面分别描述一下：

1. 在`<CATALINA-HOME>\webapps`目录中创建一个目录，这个时候将自动创建一个context，默认context的访问url为`http://host:port/dirname`,你也可以通过在`ContextRoot\META-INF`中创建一个context.xml的文件，其中包含如下的内容来指定应用的访问路径。
`<Context path="/yourUrlPath" />`
2. conf\server.xml文件中增加context元素。
第二种创建context的方法，我们可以选择在server.xml文件的`<Host>`元素，比如我们在server.xml文件中增加如下内容：
```xml server.xml
        ......
        ......
        <Context path="/mypath" docBase="/Users/tiger/develop/xxx" reloadable="true">
        </Context>
      </Host>
    </Engine>
  </Service>
</Server>
```
这样的话，我们就可以通过`http://host:port/mypath`访问上面配置的context了。

