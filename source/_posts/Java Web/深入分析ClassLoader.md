---
title: 深入分析ClassLoader
date: 2016-05-28
categories: Java Web
tags: 
- Java
- JVM
---

ClassLoader除了能将Class加载到JVM中之外，还能审查每个类该由谁加载。它是一种父优先的等级加载机制。

##	ClassLoader等级加载机制	

1.	Bootstrap ClassLoader，主要加载JVM自身工作需要的类，完全由JVM自己控制，外部无法访问，这个ClassLoader不遵守等级加载规则。
2. 	ExtClassLoader，不是JVM亲自实现。服务目标在System.getProperties("java.ext.dirs")目录下。
3. 	AppClassLoader，父类是ExtClassLoader，所有在System.getProperties("java.class.path")目录下的类都可以被这个类加载器加载，这就是我们常用到的classpath。

如果我们要实现自己的类加载器，不管是直接实现抽象类ClassLoader，还是继承URLClassLoader或其他子类，它的父加载器都是AppClassLoader，不管调用哪个父类构造器，创建对象都必须最终调用getSystemCLassLoader()作为父加载器，而getSystemClassLoader()方法获取到的正是AppClassLoader。

Bootstrap ClassLoader并不属于JVM的类层次结构，因为BootstrapClassLoader并没有遵守ClassLoader的加载规则。Bootstrap ClassLoader并没有子类，ExtClassLoader父类也不是Bootstrap ClassLoader，ExtClassLoader并没有父类，应用中能提取到的顶层父类是ExtClassLoader。

ExtClassLoader和AppClassLoader都位于sun.misc.Launcher类中，它们是Launcher类的内部类，都继承了URLClassLoader类，URLClassLoader实现了抽象类ClassLoader。

##	如何加载class文件	
抽象类ClassLoader中并没有定义如何加载，需要子类实现findClass()方法。看一下URLClassLoader实现findClass()过程：	
1.	通过URLClassPath类取得加载的class文件字节流，找到后读取它的byte字节流。	
2. 	调用defineClass()创建类对象。

创建URLClassLoader对象会根据传过来的URL数组中路径来判断是文件还是jar包，根据路径的不同分别创建FileLoader和JarLoader，或者使用默认的加载器。

###	验证与解析
1.	字节码验证，确保格式正确，行为正确。
2. 	类准备，准备字段、方法和实现接口所必须的数据结构。
3. 	解析，类装入器装入类引用的其他所有类。	
4.	初始化对象，该阶段末尾静态字段被初始化默认值。

##	实现自己的ClassLoader	
应用场景：	
*	在自定义路径下查找自定义的class类文件，也许我们需要的class文件并不总是在classPath下面。	
* 	对我们自己要加载的类做特殊处理，如保证通过网络传输的安全性，可以将类经过加密后传输，在加载到JVM之前需要对类的字节码再解密。		
*  	可以定义类的实现机制，例如实现热部署：检查已经加载的class文件是否被修改，如修改可重新加载。	

JVM表示一个类是否是同一个类有两个条件：	
1.	检查完整类名是否一致。		
2. 	检查加载这个类的ClassLoader是否是同一个，这里指的是ClassLoader实例是否是同一个。