---
layout: post
title: "ZooKeeper(一)有什么用"
tagline: "zookeeper,分布式"
description: ""
tags: [zookeeper, 分布式]
---
{% include JB/setup %}

##一、什么是ClassLoader?

         大家都知道，当我们写好一个Java程序之后，不是管是CS还是BS应用，都是由若干个.class文件组织而成的一个完整的Java应用程序，当程序在运行时，即会调用该程序的一个入口函数来调用系统的相关功能，而这些功能都被封装在不同的class文件当中，所以经常要从这个class文件中要调用另外一个class文件中的方法，如果另外一个文件不存在的，则会引发系统异常。而程序在启动的时候，并不会一次性加载程序所要用的所有class文件，而是根据程序的需要，通过Java的类加载机制（ClassLoader）来动态加载某个class文件到内存当中的，从而只有class文件被载入到了内存之后，才能被其它class所引用。所以ClassLoader就是用来动态加载class文件到内存当中用的。



##二、Java默认提供的三个ClassLoader

BootStrap ClassLoader：称为启动类加载器，是Java类加载层次中最顶层的类加载器，负责加载JDK中的核心类库，如：rt.jar、resources.jar、charsets.jar等，可通过如下程序获得该类加载器从哪些地方加载了相关的jar或class文件：


    URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
    for (int i = 0; i < urls.length; i++) {
        System.out.println(urls[i].toExternalForm());
    }

以下内容是上述程序从本机JDK环境所获得的结果：

    file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/resources.jar
    file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/rt.jar
    file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/sunrsasign.jar
    file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/jsse.jar
    file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/jce.jar
    file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/charsets.jar
    file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/classes/

其实上述结果也是通过查找sun.boot.class.path这个系统属性所得知的。


    System.out.println(System.getProperty("sun.boot.class.path"));
打印结果：C:\Program Files\Java\jdk1.6.0_22\jre\lib\resources.jar;C:\Program Files\Java\jdk1.6.0_22\jre\lib\rt.jar;C:\Program Files\Java\jdk1.6.0_22\jre\lib\sunrsasign.jar;C:\Program Files\Java\jdk1.6.0_22\jre\lib\jsse.jar;C:\Program Files\Java\jdk1.6.0_22\jre\lib\jce.jar;C:\Program Files\Java\jdk1.6.0_22\jre\lib\charsets.jar;C:\Program Files\Java\jdk1.6.0_22\jre\classes

###Extension ClassLoader：称为扩展类加载器，负责加载Java的扩展类库，默认加载JAVA_HOME/jre/lib/ext/目下的所有jar。

###App ClassLoader：称为系统类加载器，负责加载应用程序classpath目录下的所有jar和class文件。

        注意： 除了Java默认提供的三个ClassLoader之外，用户还可以根据需要定义自已的ClassLoader，而这些自定义的ClassLoader都必须继承自java.lang.ClassLoader类，也包括Java提供的另外二个ClassLoader（Extension ClassLoader和App ClassLoader）在内，
        但是Bootstrap ClassLoader不继承自ClassLoader，因为它不是一个普通的Java类，底层由C++编写，已嵌入到了JVM内核当中，当JVM启动后，Bootstrap ClassLoader也随着启动，负责加载完核心类库后，并构造Extension ClassLoader和App ClassLoader类加载器。



##三、ClassLoader加载类的原理

 ###1、原理介绍

       ClassLoader使用的是双亲委托模型来搜索类的，每个ClassLoader实例都有一个父类加载器的引用（不是继承的关系，是一个包含的关系），虚拟机内置的类加载器（Bootstrap ClassLoader）本身没有父类加载器，但可以用作其它ClassLoader实例的的父类加载器。当一个ClassLoader实例需要加载某个类时，它会试图亲自搜索某个类之前，先把这个任务委托给它的父类加载器，这个过程是由上至下依次检查的，首先由最顶层的类加载器Bootstrap ClassLoader试图加载，如果没加载到，则把任务转交给Extension ClassLoader试图加载，如果也没加载到，则转交给App ClassLoader 进行加载，如果它也没有加载得到的话，则返回给委托的发起者，由它到指定的文件系统或网络等URL中加载该类。如果它们都没有加载到这个类时，则抛出ClassNotFoundException异常。否则将这个找到的类生成一个类的定义，并将它加载到内存当中，最后返回这个类在内存中的Class实例对象。



###2、为什么要使用双亲委托这种模型呢？

       因为这样可以避免重复加载，当父亲已经加载了该类的时候，就没有必要子ClassLoader再加载一次。考虑到安全因素，我们试想一下，如果不使用这种委托模式，那我们就可以随时使用自定义的String来动态替代java核心api中定义的类型，这样会存在非常大的安全隐患，而双亲委托的方式，就可以避免这种情况，因为String已经在启动时就被引导类加载器（Bootstrcp ClassLoader）加载，所以用户自定义的ClassLoader永远也无法加载一个自己写的String，除非你改变JDK中ClassLoader搜索类的默认算法。