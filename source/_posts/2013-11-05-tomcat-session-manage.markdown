---
layout: post
title: "Tomcat Session管理机制（Tomcat源代码阅读系列之七）"
date: 2013-11-05 15:39
comments: true
keywords: Tomcat,Tomcat7,Tomcat 源代码，HttpSession,Tomcat session管理
categories: 
- 技术
- Java
- Tomcat
---
{% img center /images/2013/10/14/apache_tomcat_bag.jpg%}
本文是[Tomcat源代码阅读系列](/blog/2013/10/08/tomcat-source-code-study/)的第七篇文章，本系列前六篇文章如下：    
[在IntelliJ IDEA 和 Eclipse运行tomcat 7源代码（Tomcat源代码阅读系列之一）](/blog/2013/10/14/run-tomcat-in-idea-or-eclipse/)     
[Tomcat总体结构                             （Tomcat源代码阅读系列之二）](/blog/2013/10/16/tomcat-architecture/)    
[Tomcat启动过程（Tomcat源代码阅读系列之三）](/blog/2013/10/17/tomcat-start-process/)   
[Tomcat关闭过程（Tomcat源代码阅读系列之四）](/blog/2013/10/21/tomcat-shutdown/)  
[Tomcat请求处理流程（Tomcat源代码阅读系列之五）](/blog/2013/10/24/tomcat-request-process/)  
[Tomcat类加载器机制（Tomcat源代码阅读系列之六）](/blog/2013/10/28/tomcat-class-loader/)    

前面几篇我们分析了Tomcat的启动，关闭，请求处理的流程，tomcat的classloader机制，本篇将接着分析Tomcat的session管理方面的内容。

<!-- more -->
在开始之前，我们先来看一下总体上的结构，熟悉了总体结构以后，我们在一步步的去分析源代码。Tomcat session相光的类图如下：
{% img center /images/2013/11/06/tomcat-session.jpg %}
通过上图，我们可以看出每一个StandardContext会关联一个Manager,默认情况下Manager的实现类是StandardManager，而StandardManager内部会聚合多个Session，其中StandardSession是Session的默认实现类，当我们调用Request.getSession的时候，Tomcat通过StandardSessionFacade这个外观类将StandardSession包装以后返回。

上面清楚了总体的结构以后，我们来进一步的通过源代码来分析一下。咋们首先从Request的getSession方法看起。
```java org.apache.catalina.connector.Request#getSession
public HttpSession getSession() {
    Session session = doGetSession(true);
    if (session == null) {
        return null;
    }

    return session.getSession();
}
```
从上面的代码，我们可以看出首先首先调用doGetSession方法获取Session，然后再调用Session的getSession方法返回HttpSession，那接下来我们再来看看doGetSession方法：
```java org.apache.catalina.connector.Request#doGetSession
protected Session doGetSession(boolean create) {

    // There cannot be a session if no context has been assigned yet
    if (context == null) {
        return (null);
    }

    // Return the current session if it exists and is valid
    if ((session != null) && !session.isValid()) {
        session = null;
    }
    if (session != null) {
        return (session);
    }

    // Return the requested session if it exists and is valid
    // 1 
    Manager manager = null;
    if (context != null) {
        manager = context.getManager();
    }
    if (manager == null)
     {
        return (null);      // Sessions are not supported
    }
    // 2
    if (requestedSessionId != null) {
        try {
            session = manager.findSession(requestedSessionId);
        } catch (IOException e) {
            session = null;
        }
        if ((session != null) && !session.isValid()) {
            session = null;
        }
        if (session != null) {
            session.access();
            return (session);
        }
    }

    // Create a new session if requested and the response is not committed
    // 3
    if (!create) {
        return (null);
    }
    if ((context != null) && (response != null) &&
        context.getServletContext().getEffectiveSessionTrackingModes().
                contains(SessionTrackingMode.COOKIE) &&
        response.getResponse().isCommitted()) {
        throw new IllegalStateException
          (sm.getString("coyoteRequest.sessionCreateCommitted"));
    }

    // Attempt to reuse session id if one was submitted in a cookie
    // Do not reuse the session id if it is from a URL, to prevent possible
    // phishing attacks
    // Use the SSL session ID if one is present.
    // 4
    if (("/".equals(context.getSessionCookiePath())
            && isRequestedSessionIdFromCookie()) || requestedSessionSSL ) {
        session = manager.createSession(getRequestedSessionId());
    } else {
        session = manager.createSession(null);
    }

    // Creating a new session cookie based on that session
    if ((session != null) && (getContext() != null)
           && getContext().getServletContext().
                   getEffectiveSessionTrackingModes().contains(
                           SessionTrackingMode.COOKIE)) {
        // 5 
        Cookie cookie =
            ApplicationSessionCookieConfig.createSessionCookie(
                    context, session.getIdInternal(), isSecure());

        response.addSessionCookieInternal(cookie);
    }

    if (session == null) {
        return null;
    }

    session.access();
    return session;
}

```
下面我们就来重点分析一下，上面代码中标注了数字的地方：

1. 标注1（第17行）首先从StandardContext中获取对应的Manager对象，缺省情况下，这个地方获取的其实就是StandardManager的实例。
2. 标注2（第26行）从Manager中根据requestedSessionId获取session，如果session已经失效了，则将session置为null以便下面创建新的session,如果session不为空则通过调用session的access方法标注session的访问时间，然后返回。
3. 标注3（第43行）判断传递的参数，如果为false，则直接返回空，这其实就是对应的Request.getSession(true/false)的情况，当传递false的时候，如果不存在session，则直接返回空，不会新建。
4. 标注4 （第59行）调用Manager来创建一个新的session，这里默认会调用到StandardManager的方法，而StandardManager继承了ManagerBase，那么默认其实是调用了了ManagerBase的方法。
5. 标注5 (第72行)创建了一个Cookie，而Cookie的名称就是大家熟悉的JSESSIONID，另外JSESSIONID其实也是可以配置的，这个可以通过context节点的sessionCookieName来修改。比如<Context sessionCookieName="yoursessionId">...</Context>. 

通过doGetSession获取到Session了以后，我们发现调用了session.getSession方法，而Session的实现类是StandardSession，那么我们再来看下StandardSession的getSession方法。
```java org.apache.catalina.session.StandardSession#getSession
public HttpSession getSession() {

    if (facade == null){
        if (SecurityUtil.isPackageProtectionEnabled()){
            final StandardSession fsession = this;
            facade = AccessController.doPrivileged(
                    new PrivilegedAction<StandardSessionFacade>(){
                @Override
                public StandardSessionFacade run(){
                    return new StandardSessionFacade(fsession);
                }
            });
        } else {
            facade = new StandardSessionFacade(this);
        }
    }
    return (facade);

}

```
通过上面的代码，我们可以看到通过StandardSessionFacade的包装类将StandardSession包装以后返回。到这里我想大家应该熟悉了Session创建的整个流程。  

接着我们再来看看，Sesssion是如何被销毁的。我们在[Tomcat启动过程（Tomcat源代码阅读系列之三）](/blog/2013/10/17/tomcat-start-process/)中之处，在容器启动以后会启动一个ContainerBackgroundProcessor线程，这个线程是在Container启动的时候启动的，这条线程就通过后台周期性的调用org.apache.catalina.core.ContainerBase#backgroundProcess，而backgroundProcess方法最终又会调用org.apache.catalina.session.ManagerBase#backgroundProcess，接下来我们就来看看Manger的backgroundProcess方法。

```java org.apache.catalina.session.ManagerBase#backgroundProcess 
public void backgroundProcess() {
    count = (count + 1) % processExpiresFrequency;
    if (count == 0)
        processExpires();
}
```
上面的代码里，需要注意一下，默认情况下backgroundProcess是每10秒运行一次（StandardEngine构造的时候，将backgroundProcessorDelay设置为了10），而这里我们通过processExpiresFrequency来控制频率，例如processExpiresFrequency的值默认为6，那么相当于没一分钟运行一次processExpires方法。接下来我们再来看看processExpires。
```java org.apache.catalina.session.ManagerBase#processExpires
public void processExpires() {

    long timeNow = System.currentTimeMillis();
    Session sessions[] = findSessions();
    int expireHere = 0 ;
    
    if(log.isDebugEnabled())
        log.debug("Start expire sessions " + getName() + " at " + timeNow + " sessioncount " + sessions.length);
    for (int i = 0; i < sessions.length; i++) {
        if (sessions[i]!=null && !sessions[i].isValid()) {
            expireHere++;
        }
    }
    long timeEnd = System.currentTimeMillis();
    if(log.isDebugEnabled())
         log.debug("End expire sessions " + getName() + " processingTime " + (timeEnd - timeNow) + " expired sessions: " + expireHere);
    processingTime += ( timeEnd - timeNow );

}
```
上面的代码比较简单，首先查找出当前context的所有的session，然后调用session的isValid方法，接下来我们在看看Session的isValid方法。
```java org.apache.catalina.session.StandardSession#isValid
public boolean isValid() {

    if (this.expiring) {
        return true;
    }

    if (!this.isValid) {
        return false;
    }

    if (ACTIVITY_CHECK && accessCount.get() > 0) {
        return true;
    }

    if (maxInactiveInterval > 0) {
        long timeNow = System.currentTimeMillis();
        int timeIdle;
        if (LAST_ACCESS_AT_START) {
            timeIdle = (int) ((timeNow - lastAccessedTime) / 1000L);
        } else {
            timeIdle = (int) ((timeNow - thisAccessedTime) / 1000L);
        }
        if (timeIdle >= maxInactiveInterval) {
            expire(true);
        }
    }

    return (this.isValid);
}

```
查看上面的代码，主要就是通过对比当前时间和上次访问的时间差是否大于了最大的非活动时间间隔，如果大于就会调用expire(true)方法对session进行超期处理。这里需要注意一点，默认情况下LAST_ACCESS_AT_START为false，读者也可以通过设置系统属性的方式进行修改，而如果采用LAST_ACCESS_AT_START的时候，那么请求本身的处理时间将不算在内。比如一个请求处理开始的时候是10:00,请求处理花了1分钟，那么如果LAST_ACCESS_AT_START为true，则算是否超期的时候，是从10:00算起，而不是10:01。

接下来我们再来看看expire方法，代码如下：
```java org.apache.catalina.session.StandardSession#expire
public void expire(boolean notify) {

    // Check to see if expire is in progress or has previously been called
    if (expiring || !isValid)
        return;

    synchronized (this) {
        // Check again, now we are inside the sync so this code only runs once
        // Double check locking - expiring and isValid need to be volatile
        if (expiring || !isValid)
            return;

        if (manager == null)
            return;

        // Mark this session as "being expired"
        // 1         
        expiring = true;

        // Notify interested application event listeners
        // FIXME - Assumes we call listeners in reverse order
        Context context = (Context) manager.getContainer();

        // The call to expire() may not have been triggered by the webapp.
        // Make sure the webapp's class loader is set when calling the
        // listeners
        ClassLoader oldTccl = null;
        if (context.getLoader() != null &&
                context.getLoader().getClassLoader() != null) {
            oldTccl = Thread.currentThread().getContextClassLoader();
            if (Globals.IS_SECURITY_ENABLED) {
                PrivilegedAction<Void> pa = new PrivilegedSetTccl(
                        context.getLoader().getClassLoader());
                AccessController.doPrivileged(pa);
            } else {
                Thread.currentThread().setContextClassLoader(
                        context.getLoader().getClassLoader());
            }
        }
        try {
            // 2
            Object listeners[] = context.getApplicationLifecycleListeners();
            if (notify && (listeners != null)) {
                HttpSessionEvent event =
                    new HttpSessionEvent(getSession());
                for (int i = 0; i < listeners.length; i++) {
                    int j = (listeners.length - 1) - i;
                    if (!(listeners[j] instanceof HttpSessionListener))
                        continue;
                    HttpSessionListener listener =
                        (HttpSessionListener) listeners[j];
                    try {
                        context.fireContainerEvent("beforeSessionDestroyed",
                                listener);
                        listener.sessionDestroyed(event);
                        context.fireContainerEvent("afterSessionDestroyed",
                                listener);
                    } catch (Throwable t) {
                        ExceptionUtils.handleThrowable(t);
                        try {
                            context.fireContainerEvent(
                                    "afterSessionDestroyed", listener);
                        } catch (Exception e) {
                            // Ignore
                        }
                        manager.getContainer().getLogger().error
                            (sm.getString("standardSession.sessionEvent"), t);
                    }
                }
            }
        } finally {
            if (oldTccl != null) {
                if (Globals.IS_SECURITY_ENABLED) {
                    PrivilegedAction<Void> pa =
                        new PrivilegedSetTccl(oldTccl);
                    AccessController.doPrivileged(pa);
                } else {
                    Thread.currentThread().setContextClassLoader(oldTccl);
                }
            }
        }

        if (ACTIVITY_CHECK) {
            accessCount.set(0);
        }
        setValid(false);

        // Remove this session from our manager's active sessions
        // 3 
        manager.remove(this, true);

        // Notify interested session event listeners
        if (notify) {
            fireSessionEvent(Session.SESSION_DESTROYED_EVENT, null);
        }

        // Call the logout method
        if (principal instanceof GenericPrincipal) {
            GenericPrincipal gp = (GenericPrincipal) principal;
            try {
                gp.logout();
            } catch (Exception e) {
                manager.getContainer().getLogger().error(
                        sm.getString("standardSession.logoutfail"),
                        e);
            }
        }

        // We have completed expire of this session
        expiring = false;

        // Unbind any objects associated with this session
        // 4
        String keys[] = keys();
        for (int i = 0; i < keys.length; i++)
            removeAttributeInternal(keys[i], notify);

    }

}

```
上面代码的主流程我已经标注了数字，我们来逐一分析一下：

1. 标注1（第18行）标记当前的session为超期
2. 标注2（第41行）出发HttpSessionListener监听器的方法。
3. 标注3（第89行）从Manager里面移除当前的session
4. 标注4（第113行）将session中保存的属性移除。

到这里我们已经清楚了Tomcat中对与StandardSession的创建以及销毁的过程，其实StandardSession仅仅是实现了内存中Session的存储，而Tomcat还支持将Session持久化，以及Session集群节点间的同步。这些内容我们以后再来分析。
