---
layout: post
title: "Jvm内存模型以及垃圾收集策略解析系列（二）"
date: 2010-02-21 13:40
comments: true
keywords: java,jvm，内存模型，垃圾收集
categories: 
- Java
- 技术
---

本文是`Jvm内存模型以及垃圾收集策略解析系列`第二篇，第一篇为[Jvm内存模型以及垃圾收集策略解析系列(一)](/blog/2010/02/21/jvm-memory-and-gc/)。  
本文之前发布在本人Iteye的[博客](http://xmuzyq.iteye.com)上，换了新博客后，重新整理一下，发布在此，希望对Jvm 内存以及垃圾收集策略感兴趣的朋友有点帮助。 

[Jvm内存模型以及垃圾收集策略解析系列(一)](/blog/2010/02/21/jvm-memory-and-gc/)介绍了下面的前3部分，本篇文章将介绍第4和第5部分  

1. Java虚拟机规范规定的Jvm 内存概念模型
2. HotSpot Jvm 内存的模型
3. 常见的垃圾收集策略
4. `HotSpot Jvm 垃圾收集策略`
5. `HotSpot Jvm 垃圾收集器的配置策略`

#4.HotSpot Jvm 垃圾收集策略
GC的执行时要耗费一定的CPU资源和时间的，因此在JDK1.2以后，JVM引入了分代收集的策略，其中对新生代采用"Mark-Compact"策略，而对老生代采用了“Mark-Sweep"的策略。其中新生代的垃圾收集器命名为“minor gc”，老生代的GC命名为"Full Gc 或者Major GC".其中用System.gc()强制执行的是Full Gc.
HotSpot Jvm的垃圾收集器按照并发性可以分为如下三种类型：

<!-- more -->

##4.1 串行收集器（Serial Collector）  
Serial Collector是指任何时刻都只有一个线程进行垃圾收集，这种策略有一个名字“stop the whole world",它需要停止整个应用的执行。这种类型的收集器适合于单CPU的机器。
Serial Collector 有如下两个：  

1. Serial Copying Collector:  
此种GC用-XX:UseSerialGC选项配置，它只用于新生代对象的收集。1.5.0以后.  
`-XX:MaxTenuringThreshold`来设置对象复制的次数。当eden空间不够的时候，GC会将eden的活跃对象和一个名叫From survivor空间中尚不够资格放入Old代的对象复制到另外一个名字叫To Survivor的空间。而此参数就是用来说明到底From survivor中的哪些对象不够资格，假如这个参数设置为31，那么也就是说只有对象复制31次以后才算是有资格的对象。
>这里需要注意几个个问题：  
From Survivor和To survivor的角色是不断的变化的，同一时间只有一块空间处于使用状态，这个空间就叫做From Survivor区，当复制一次后角色就发生了变化。  
如果复制的过程中发现To survivor空间已经满了，那么就直接复制到old generation.  
比较大的对象也会直接复制到Old generation,在开发中，我们应该尽量避免这种情况的发生。  

2.  Serial  Mark-Compact Collector：   
串行的标记-整理收集器是JDK5 update6之前默认的老生代的垃圾收集器，此收集使得内存碎片最少化，但是它需要暂停的时间比较长

##4.2 并行收集器（Parallel Collector）  
Parallel Collector主要是为了应对多CPU，大数据量的环境。  
Parallel Collector又可以分为以下三种：  

1. Parallel Copying Collector  
此种GC用-XX:UseParNewGC参数配置,它主要用于新生代的收集,此GC可以配合CMS一起使用，适用于1.4.1以后。
2. Parallel Mark-Compact Collector  
此种GC用-XX:UseParallelOldGC参数配置，此GC主要用于老生代对象的收集。适用于1.6.0以后。
3. Parallel scavenging Collector  
此种GC用-XX:UseParallelGC参数配置，它是对新生代对象的垃圾收集器，但是它不能和CMS配合使用，它适合于比较大新生代的情况，此收集器起始于jdk 1.4.0。它比较适合于对吞吐量高于暂停时间的场合。

串行收集器和并行收集器可以通过如下的图来表示：  

{% img center /images/2010/02/21/serial-and-parallel-gc.jpg %}


##4.3 并发收集器 (Concurrent Collector) 
Concurrent Collector通过并行的方式进行垃圾收集，这样就减少了垃圾收集器收集一次的时间，这种GC在实时性要求高于吞吐量的时候比较有用。此种GC可以用参数-XX:UseConcMarkSweepGC配置，此GC主要用于老生代和Perm代的收集。 
并发收集器可以通过下图形象的描述：
{% img center /images/2010/02/21/concurrent-gc.jpg %}

> 对于并发和并行收集器，我们需要注意一点：并发收集器是指垃圾收集器线程和应用线程可以并发的执行，也就是清除的时候不需要stop the world，但是并行收集器指的的是可以多个线程并行的进行垃圾收集，并行收集器还是要暂停应用的（即所谓的stop the world）


#5.HotSpot Jvm 垃圾收集器的配置策略
通过上面的描述，我们知道HotSpot Jvm中都有哪些垃圾收集器供我们使用，接下来我们总结一下如何配置垃圾收集器。在继续之前我们需要明白，上面所讲的垃圾收集器有些用于新生代，有些用于老年代，并且不是任何两个都可以配对使用的，下面我们通过下图来形象的描述一下哪些收集器可以配对使用：
{% img center /images/2010/02/21/hotspot-gc-collectors.png %}

Ok,知道新生代和老年代垃圾收集器都有哪些收集器以后，咋们接下来看看具体如何来选择垃圾收集器。这需要根据我们的应用特点来进行选择。下面我们分两种情况来分别描述一下不同情况下的垃圾收集配置策略。

##5.1 吞吐量优先
吞吐量是指GC的时间与运行总时间的比值，比如系统运行了100分钟，而GC占用了一分钟，那么吞吐量就是99%，吞吐量优先一般运用于对响应性要求不高的场合，比如web应用，因为网络传输本来就有延迟的问题，GC造成的短暂的暂停使得用户以为是网络阻塞所致。  
吞吐量优先可以通过-XX:GCTimeRatio来指定。当通过-XX:GCTimeRatio不能满足系统的要求以后，我们可以更加细致的来对JVM进行调优。  
首先因为要求高吞吐量，这样就需要一个较大的Young generation，此时就需要引入“`Parallel scavenging Collector`”,可以通过参数：`-XX:UseParallelGC`来配置。  
```java Jvm config
java -server -Xms3072m -Xmx3072m -XX:NewSize=2560m -XX:MaxNewSize=2560 XX:SurvivorRatio=2 - XX:+UseParallelGC 
``` 
当年轻代使用了"`Parallel scavenge collector`"后，老生代就不能使用"CMS"GC了，在JDK1.6之前，此时老生代只能采用串行收集，而JDK1.6引入了并行版本的老生代收集器，可以用参数`-XX:UseParallelOldGC`来配置。

1. 控制并行的线程数  
缺省情况下，Parallel scavenging Collector 会开启与cpu数量相同的线程进行并行的收集，但是也可以调节并行的线程数。假如你想用4个并行的线程去收集Young generation的话，那么就可以配置-XX:ParallelGCThreads=4,此时JVM的配置参数如下：
```java Jvm config
java -server -Xms3072m -Xmx3072m -XX:NewSize=2560m -XX:MaxNewSize=2560 XX:SurvivorRatio=2 -XX:+UseParallelGC -XX:ParallelGCThreads=4
```
2. 自动调节新生代    
在采用了"Parallel scavenge collector"后，此GC会根据运行时的情况自动调节survivor ratio来使得性能最优，因此"Parallel scavenge collector"应该总是开启此参数。此时JVM的参数配置如下：
```java Jvm config
java -server -Xms3072m -Xmx3072m -XX:+UseParallelGC -XX:ParallelGCThreads=4 -XX:+UseAdaptiveSizePolicy
```
##5.2 响应时间优先
响应时间优先是指GC每次运行的时间不能太久，这种情况一般使用与对及时性要求很高的系统，比如股票系统等。  

响应时间优先可以通过参数-XX:MaxGCPauseMillis来配置，配置以后JVM将会自动调节年轻代，老生代的内存分配来满足参数设置。
  
在一般情况下，JVM的默认配置就可以满足要求，只有默认配置不能满足系统的要求时候，才会根据具体的情况来对JVM进行性能调优。如果采用默认的配置不能满足系统的要求，那么此时就可以自己动手来调节。此时"Young generation"可以采用"Parallel copying collector"，而"Old generation"则可以采用"Concurrent Collector".  
举个例子来说，以下参数设置了新生代用Parallel Copying Collector，老生代采用CMS收集器。  
```java
java -server -Xms512m -Xmx512m  -XX:NewSize=64m -XX:MaxNewSize=64m -XX:SurvivorRatio=2  -XX:+UseConcMarkSweepGC -XX:+UseParNewGC 
```
>此时需要注意两个问题：  
>1.如果没有指定-XX:+UseParNewGC，则采用默认的非并行版本的copy collector.  
>2.如果在一个单CPU的系统上设置了-XX:+UseParNewGC ,则默认还是采用缺省的copy collector.

1. 控制并行的线程数  
默认情况下，Parallel copy collector启动和CPU数量一样的线程，也可以通过参数-XX:ParallelGCThreads来指定，比如你想用3个线程去进行并发的复制收集，那么可以改变上述参数如下：
```java
java -server -Xms512m -Xmx512m -XX:NewSize=64m  -XX:MaxNewSize=64m -XX:SurvivorRatio=2        -XX:ParallelGCThreads=4 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC 
```

2. 控制并发收集的临界值
默认情况下，CMS gc在"old generation"空间占用率高于68%的时候，就会进行垃圾收集，而如果想控制收集的临界值，可以通过参数：-XX:CMSInitiatingOccupancyFraction来控制，比如改变上述的JVM配置如下：
```java
java -server -Xms512m -Xmx512m -XX:NewSize=64m -XX:MaxNewSize=64m -XX:SurvivorRatio=2  -XX:ParallelGCThreads=4 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:CMSInitiatingOccupancyFraction=35
```

