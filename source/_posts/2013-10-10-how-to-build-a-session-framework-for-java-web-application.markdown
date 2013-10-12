---
layout: post
title: "如何构建Java web应用的session框架"
date: 2012-05-10 09:31
comments: true
keywords: java web,httpsession,水平伸缩性，无状态架构
categories: 
- Java
- 技术
---

之前写过一个Java web的session框架，并且已经用于生产环境，运行良好。今天将我之前Evernote的笔记重新整理一下发布到博客上供有兴趣的同学参考下，也欢迎各位一起讨论。

做web开发的朋友都知道，一个网站在发展的过程中，随着流量的不断增加，必然会遇到伸缩性的问题，虽然目前硬件的价格已经在减低，有时候可以通过垂直伸缩的方式来达到应对访问量不断增加的麻烦，但是垂直伸缩总是会遇到瓶颈，那么此时就需要水平伸缩了。当水平伸缩的时候，最重要的一点就是状态管理，而web应用的状态通产采用httpsession的管理方式，不同的web server(比如tomcat,jboss,jetty等等)都提供了对httpsession的支持，但是webserver通常采用了集群节点之间互相复制session状态的方式来进行状态管理，这样随着集群节点的增加，集群之间的复制的开销会越来越大，这从某种程度上来说也限制了应用的伸缩性。本文就简单总结一下构建一个Java web 应用的session框架的思路是什么样子。

本文将会从session状态的存储，session信息的管理，安全性问题，以及如何与Servlet Container结合。
>在开始之前，我们首先需要明确一点这里所说的session的概念是广义的，不仅仅是指httpSession。

<!-- more -->
#Session状态存储
咋们首先来谈谈Session状态的存储。我们先来看看平常的日常工作当中，我们是怎么存储Session状态信息的。我们举个例子来说，比如用户的浏览历史，我们可能会将其保存在http cookie中，另外比如用户是否登陆的信息，我们可能选择保存在httpsession之中。上面说了存储到httpsession中会受限于web server的实现，伸缩性有限。那么我们在构建session框架的时候，可以考虑用一个分布式的缓存服务器来存储session状态，比如可以利用memecached服务器来进行存储。  

另外这里面也涉及到另外一个问题，状态的跟踪问题，我们如何区分不同的用户的session信息？这里其实就需要通过cookie来实现了，我们会给每个用户产生的session分配一个唯一的Id，把这个id存放在cookie中，当用户请求服务器的时候会带上sessionId,服务器从cookie中获取sessionId后可以根据Id从缓存中获取到session状态信息。

说到这里，可能有同学会问？为什么我们不能把信息都放到cookie中，这样服务器端都不用存储任何的状态信息，这样对于服务器来说不也是无状态了吗？其实这里面主要涉及安全性以及浏览器的实现问题，因为存储到cookie中的信息是不安全的，黑客可以进行cookie劫持，这样你保存到cookie中的信息就会被非法用户获取了。另外我们知道不同的浏览器对cookie数量以及大小是有限制的，比如IE8限制cookie的大小为4095字节，每个域名cookie的数量为50个，这样以来就可能会遇到cookie丢失的问题。

综上，Session状态的存储，我们需要结合客户端存储和服务器端存储，在客户端存储中，我们借用http cookie来存储sessionId,而session的具体信息我们可以存放到服务器端，而具体实现的过程中，我们可以将起放入分布式缓存服务器中。

#Session信息的管理
接下来我们再来说说Session信息管理，一些公司可能对这块没有什么重视，session状态的管理完全依赖于开发人员自己，开发人员可以随意将信息写入到cookie或者httpsession中，这样造成的问题就是session状态混乱，最后随着开发人员的离职，新来的人只能通过查看源代码的方式来了解session中都放入了什么信息，到后来可能公司没人知道在cookie或者httpsession中到底存放了哪些信息了？这对与系统的维护以及扩展都是不利的，那么怎么解决这个问题？

其实这个时候我们就可以通过session信息的统一配置话管理来解决了。具体来说就是Session框架通过一个配置文件对可放入的session信息进行统一的管理，要想往cookie或者服务器session中放入任何信息都要在配置文件中配置，这样才容许写入。这样要知道session中存放了哪些信息只需要查看配置文件即可知道了。

不过采用配置文件管理session信息了以后，可能又会遇到一个问题，配置文件如何管理？这个不同的公司可以有不同的做法，比如配置文件可以存放在数据库中，session框架启动的时候去数据库查询到最新的配置信息，或者也可以将其放入classpath文件中，session框架通过启动的时候去classpath中获取，另外一些公司都有统一的配置管理服务器，这样可以将session配置也纳入到配置管理服务器中，这样就更加规范了。

#信息安全性问题
上面说了session信息的存储，我们的Session框架要支持两种存储方式，一种是cookie的客户端存储，一种是存储到服务端，当存储到客户端cookie中的，信息容易被非法意图的人窃取，如果什么信息都明文保存在cookie中，那么就存在用户信息泄露的风险。那么此时就需要对放入cookie的信息进行加密处理。关于加密和解密算法本人也没有深入研究过，不过这方面已经有很多人给出了解决方案。我在写Session框架的时候，采用了[Blowfish](http://www.schneier.com/blowfish.html)，有兴趣的同学可以去看看。

#如何与Servlet Container结合
本文的最后，咋们来看看在Java web 开发中，自己开发的Session框架如何与Servlet 容器结合起来。
Servlet规范中有过滤器的概念，过滤器是每个请求过来的时候，可以在请求进入Servlet之前和之后可以做一些通用的事情，那么我们的Session框架可以提供一个SessionFilter纳入到Servlet容器的管理。下面通过一个简单图来形象的描述一下Session框架中主要的角色。
{% img center /images/2012/05/10/sessionFramework.jpg %}
上图中绿色的部分为Session框架的核心部分，我们下面分别来描述一下。
##SessionFilter
SessionFilter的主要职责就是对web server生成的HttpServletRequest和HttpServletReponse进行封装，将其封装为`CustomHttpServletRequest`和`CustomHttpServletReponse`.
SessionFilter的核心代码如下：
```java SessionFilter.java
	@Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        //对原生的HttpServletRequest和HttpServletReponse进行封装。
        CustomSessionServletRequest customRequest = new CustomSessionServletRequest((HttpServletRequest) request);
        CustomSessionServletResponse customResponse = new CustomSessionServletResponse((HttpServletResponse) response); 
        //对于一些静态资源可以不经过session框架过滤
        if (letitgo(request, response, chain, customRequest)) return;
        //reponseBuffer开关，控制服务器刷新响应流的方式，如果打开的话，会等整个请求处理完成后一次性刷到客户端
        if (needResponseBuffered) {
            if (logger.isDebugEnabled()) {
                logger.debug("session framework responseBuffered is on");
            }
            customResponse.setWriterBuffered(true);
        }
        CustomSession customSession = createCustomSession(customRequest, customResponse);
        try {
            chain.doFilter(customRequest, customResponse);
            if (null != customSession) {
                if (logger.isDebugEnabled()) {
                    logger.debug("session framework start to commit session--" + "customSession.commit");
                }
                //将后续业务写入session的信息进行存储，这里就涉及到了将信息写入cookie或者缓存
                customSession.commit();
            }
        } catch (Exception ex) {
            logger.error("session framework occur exception", ex);
            throw new RuntimeException(ex);
        } finally {
            if (logger.isDebugEnabled()) {
                logger.debug("session framework start to commit buffer--" + "customResponse.commitBuffer");
            }
            //将响应流刷到客户端
            customResponse.commitBuffer();
        }
    }
```
##CustomHttpServletRequest
CustomHttpServletRequest包转了原生的HttpServletRequest，它最核心的就是要覆盖getSession方法，主要的代码如下：
```java CustomHttpServletRequest.java
	@Override
    public CustomSession getSession() {
        return session;
    }
    @Override
    public CustomSession getSession(boolean create) {
        return getSession();
    }
```
这样当应用通过getSession返回的则是经过封装以后的代码。
##CustomHttpServletReponse
CustomHttpServletReponse封装了原生的HttpServletReponse,此类的实现的时候需要注意在Servlet3.0之前，不支持httponly的cookie，要写入Httponly的cookie需要手动通过addHeader的方法去加入，而Servlet3.0以后，可以直接通过addCookie方法实现，具体的伪代码如下：
```java
public void addCookie(CustomCookie cookie) {
        if (cookie.isHttpOnly()) {
            addHeader(SET_COOKIE, buildHttpOnlyCookie(cookie));
        } else {
            super.addCookie(cookie);
        }
}
```
另外我们知道标准的Servlet 输出流有一个缓存区，当应用向缓存区写入数据的时候，如果缓存区已经满了就会刷流到客户端了，这样的话就有可能造成一种情况：部分流已经刷到客户端了，但是后来服务器处理抛异常了，这样用户可能看到的状态可能和服务器不一致，为了解决这个问题，我们可以重写getOutputStream和getWriter方法，这两个方法在返回一个经过我们包装的输出流，这样Session框架就可以保留应用写入的数据到最后请求处理完了以后再由SessionFilter刷新流到客户端。具体的伪代码可以参考如下：
```java 
	@Override
    public ServletOutputStream getOutputStream() throws IOException {
        if (this.isWriterBuffered) {
            if (logger.isDebugEnabled()) {
                logger.debug("Created new byte buffer");
            }
            //这里返回一个ByteArrayOutputStream，方便Session框架控制输出流
            ByteArrayOutputStream bytes = new ByteArrayOutputStream();
            stream = new BufferedServletOutputStream(bytes);
            return stream;
        } else {
            this.getSession().commit();
            return super.getOutputStream();
        }
    }
    
    @Override
    public PrintWriter getWriter() throws IOException {
        if (this.isWriterBuffered) {
            if (logger.isDebugEnabled()) {
                logger.debug("response.getWriter(): Created new character buffer");
            }
            //这里返回StringWriter 方便Session框架控制输出流
            StringWriter chars = new StringWriter();
            writer = new BufferedServletWriter(chars);
            return writer;
        } else {
            this.getSession().commit();
            return super.getWriter();
        }
    }
```
另外需要重写的一些方法比如sendError,sendRedirect也需要重写。

##CustomHttpSession
CustomHttpSession主要负责管理Session中的状态信息，它是HttpSession的子类，它会根据Session框架的配置，将不同的信息保存到对应的SessionHolder中，对于CustomHttpSession，我们主要需要重写setAttribute和getAttribute方法。它的伪代码如下：
```java
@Override
    public void setAttribute(String name, Object value) {
        //1. 根据Session框架的配置文件，找到name的属性对应的session配置项
        final SessionConfigItem sessionConfigItem = sessionConfig.getSessionConfigItem(name);
        if (sessionConfigItem == null) {
            return;//如果配置项为空，说明此name的属性没有经过session框架配置，不能写入
        }
        //2. 根据配置类型获取具体的SessionHolder
        final SessionHolder sessionHolder = sessionHolders.get(sessionConfigItem.getHolderType());
        //2. 找到对应的SessionHolder将其存储
        sessionHolder.setAttribute(sessionConfigItem, value);

    }
    @Override
    public Object getAttribute(String name) {
        final SessionConfigItem sessionConfigItem = sessionConfig.getSessionConfigItem(name);
        if (sessionConfigItem == null) {
            return null;
        }
        final SessionHolder sessionHolder = sessionHolders.get(sessionConfigItem.getHolderType());
        if (sessionHolder == null) {
            return null;
        }
        return sessionHolder.getAttribute(sessionConfigItem);
    }
```
##SessionHolder
SessionHolder抽象了Session保存的接口，具体实现可以有好多种，比如你可以选择把session信息保存到cookie中，也可以将其保存到缓存中，甚至你可以将其保存到文件系统中。我自己写的session框架，根据前面的讨论，提供了两种存储方式，CookieHolder和CacheHolder分别对应客户端存储和服务器端缓存存储。在CookieHolder中要涉及到对cookie的解析，保存以及加密等操作，而CacheHolder涉及到从分布式缓存中查询到Session的信息以及同步session信息到缓存等一系列操作，具体代码我就贴了。

上面就是写一个Session框架大体的思路，对此有兴趣的同学可以一起讨论一下。

