---
title: Tomcat系统架构
date: 2016-05-22
categories: Java Web
tags: 
- Java
- Tomcat
---

#	Tomcat总体结构
Tomcat有2个核心组件：  
*	Connector  
* 	Container  

其中Connector组件可以被替换，一个Container可以选择对用多个Connector。多个Connector和一个Container形成一个Service。有了Service就可以对外提供服务了。但Service需要一个生存环境，这时就非Server莫属了。所以整个Tomcat的生命周期由Server控制。

Connector主要负责对外交流，Container主要处理Connector接受的请求，主要处理内部事务。其实，Service只是在Connector和Container外面多包一层，把它们组装在一起，对外提供服务。一个Service可以设置多个Connector，但只能有一个Container容器。

##	Service

![Service](http://o7s628cn2.bkt.clouddn.com/image/javaweb/Service.png)

从org.apache.catalina.Service接口中定义方法可以看出，它主要是为了关联Connector和Container，同时会初始化其他组件。所有组件的生命周期在一个Lifecycle的接口中控制。

![StandardService](http://o7s628cn2.bkt.clouddn.com/image/javaweb/StandardService.png)

从StandardService类结构方法可以看出，除了Service接口方法的实现以及控制组件生命周期的Lifecycle接口的实现，还有几个方法用于实现时间监听方法。不仅是这个Service组件，在Tomcat中其他组件也同样有这个几个方法，这也是一个典型设计模式。

下面看一下StandardService中几个主要方法，setContainer方法如下：

```	java
public void setContainer(Container container) {

        Container oldContainer = this.container;
        if ((oldContainer != null) && (oldContainer instanceof Engine))
            ((Engine) oldContainer).setService(null);
        this.container = container;
        if ((this.container != null) && (this.container instanceof Engine))
            ((Engine) this.container).setService(this);
        if (getState().isAvailable() && (this.container != null)) {
            try {
                this.container.start();
            } catch (LifecycleException e) {
                // Ignore
            }
        }
        if (getState().isAvailable() && (oldContainer != null)) {
            try {
                oldContainer.stop();
            } catch (LifecycleException e) {
                // Ignore
            }
        }

        // Report this property change to interested listeners
        support.firePropertyChange("container", oldContainer, this.container);

    }
```

这段代码很简单，首先判断当前Service有没有已经关联的Container，如果已经关联，去除这个关联关系----((Engine) oldContainer).setService(null)。如果这个oldContainer已经启动了，结束它的生命周期，然后再替换新的。

AddConnector方法如下：

```	java
public void addConnector(Connector connector) {

        synchronized (connectors) {
            connector.setService(this);
            Connector results[] = new Connector[connectors.length + 1];
            System.arraycopy(connectors, 0, results, 0, connectors.length);
            results[connectors.length] = connector;
            connectors = results;

            if (getState().isAvailable()) {
                try {
                    connector.start();
                } catch (LifecycleException e) {
                    log.error(sm.getString(
                            "standardService.connector.startFailed",
                            connector), e);
                }
            }

            // Report this property change to interested listeners
            support.firePropertyChange("connector", null, connector);
        }

    }
```

首先设置关联关系，然后初始化，开始新的生命周期。Connector用的是数组，而不是List集合，从性能角度考虑可以理解。有趣的是这里用了数组但并没有一开始就分配一个固定大小的数组。

##	Server
Server要完成的任务很简单，提供一个接口让其他程序能够访问到这个Service集合，同时要维护它所包含的所有Service生命周期，包括如何初始化、如何结束服务、如何找到别人要访问的Service。

Server的类结构图如下：
![Server](http://o7s628cn2.bkt.clouddn.com/image/javaweb/Server.png)

它的标准实现类StandardServer实现了上面这些方法，同时也实现了Lifecycle、MbeanRegistration两个接口的所有方法。下面看一下StandardServer一个重要方法addService的实现：

```	java
public void addService(Service service) {

    service.setServer(this);

    synchronized (services) {
        Service results[] = new Service[services.length + 1];
        System.arraycopy(services, 0, results, 0, services.length);
        results[services.length] = service;
        services = results;

        if (getState().isAvailable()) {
            try {
                service.start();
            } catch (LifecycleException e) {
                // Ignore
            }
        }

        // Report this property change to interested listeners
        support.firePropertyChange("service", null, service);
    }

}
```

从第一行就知道了Service和Server是相互关联的，Server也是和Service管理Connector一样管理它，也是将Service放在一个数组中。

##	Lifecycle
在Tomcat中组件的生命周期通过Lifecycle接口来控制，组件只要集成这个接口并实现其中方法就可以统一被拥有它的组件控制了。最高级的组件就是Server，而控制Server的是Startup，也就是启动和关闭Tomcat。Lifecycle接口方法的实现都在其他组件中，组件的生命周期由包含它的父组件控制，所以它的Start方法自然就是调用它下面组件的Start方法，Stop方法也是一样。

```	java
protected void startInternal() throws LifecycleException {

    fireLifecycleEvent(CONFIGURE_START_EVENT, null);
    setState(LifecycleState.STARTING);

    globalNamingResources.start();
    
    // Start our defined Services
    synchronized (services) {
        for (int i = 0; i < services.length; i++) {
            services[i].start();
        }
    }
}
```
```	java
protected void stopInternal() throws LifecycleException {

    setState(LifecycleState.STOPPING);
    fireLifecycleEvent(CONFIGURE_STOP_EVENT, null);
    
    // Stop our defined Services
    for (int i = 0; i < services.length; i++) {
        services[i].stop();
    }

    globalNamingResources.stop();
    
    stopAwait();
}
```
监听的代码会包围Service组件的启动过程，但所有Service必须要实现Lifecycle接口，这样做会更加灵活。

##	Connector组件
它的主要任务数负责接收浏览器发过来的TCP连接请求，创建一个Request和Response对象分别用于和请求端交换数据。然后会产生一个线程来处理这个请求并把产生的Request和Response对象传给处理这个请求的线程，处理这个请求的线程就是Container要做的事了。

在Tomcat7中，Connector有3种选择：	
*	BIO--Http11Protocol
* 	NIO--Http11NioProtocol
*  apr--Http11AprProtocol

```	java
public Connector(String protocol) {
    setProtocol(protocol);
    // Instantiate protocol handler
    try {
        Class<?> clazz = Class.forName(protocolHandlerClassName);
        this.protocolHandler = (ProtocolHandler) clazz.newInstance();
    } catch (Exception e) {
        log.error(sm.getString(
                "coyoteConnector.protocolHandlerInstantiationFailed"), e);
    }
}
```
通过上述代码可以看出，Connector会根据我们在server.xml中配置的Connector类型来初始化。

这里以常用的NIO来说明，找到Http11NioProtocol基类的start方法：

```	java
public void start() throws Exception {
    if (getLog().isInfoEnabled())
        getLog().info(sm.getString("abstractProtocolHandler.start",
                getNameInternal()));
    try {
        endpoint.start();
    } catch (Exception ex) {
        getLog().error(sm.getString("abstractProtocolHandler.startError",
                getNameInternal()), ex);
        throw ex;
    }

    // Start async timeout thread
    asyncTimeout = new AsyncTimeout();
    Thread timeoutThread = new Thread(asyncTimeout, getNameInternal() + "-AsyncTimeout");
    timeoutThread.setPriority(endpoint.getThreadPriority());
    timeoutThread.setDaemon(true);
    timeoutThread.start();
}
```
start方法中调用endpoint.start()，找到对应方法，在NioEndpoint类中：

```	java
public void startInternal() throws Exception {

        if (!running) {
            running = true;
            paused = false;

            processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                    socketProperties.getProcessorCache());
            eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                            socketProperties.getEventCache());
            nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                    socketProperties.getBufferPool());

            // Create worker collection
            if ( getExecutor() == null ) {
                createExecutor();
            }

            initializeConnectionLatch();

            // Start poller threads
            pollers = new Poller[getPollerThreadCount()];
            for (int i=0; i<pollers.length; i++) {
                pollers[i] = new Poller();
                Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
                pollerThread.setPriority(threadPriority);
                pollerThread.setDaemon(true);
                pollerThread.start();
            }

            startAcceptorThreads();
        }
    }
```

当执行到startAcceptorThreads()时，就会进入等待请求的状态，知道一个新的请求到来才会激活它继续执行。
