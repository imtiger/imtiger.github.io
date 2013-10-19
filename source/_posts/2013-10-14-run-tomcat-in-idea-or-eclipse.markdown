---
layout: post
title: "在IntelliJ IDEA 和 Eclipse运行tomcat 7源代码（Tomcat源代码阅读系列之一）"
date: 2013-10-14 14:39
comments: true
keywods: tomcat,intelliJ IDEA,eclipse,java,tomcat源代码
categories: 
- Tomcat
- Java
- 技术
---
{% img center /images/2013/10/14/apache_tomcat_bag.jpg %}
本文是[Tomcat源代码阅读系列](/blog/2013/10/08/tomcat-source-code-study/)的第一篇文章，在阅读Tomcat源代码之前，我们首先需要将Tomcat的源代码在IDE里面运行起来，这样方便我们阅读的过程中调试。本文总结一下在IDEA 或者 Eclipse中运行Tomcat源代码环境的搭建过程，同时我们通过Maven来负责项目的构建。

在进行搭建之前，我们首先来说一下总体的思路。我们知道Tomcat运行的时候，一部分是源代码编译以后的可运行的Jar,另外一部分则是运行时的环境（也就是我们从官方下载下来的二进制分发包中的一系列的配置文件以及目录结构，说的更直白点就是CATALINA_HOME环境变量指定的目录）,本文对于第一部分采用IntelliJ IDEA 运行tomcat-7.0.42 tag的源代码，而对于第二部分运行环境，我们则直接采用tomcat-7.0.42的二进制分发包。明白了上述的思路以后，咋们就来一步步的搭建吧。

<!-- more -->
首先咋们来看看搭建完成以后的总体的目录结构，然后再一步步的去分解搭建过程。笔者搭建完以后，最终的运行结构如下图所示：
{% img center /images/2013/10/14/project-structure.png %}

下面分别解释一下上图工程结构中涉及到的文件和目录：  

1. .idea和tomcat-study.iml是IntelliJ IDEA的文件，如果你用Eclipse的话不会存在这两个东东 。
2. catalina-home是从官方下载的7.0.42的二进制分发包解压后的目录
3. target是Maven编译项目以后生成的文件夹，熟悉Maven的读者应该很熟悉此目录
4. tomcat-7.0.42-sourcecode是从Tomcat[官方仓库](http://svn.apache.org/repos/asf/tomcat/tc7.0.x/tags/TOMCAT_7_0_42/)下载的tags的源代码
5. pom.xml是Maven的配置文件,此工程中有两个pom.xml，这里运用了Maven聚合的特性。

了解了最终的结构以后，咋们就来一步步的搭建它吧。
###第一步 创建项目目录结构
本文假设我们将项目放在`~/develop/java`目录中。
```bash create project structure
cd ~/develop/java
mkdir Tomcat
cd Tomcat
touch pom.xml
```

###第二步 下载Tomcat 7.0.42二进制分发包
我们通过apache-tomcat-7.0.42的[官方地址](http://mirrors.hust.edu.cn/apache/tomcat/tomcat-7/v7.0.42/bin/apache-tomcat-7.0.42.tar.gz)下载它。具体的过程如下：
```bash download apache-tomcat-7.0.42 binary distribution
cd ~/develop/java/Tomcat
wget http://mirrors.hust.edu.cn/apache/tomcat/tomcat-7/v7.0.42/bin/apache-tomcat-7.0.42.tar.gz
tar -zxvf apache-tomcat-7.0.42.tar.gz
rm apache-tomcat-7.0.42.tar.gz
mv apache-tomcat-7.0.42 catalina-home
``` 

###第三步 下载Tomcat 7.0.42 源代码
接下来我们从Tomcat的[官方SVN仓库](http://svn.apache.org/repos/asf/tomcat/tc7.0.x/tags/TOMCAT_7_0_42/)下载Tomcat 7.0.42源代码，具体的步骤如下：
```bash download Tomcat 7.0.42 source code
cd ~/develop/java/Tomcat
svn co http://svn.apache.org/repos/asf/tomcat/tc7.0.x/tags/TOMCAT_7_0_42/  tomcat-7.0.42-sourcecode
```
在这一步中，我们将7.0.42的源代码迁入到了tomcat-7.0.42-sourcecode目录中。

###第四步 创建聚合模块pom.xml

因为我们通过maven来对项目进行构建，这就需要我们来创建一个pom.xml文件，具体过程如下：
```bash create aggregation child project pom.xml
cd ~/develop/java/Tomcat/tomcat-7.0.42-sourcecode
touch pom.xml
```
用你喜欢的编辑器打开pom.xml然后用下面的内容替换它的内容：

```xml pom.xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">


    <modelVersion>4.0.0</modelVersion>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>Tomcat7.0</artifactId>
    <name>Tomcat7.0</name>
    <version>7.0</version>

    <build>
        <finalName>Tomcat7.0</finalName>
        <sourceDirectory>java</sourceDirectory>
        <testSourceDirectory>test</testSourceDirectory>
        <resources>
            <resource>
                <directory>java</directory>
            </resource>
        </resources>
        <testResources>
            <testResource>
                <directory>test</directory>
            </testResource>
        </testResources>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.3</version>

                <configuration>
                    <encoding>UTF-8</encoding>
                    <source>1.6</source>
                    <target>1.6</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.4</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>ant</groupId>
            <artifactId>ant</artifactId>
            <version>1.7.0</version>
        </dependency>
        <dependency>
            <groupId>wsdl4j</groupId>
            <artifactId>wsdl4j</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>javax.xml</groupId>
            <artifactId>jaxrpc</artifactId>
            <version>1.1</version>
        </dependency>
        <dependency>
            <groupId>org.eclipse.jdt.core.compiler</groupId>
            <artifactId>ecj</artifactId>
            <version>4.2.2</version>
        </dependency>
    </dependencies>

</project>

```
对于pom.xml文件我们需要注意以下几点：

1. 因为下载代码不符合Maven默认的目录结构约定，因此需要修改`resources`和`testResources`为`java`和 `test`，而不是默认的`src/main/resource`和`src/test/resource`,修改`sourceDirectory`和`testSourceDirectory`为，`java`和`test`,而不是默认的`src/main/java`和`src/test/java`.
2. 因为Tomcat源代码的编译需要wsdl4j，jaxrpc,ecj等jar包，因此需要增加相关的依赖。

###第五步 创建项目的根pom.xml文件
这一步我么在Tomcat目录中创建pom.xml文件，这里采用了Maven中聚合的概念.具体过程如下：
```bash create root pom.xml
cd ~/develop/java/Tomcat
touch pom.xml
```
用你喜欢的编辑器打开刚创建的空的pom.xml文件，修改它的内容如下：
```xml ~/develop/java/Tomcat/pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    <groupId>net.imtiger</groupId>
    <artifactId>tomcat-study</artifactId>
    <name>Tomcat 7.0 Study</name>
    <version>1.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>tomcat-7.0.42-sourcecode</module>
    </modules>
</project>
```
###第六步 用IntelliJ IDEA 打开项目根目录的pom.xml
这一步需要注意，要用IDEA 打开项目根目录的pom.xml文件（也就是~/develop/java/Tomcat/pom.xml）

###第七步 运行Tomcat
终于到激动人心的时刻了,我们知道任何Java程序都会有一个`public static void main(String… args)`的入口，Tomcat本身是用Java写的，因此它也不例外，对于Tomcat来说，入口类是`org.apache.catalina.startup.Bootstrap`,我们找到这个类，然后在IntelliJ IDEA中创建一个运行配置，其中最主要的就是VM options的配置了，在VM options里面填写如下的参数：

```java VM options
-Dcatalina.home=catalina-home -Dcatalina.base=catalina-home 
-Djava.endorsed.dirs=catalina-home/endorsed -Djava.io.tmpdir=catalina-home/temp 
-Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager 
-Djava.util.logging.config.file=catalina-home/conf/logging.properties
```
配置好后，IntelliJ IDEA的配置界面如下：
{% img center /images/2013/10/14/vm-options.png %}


点击运行，即可看到Tomcat已经启动了，日志已经打到了IntelliJ IDEA的控制台上了，这个时候可以启动浏览器输入[http://127.0.0.1:8080](http://127.0.0.1:8080)看看是否启动成功。

接下来，咋们在Bootstrap的main方法中，增加一句`System.out.println("Have fun and Enjoy!");`,然后运行一下，看看加入的信息是否被打入到了控制台，在笔者的电脑上打印可以看到信息已经输出如下信息：
```java console log
Have fun and Enjoy!
//ignore other log info
2013-10-14 17:19:35 org.apache.catalina.startup.Catalina start
信息: Server startup in 908 ms
```
上面就是Tomcat7.0.42源代码在IntelliJ IDEA运行环境搭建的完整的过程。因为笔者日常开发采用的是IntelliJ IDEA,所以本文就只写了IntelliJ IDEA的搭建，但是本文采用了Maven来进行构建的，理论上来说其它IDE，比如Eclipse，只要支持Maven,则可以采用本文同样的方法进行，用Eclipse开发的童鞋，按照本文的步骤理论上也是可以运行起来的。

最后，列出几个笔者在搭建的过程中遇到的几个小问题。

1. `org.apache.catalina.connector.TestRequest`类的`prepareRequestBug54984`中有两个特殊字符`äö`,在SVN 迁出的时候变为了乱码，导致Maven在编译的时候编译不过，大家可以复制`äö`替换乱码的字符即可。
2. `CompilationUnit`类中的`public boolean ignoreOptionalProblems()`方法被标记为了@Override，但是其实现的接口`ICompilationUnit`属于`org.eclipse.jdt.core.compiler:ecj`，而3.x版本的`ICompilationUnit`中没有`ignoreOptionalProblems`方法，4.x的版本中才有，因此为了编译通过，本文采用了4.2.2版本。  

另外本文最终搭建好的环境，我已经放在Github上了，不想搭建的童鞋可以直接clone一份使用。[Github仓库地址](https://github.com/imtiger/Tomcat)

下篇：  
[Tomcat总体结构（Tomcat源代码阅读系列之二）](/blog/2013/10/16/tomcat-architecture/) 
	

