---
layout: post
title: "ThreadPool对ThreadLocal的影响"
tagline: ""
description: ""
tags: [ThreadPool, ThreadLocal]
---
{% include JB/setup %}
##ThreadLocal
ThreadLocal很容易望文生义"本地线程"。其实ThreadLocal并不是一个本地线程，而是Thread的一个局部变量。ThreadLocal为每一个使用改变量的线程提供一
个独立的副本，每一个线程都可以独立改变自己的副本，而不会影响其他线程所对应的副本。

##ThreadLocal实现原理

    /* ThreadLocal values pertaining to this thread. This map is maintained
         * by the ThreadLocal class. */
        ThreadLocal.ThreadLocalMap threadLocals = null;

    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }


*  ThreadLocal真正的实现在于ThreadLocal的内部有一个ThreadLocalMap的成员变量

*  ThreadLocalMap的实现跟HashMap的实现差不多,不同的地方在于Hashcode的计算规则，key为Thread本身

*  ThreadLocal的get()方法，可以看出直接在ThreadMap中取map.getEntry(this);


*  ThreadLocal的set()方法，set设置的value是引用,在单线程中，Map中put的对象是引用关系，取出来之后再赋值，再取出来对象也会发生变化
例如下面代码：

        Map<String,StringBuffer> map = new HashMap<String,StringBuffer>();
        StringBuffer sb = new StringBuffer("aa");

        map.put("1",sb);
        map.put("2",sb);
        map.get("1").append("3");
        System.out.println(map.get("2")); //输出aa3

在看ThreadLocal的源码中，看到value的赋值是引用赋值，并不是拷贝操作。那么很容易产生疑问？：
为什么在调用ThreadLocal变量的set方法，不会对其他线程造成影响。
还是太天真，其实很简单，再多线程环境下，每个线程都有一个栈空间，set(value)的时候，每个线程的value，本来就是不同的。



##ThreadLocal的应用举例
SimpleDataFormat是一个线程不安全的类，多线程的情况下一般使用的方式是：

*  每次调用都new出来一个SimpleDataFormat对象
*  使用加锁的方式使用SimpleDataFormat

###骚年还是太单纯，ThreadLocal在这里可以用一下

    public class Foo{
        private static final ThreadLocal<SimpleDateFormat> formatter = new ThreadLocal<SimpleDateFormat>(){
            @Override
            protected SimpleDateFormat initialValue()
            {
                return new SimpleDateFormat("yyyyMMdd HHmm");
            }
        };

        public String formatIt(Date date)
        {
            return formatter.get().format(date);
        }
    }

网上看到的，不知优雅不优雅，（恩，“优雅”）

##ThreadLocal 与在线程池中出现的坑
在代码过程在出现一种错误，一个Service单例中一个ThreadLocal,日志中发现，在没有给这个变量赋值的情况下，取出的值竟然不是空！

###原因是：tomcat线程池并不会对使用完的线程回收清理，如果并发量大，取出了已经被使用的线程后，取出的ThreadLocal的变量可能是旧值
解决办法：
在使用ThreadLocal变量的时候，在使用完毕之后，需要显示的主动的释放资源，这是一个很好的习惯

    threadLocal.remove();//（听说在某个版本后自动调用这个了，不确定。待定下！）

这样做优点：

*  避免因为线程池的重用而导致的threadlocal变量中get的数据是老线程的数据，而导致不正确

*  在一些Web容器中，比如tomcat、jboss的中，如果不主动释放资源，那么可能会导致out of memory的异常出现。

那么为什么会出现OOM呢？

因为在tomcat或者其他web容器中，通常有线程池。线程池中的线程是复用的，Thread不会被销毁，而ThreadLocal变量是保存在Thread中的ThreadLocalMap中的，
所以如果不主动清理这些变量，那么这些ThreadLocal变量一直会存在线程池中，不被清理。就算tomcat中的应用销毁了，这些threadlocal变量还在。这就造成了memory leak，
但还不至于出现OOM。那么什么情况下出现OOM呢？

当应用不停的reload的情况下会出现OOM。为什么呢？当tomcat中的应用卸载的时候，classloader会被回收，那么GC会清理在perm heap中不被使用的class（注：PermGen中可以被回收的条件是：
classloader可以被回收，其下的所有加载过的没有对应实例的类信息（保存在permgen）可以被回收）。发现它无法清理ThreadLocal变量中引用的那个class，因为这个class正在被线程池中的线程使用。
如果那个class又引用了其他class，那么所有被引用的class将都不能被清理。这样就导致这些类信息在PermGen中发生memory leak了。
然后当应用重新load之后，相应的类信息又被新的classloader重新在permGen中加载一遍。在最坏的情况下，应用不停的reload，那么permGen终将被class信息撑爆，造成OOM。


