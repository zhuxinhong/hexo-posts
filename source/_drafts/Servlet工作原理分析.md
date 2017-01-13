---
title: Servlet工作原理分析
date: 2016-05-22
categories: Java Web
tags: 
- Java
- Servlet
---

#	Tomcat中的Servlet容器
在Tomcat容器等级中，Context容器直接管理Servlet容器中的包装类Wrapper，所以Context容器如何运行将直接影响Servlet的工作方式。
Tomcat容器模型：  
1.	Container容器  
2. 	Engine  
3. 	Host  
4. 	Servlet容器	
5. Context	

Tomcat容器分为4个等级，管理Servlet的容器是Context容器，一个Context对应一个Web工程。

#	Servlet容器启动
Tomcat7开始支持嵌入式功能，增加了一个启动类org.apache.catalina.startup.Tomcat。创建一个实例对象并调用start方法就可以很容易地启动Tomcat。

```
 public Context addWebapp(Host host, String url, String name, String path) {
        silence(host, url);

        Context ctx = new StandardContext();
        ctx.setName(name);
        ctx.setPath(url);
        ctx.setDocBase(path);

        ctx.addLifecycleListener(new DefaultWebXmlListener());
        
        ContextConfig ctxCfg = new ContextConfig();
        ctx.addLifecycleListener(ctxCfg);
        
        // prevent it from looking ( if it finds one - it'll have dup error )
        ctxCfg.setDefaultWebXml(noDefaultWebXmlPath());

        if (host == null) {
            getHost().addChild(ctx);
        } else {
            host.addChild(ctx);
        }

        return ctx;
    }
```

一个Web应用对应一个Context容器，也就是Servlet运行时的Servlet容器。添加一个Web应用时将会创建一个StandardContext容器，并给这个容器设置必要的参数，url和path分别代表这个应用在Tomcat中的访问路径和这个应用实际的物理路径。

```
public void start() throws LifecycleException {
    getServer();
    getConnector();
    server.start();
}
```

调用Tomcat的start方法启动Tomcat。Tomcat启动逻辑基于观察者模式设计，所有容器都会继承LifeCycle接口，它管理着容器的整个生命周期，所有容器的修改和状态的改变都会由它去通知已经注册的观察者（Listener）。


