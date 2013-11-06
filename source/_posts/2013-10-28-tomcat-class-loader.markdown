---
layout: post
title: "Tomcat类加载器机制（Tomcat源代码阅读系列之六）"
date: 2013-10-28 14:06
comments: true
keywords: Tomcat,Tomcat7,Tomcat 源代码，classloader，类加载器
categories: 
- Java
- Tomcat
- 技术
---
{% img center /images/2013/10/14/apache_tomcat_bag.jpg%}
本文是[Tomcat源代码阅读系列](/blog/2013/10/08/tomcat-source-code-study/)的第六篇文章，本系列前五篇文章如下：    
[在IntelliJ IDEA 和 Eclipse运行tomcat 7源代码（Tomcat源代码阅读系列之一）](/blog/2013/10/14/run-tomcat-in-idea-or-eclipse/)     
[Tomcat总体结构                             （Tomcat源代码阅读系列之二）](/blog/2013/10/16/tomcat-architecture/)    
[Tomcat启动过程（Tomcat源代码阅读系列之三）](/blog/2013/10/17/tomcat-start-process/)   
[Tomcat关闭过程（Tomcat源代码阅读系列之四）](/blog/2013/10/21/tomcat-shutdown/)  
[Tomcat请求处理流程（Tomcat源代码阅读系列之五）](/blog/2013/10/24/tomcat-request-process/)

前面几篇我们分析了Tomcat的启动，关闭以及请求处理的流程，本篇将接着分析Tomcat的类加载器机制，如果大家对Java的类加载器机制不熟悉的话，建议首先熟悉一下Java的类加载器机制后再来查看本篇文章，对于Java的类加载器机制，大家可以参考自行Google或者参考笔者的另外一篇文章：[Java Classloader机制解析](/blog/2009/11/09/java-classloader/).

<!-- more -->

要说Tomcat的Classloader机制，我们还得从Bootstrap开始。在BootStrap初始化的时候，调用了`org.apache.catalina.startup.Bootstrap#initClassLoaders`方法，这个方法里面创建了3个ClassLoader,它们分别是commonLoader,catalinaLoader,sharedLoader，其中catalinaLoader,sharedLoader的父亲加载器是commonLoader，initClassLoaders执行的过程中会执行createClassLoader，而createClassLoader是根据conf/catalina.properties文件中common.loader，server.loader，shared.loader的值来初始化，它的代码如下：
```java org.apache.catalina.startup.Bootstrap#createClassLoader
private ClassLoader createClassLoader(String name, ClassLoader parent)
    throws Exception {

    String value = CatalinaProperties.getProperty(name + ".loader");
    // 1 
    if ((value == null) || (value.equals("")))
        return parent;
    
    // 2
    value = replace(value);

    List<Repository> repositories = new ArrayList<Repository>();

    StringTokenizer tokenizer = new StringTokenizer(value, ",");
    while (tokenizer.hasMoreElements()) {
        String repository = tokenizer.nextToken().trim();
        if (repository.length() == 0) {
            continue;
        }

        // Check for a JAR URL repository
        try {
            @SuppressWarnings("unused")
            URL url = new URL(repository);
            repositories.add(
                    new Repository(repository, RepositoryType.URL));
            continue;
        } catch (MalformedURLException e) {
            // Ignore
        }

        // Local repository
        if (repository.endsWith("*.jar")) {
            repository = repository.substring
                (0, repository.length() - "*.jar".length());
            repositories.add(
                    new Repository(repository, RepositoryType.GLOB));
        } else if (repository.endsWith(".jar")) {
            repositories.add(
                    new Repository(repository, RepositoryType.JAR));
        } else {
            repositories.add(
                    new Repository(repository, RepositoryType.DIR));
        }
    }
    // 3
    ClassLoader classLoader = ClassLoaderFactory.createClassLoader
        (repositories, parent);


    return classLoader;

}
```

以上代码删除了与本篇无关的代码，下面我们分别来分析一下标注的地方：

1.  标注1的代码（第5行）判断如果catalina.properties中没有配置对应的loader属性的话，直接返回父加载器，而默认情况下，server.loader,shared.loader为空，那么此时的catalinaLoader,sharedLoader其实是同一个ClassLoader.
2.  标注2（第9行）的地方根据环境变量的配置替换字符串中的值.默认情况下，common.loader的值为common.loader=${catalina.base}/lib,${catalina.base}/lib/*.jar,${catalina.home}/lib,${catalina.home}/lib/*.jar,这里会将catalina.base和catalina.home用环境变量的值替换。
3.  标注3（第46行）的代码最终调用`org.apache.catalina.startup.ClassLoaderFactory#createClassLoader`静态工厂方法创建了URLClassloader的实例，而具体的URL其实就是*.loader属性配置的内容，此外如果parent为null的话，则直接用系统类加载器。

上面分析了Tomcat在启动的时候，初始化的几个ClassLoader，接下来我们再来继续看看，这些ClassLoader具体都用在什么地方。

我们接着来看org.apache.catalina.startup.Bootstrap#init方法，在初始化完3个classLoader以后，接下来首先通过catalinaLoader加载了`org.apache.catalina.startup.Catalina`l类，然后通过放射调用了org.apache.catalina.startup.Catalina#setParentClassLoader,具体代码片段如下：
```java org.apache.catalina.startup.Bootstrap#init
Class<?> startupClass =
    catalinaLoader.loadClass
    ("org.apache.catalina.startup.Catalina");
Object startupInstance = startupClass.newInstance();

String methodName = "setParentClassLoader";
Class<?> paramTypes[] = new Class[1];
paramTypes[0] = Class.forName("java.lang.ClassLoader");
Object paramValues[] = new Object[1];
paramValues[0] = sharedLoader;
Method method =
    startupInstance.getClass().getMethod(methodName, paramTypes);
method.invoke(startupInstance, paramValues);
```
通过上面的代码，我们可以清楚的看到调用了Catalina的setParentClassLoader放，那么到这里我们可能又要想知道，设置了parentClassLoader以后，sharedLoader又是在哪里使用的呢？这就需要我们接着来分析容器启动的代码。我们通过查看`org.apache.catalina.startup.Catalina#getParentClassLoader`调用栈，我们看到在StandardContext的startInternal方法中调用了它，那么我们查看一下它的代码，包含了如下代码片段：
```java org.apache.catalina.core.StandardContext#startInternal 
if (getLoader() == null) {
            WebappLoader webappLoader = new WebappLoader(getParentClassLoader());
            webappLoader.setDelegate(getDelegate());
            setLoader(webappLoader);
}
try {

    if (ok) {
        
        // Start our subordinate components, if any
        if ((loader != null) && (loader instanceof Lifecycle))
            ((Lifecycle) loader).start();
        //other code    
    }
catch(Exception e){
}
```
通过查看上面的代码，我们看到在StandardContext启动的时候，会创建webapploader，创建webapploader的时候会将getParentClassLoader方法返回的结果（这里返回的其实就是sharedLoader）赋值给自己的parentClassLoader变量,接着又会调用到Webapploader的start方法，因为WebappLoader符合Tomcat组件生命周期管理的模板方法模式，因此会调用到它的startInternal方法。我们接下来就来看看WebappLoader的startInternal，我们摘取一部分与本篇相关的代码片段如下：

```java org.apache.catalina.loader.WebappLoader#startInternal
classLoader = createClassLoader();
classLoader.setResources(container.getResources());
classLoader.setDelegate(this.delegate);
classLoader.setSearchExternalFirst(searchExternalFirst);

```
从上的代码可以看到调用了createClassLoader方法创建一个classLoader，那么我们再看来看看createClassLoader的代码：
```java org.apache.catalina.loader.WebappLoader#createClassLoader 
private WebappClassLoader createClassLoader()
    throws Exception {

    Class<?> clazz = Class.forName(loaderClass);
    WebappClassLoader classLoader = null;

    if (parentClassLoader == null) {
        parentClassLoader = container.getParentClassLoader();
    }
    Class<?>[] argTypes = { ClassLoader.class };
    Object[] args = { parentClassLoader };
    Constructor<?> constr = clazz.getConstructor(argTypes);
    classLoader = (WebappClassLoader) constr.newInstance(args);

    return classLoader;

}
```
在上面的代码里面，loaderClass是WebappLoader的实例变量，其值为`org.apache.catalina.loader.WebappClassLoader`，那么上面的代码其实就是通过反射调用了WebappClassLoader的构造函数，然后传递了sharedLoader作为其父亲加载器。

代码阅读到这里，我们已经基本清楚了Tomcat中ClassLoader的总体结构，总结如下：
在Tomcat存在common,cataina,shared三个公共的classloader,默认情况下，这三个classloader其实是同一个，都是common classloader,而针对每个webapp，也就是context（对应代码中的StandardContext类），都有自己的WebappClassLoader来加载每个应用自己的类。上面的描述，我们可以通过下图形象化的描述：
{% img center /images/2013/10/30/tomcat-classloader.png %}

清楚了Tomcat总体的ClassLoader结构以后，咋们就来进一步来分析一下WebAppClassLoader的代码，我们知道Java的ClassLoader机制有parent-first的机制，而这种机制是在loadClass方法保证的，一般情况下，我们只需要重写findClass方法就好了，而对于WebAppClassLoader，通过查看源代码，我们发现loadClass和findClass方法都进行了重写，那么我们首先就来看看它的loadClass方法,它的代码如下：
```java org.apache.catalina.loader.WebappClassLoader#loadClass
public synchronized Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException {

    if (log.isDebugEnabled())
        log.debug("loadClass(" + name + ", " + resolve + ")");
    Class<?> clazz = null;

    // Log access to stopped classloader
    if (!started) {
        try {
            throw new IllegalStateException();
        } catch (IllegalStateException e) {
            log.info(sm.getString("webappClassLoader.stopped", name), e);
        }
    }

    // (0) Check our previously loaded local class cache
    // 1 
    clazz = findLoadedClass0(name);
    if (clazz != null) {
        if (log.isDebugEnabled())
            log.debug("  Returning class from cache");
        if (resolve)
            resolveClass(clazz);
        return (clazz);
    }

    // (0.1) Check our previously loaded class cache
    // 2
    clazz = findLoadedClass(name);
    if (clazz != null) {
        if (log.isDebugEnabled())
            log.debug("  Returning class from cache");
        if (resolve)
            resolveClass(clazz);
        return (clazz);
    }

    // (0.2) Try loading the class with the system class loader, to prevent
    //       the webapp from overriding J2SE classes
    // 3 
    try {
        clazz = system.loadClass(name);
        if (clazz != null) {
            if (resolve)
                resolveClass(clazz);
            return (clazz);
        }
    } catch (ClassNotFoundException e) {
        // Ignore
    }

    // (0.5) Permission to access this class when using a SecurityManager
    if (securityManager != null) {
        int i = name.lastIndexOf('.');
        if (i >= 0) {
            try {
                securityManager.checkPackageAccess(name.substring(0,i));
            } catch (SecurityException se) {
                String error = "Security Violation, attempt to use " +
                    "Restricted Class: " + name;
                log.info(error, se);
                throw new ClassNotFoundException(error, se);
            }
        }
    }

    //4 
    boolean delegateLoad = delegate || filter(name);

    // (1) Delegate to our parent if requested
    // 5 
    if (delegateLoad) {
        if (log.isDebugEnabled())
            log.debug("  Delegating to parent classloader1 " + parent);
        ClassLoader loader = parent;
        if (loader == null)
            loader = system;
        try {
            clazz = Class.forName(name, false, loader);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Loading class from parent");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }
    }

    // (2) Search local repositories
    if (log.isDebugEnabled())
        log.debug("  Searching local repositories");
    // 6 
    try {
        clazz = findClass(name);
        if (clazz != null) {
            if (log.isDebugEnabled())
                log.debug("  Loading class from local repository");
            if (resolve)
                resolveClass(clazz);
            return (clazz);
        }
    } catch (ClassNotFoundException e) {
        // Ignore
    }

    // (3) Delegate to parent unconditionally
    // 7
    if (!delegateLoad) {
        if (log.isDebugEnabled())
            log.debug("  Delegating to parent classloader at end: " + parent);
        ClassLoader loader = parent;
        if (loader == null)
            loader = system;
        try {
            clazz = Class.forName(name, false, loader);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Loading class from parent");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }
    }

    throw new ClassNotFoundException(name);

}

```
我们一步步的来分析一下上面的代码做了什么事情。  

1. 标注1(第18行)代码，首先从当前ClassLoader的本地缓存中加载类，如果找到则返回。
2. 标注2(第29行)代码，在本地缓存没有的情况下，调用ClassLoader的findLoadedClass方法查看jvm是否已经加载过此类，如果已经加载则直接返回。
3. 标注3(第41行)代码，通过系统的来加载器加载此类，这里防止应用写的类覆盖了J2SE的类,这句代码非常关键，如果不写的话，就会造成你自己写的类有可能会把J2SE的类给替换调，另外假如你写了一个javax.servlet.Servlet类，放在当前应用的WEB-INF/class中，如果没有此句代码的保证，那么你自己写的类就会替换到Tomcat容器Lib中包含的类。
4. 标注4(第68行)代码，判断是否需要委托给父类加载器进行加载，delegate属性默认为false，那么delegatedLoad的值就取决于filter的返回值了，filter方法中根据包名来判断是否需要进行委托加载，默认情况下会返回false.因此delegatedLoad为false
5. 标注5(第72行)代码，因为delegatedLoad为false,那么此时不会委托父加载器去加载，这里其实是没有遵循parent-first的加载机制。
6. 标注6(第96行)调用findClass方法在webapp级别进行加载
7. 标注7(第111行)如果还是没有加载到类，并且不采用委托机制的话，则通过父类加载器去加载。

通过上面的描述，我们可以知道Tomcat在加载webapp级别的类的时候，默认是不遵守parent-first的，这样做的好处是更好的实现了应用的隔离，但是坏处就是加大了内存浪费，同样的类库要在不同的app中都要加载一份。

上面分析完了loadClass，我们接着在来分析一下findClass，通过分析findClass的代码，最终会调用`org.apache.catalina.loader.WebappClassLoader#findClassInternal`方法，那我们就来分析一下它的代码：
```java org.apache.catalina.loader.WebappClassLoader#findClassInternal
protected Class<?> findClassInternal(String name)
    throws ClassNotFoundException {
    
    //
    if (!validate(name))
        throw new ClassNotFoundException(name);

    String tempPath = name.replace('.', '/');
    String classPath = tempPath + ".class";

    ResourceEntry entry = null;

    if (securityManager != null) {
        PrivilegedAction<ResourceEntry> dp =
            new PrivilegedFindResourceByName(name, classPath);
        entry = AccessController.doPrivileged(dp);
    } else {
        // 1 
        entry = findResourceInternal(name, classPath);
    }

    if (entry == null)
        throw new ClassNotFoundException(name);

    Class<?> clazz = entry.loadedClass;
    if (clazz != null)
        return clazz;

    synchronized (this) {
        clazz = entry.loadedClass;
        if (clazz != null)
            return clazz;

        if (entry.binaryContent == null)
            throw new ClassNotFoundException(name);
        
        try {
            // 2
            clazz = defineClass(name, entry.binaryContent, 0,
                    entry.binaryContent.length,
                    new CodeSource(entry.codeBase, entry.certificates));
        } catch (UnsupportedClassVersionError ucve) {
            throw new UnsupportedClassVersionError(
                    ucve.getLocalizedMessage() + " " +
                    sm.getString("webappClassLoader.wrongVersion",
                            name));
        }
        entry.loadedClass = clazz;
        entry.binaryContent = null;
        entry.source = null;
        entry.codeBase = null;
        entry.manifest = null;
        entry.certificates = null;
    }

    return clazz;

}
```
上面的代码标注1（第19行）的地方通过名称去当前webappClassLoader的仓库中查找对应的类文件，标注2(第38行)的代码，将找到的类文件通过defineClass转变为Jvm可以识别的Class对象返回。
