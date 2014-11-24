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
*  ThreadLocal真正的实现在于ThreadLocal的内部有一个ThreadLocalMap的成员变量

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

*  ThreadLocalMap的实现跟HashMap的实现差不多,不同的地方在于Hashcode的计算规则，key为Thread本身

*  ThreadLocal的get()方法，可以看出直接在ThreadMap中取map.getEntry(this);

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

*  ThreadLocal的set()方法，set设置的value是引用还是？？？？测试下

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

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