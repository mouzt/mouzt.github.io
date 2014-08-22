---
layout: post
title: "EventBus浅浅析"
tagline: "EventBus Note"
description: ""
tags: [EventBus, guava ]
---
{% include JB/setup %}
##EventBus

Guava的事件处理机制，是设计模式中的观察者模式（生产/消费者编程模型）
##如何使用
*  定义一个observer，并加入@Subscribe作为消息回调函数
*  将observer注册到EventBus；EventBus.register(this);
*  消息投递: eventBus.post(logTo);

##源码浅浅析
*  ##定义Observer


class EventBusChangeRecorder {
    // Subscribe annotation，并且只有一个 ChangeEvent 方法参数
    @Subscribe
    public void recordCustomerChange(ChangeEvent e) {
        recordChange(e.getChange());
    }
}

* ##注册到EventBus

通过一个mutimap存储订阅方法，其中key为参数类型.在一个observer类里面，可以定义多个@Subscribe，根据method.getParameterTypes()[0]来缓存参数的类型


Multimap<Class<?>, EventSubscriber> methodsInListener = HashMultimap.create();
//SubscriberFindingStrateg.findAllSubscribers(Object)
//根据注解得到所有的订阅方法
public void register(Object object) {
    Multimap<Class<?>, EventSubscriber> methodsInListener =
        finder.findAllSubscribers(object);
    subscribersByTypeLock.writeLock().lock();
    try {
      subscribersByType.putAll(methodsInListener);
    } finally {
      subscribersByTypeLock.writeLock().unlock();
    }
  }

**@Subscribe所annotate的method的参数，不能支持泛型。因为在运行的时候，因为Type Erasure导致拿不到"真正"的parameterType**

* ##post到EventBus

EventBus做了缓存，所有的EventBus都注册到一个Set里面
//得到所有分发的类型，获取到所有的订阅者，然后插入到消息分发队列中

  public void post(Object event) {
    Set<Class<?>> dispatchTypes = flattenHierarchy(event.getClass());

    boolean dispatched = false;
    for (Class<?> eventType : dispatchTypes) {
      subscribersByTypeLock.readLock().lock();
      try {
        Set<EventSubscriber> wrappers = subscribersByType.get(eventType);

        if (!wrappers.isEmpty()) {
          dispatched = true;
          for (EventSubscriber wrapper : wrappers) {
            enqueueEvent(event, wrapper);
          }
        }
      } finally {
        subscribersByTypeLock.readLock().unlock();
      }
    }

    if (!dispatched && !(event instanceof DeadEvent)) {
      post(new DeadEvent(this, event));
    }

    dispatchQueuedEvents();
  }

需要分发的消息会被提交到
private final ConcurrentLinkedQueue<EventWithSubscriber> eventsToDispatch的队列中

之后进行消息分发

protected void dispatchQueuedEvents() {
    while (true) {
        EventWithSubscriber eventWithSubscriber = eventsToDispatch.poll();
        if (eventWithSubscriber == null) {
            break;
        }
    dispatch(eventWithSubscriber.event, eventWithSubscriber.subscriber);
    }
}
###那么为什么要选用ConcurrentLinkedQueue而不是LinkedBlockingQueue呢？

简单来说，ConcurrentLinkedQueue是无锁的，没有synchronized，也没有Lock.lock，依靠CAS保证并发，同时，也不提供阻塞方法put()和take()，速度上面肯定无锁的会更快一些

> ##AsyncEventBus

BlockingQueue implementations are designed to be used primarily for producer-consumer queues