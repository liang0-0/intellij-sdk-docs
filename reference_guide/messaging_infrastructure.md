---
title: 消息系统基础框架
---


# 目的

本文介绍了消息系统基础架构，主要为开发者以及插件开发者阅读。它旨在回答为什么、何时以及如何使用它。


# 理论基础

那么，什么是以及为什么我要使用消息系统？ 实际上消息系统使用了[观察者模式](https://en.wikipedia.org/wiki/Observer_pattern)
而且还提供了额外的特性， 比如光爆， 集成， 内嵌事件处理(内嵌事件说的是在一个发生的事件中再次触发了事件，即事件中的事件).

# 设计

这里有消息系统API的主要组件

## Topic

这个类属于消息系统的端点（endPoint）.即 客户端允许在particular bus内注册 topic，然后在particular bus中发送消息。

![Topic](img/topic.png)

*  *显示名称：display name*  就是一个能看得懂的名称，主要目的是为打印日志以及监控中使用到；
*  *广播方向：broadcast direction*  在广播那里会详细解释。默认值为 *TO\_CHILDREN*;
*  *监听类：listener class*  是特定topic的业务相关的接口。
在消息系统中，订阅者注册这个监听接口的实现，然后发布者稍后可以检索符合（IS-A）的对象，并调用在那里定义的任何方法。消息传递基础结构负责将消息分发给主题的所有订户，即在已注册的回调上调用具有相同参数的相同方法；

## 消息总线

消息系统中的核心。主要用于以下场景：

![Bus](img/bus.png)

## Connection

Manages all subscriptions for particular client within particular bus.

![Connection](img/connection.png)

*  keeps number of *topic handler* mappings (callbacks to invoke when message for the target topic is received)
*Note*: not more than one handler per-topic within the same connection is allowed;

*  it's possible to specify *default handler* and subscribe to the target topic without explicitly provided callback.
Connection will use that *default handler* when storing *(topic-handler)* mapping;

*  it's possible to explicitly release acquired resources (*disconnect()* method).
Also it can be plugged to standard semi-automatic disposing 
(
[Disposable](upsource:///platform/util/src/com/intellij/openapi/Disposable.java)
);

## Putting altogether

*Defining business interface and topic*

```java
public interface ChangeActionNotifier {

    Topic<ChangeActionNotifier> CHANGE_ACTION_TOPIC = Topic.create("custom name", ChangeActionNotifier.class)

    void beforeAction(Context context);
    void afterAction(Context context);
}
```

*Subscribing*

![Subscribing](img/subscribe.png)

```java
public void init(MessageBus bus) {
    bus.connect().subscribe(ActionTopics.CHANGE_ACTION_TOPIC, new ChangeActionNotifier() {
        @Override
        public void beforeAction(Context context) {
            // Process 'before action' event.
        }
        @Override
        public void afterAction(Context context) {
            // Process 'after action' event.
        }
    });
}
```

*Publishing*

![Publishing](img/publish.png)

```java
public void doChange(Context context) {
    ChangeActionNotifier publisher = myBus.syncPublisher(ActionTopics.CHANGE_ACTION_TOPIC);
    publisher.beforeAction(context);
    try {
        // Do action
        // ...
    } finally {
        publisher.afterAction(context)
    }
}
```

*Existing resources*

*  *MessageBus* instances are available via
[ComponentManager.getMessageBus()](upsource:///platform/core-api/src/com/intellij/openapi/components/ComponentManager.java)<!--#L85-->
(many standard interfaces implement it, e.g.
[Application](upsource:///platform/core-api/src/com/intellij/openapi/application/Application.java),
[Project](upsource:///platform/core-api/src/com/intellij/openapi/project/Project.java);

*  number of public topics are used by the *IntelliJ Platform*, e.g.
[AppTopics](upsource:///platform/platform-api/src/com/intellij/AppTopics.java),
[ProjectTopics](upsource:///platform/projectModel-api/src/com/intellij/ProjectTopics.java)
etc.
So, it's possible to subscribe to them in order to receive information about the processing;

# Broadcasting

Message buses can be organised into hierarchies. Moreover, the *IntelliJ Platform* has them already:

![Standard hierarchy](img/standard-hierarchy.png)

That allows to notify subscribers registered in one message bus on messages sent to another message bus.

*Example:*

![Parent-child broadcast](img/parent-child-broadcast.png)

Here we have a simple hierarchy (*application bus* is a parent of *project bus*) with three subscribers for the same topic.

We get the following if *topic1* defines broadcast direction as *TO\_CHILDREN*:
1.  A message is sent to *topic1* via *application bus*;
2.  *handler1* is notified about the message;
3.  The message is delivered to the subscribers of the same topic within *project bus* (*handler2* and *handler3*);

*Benefits*

We don't need to bother with memory management of subscribers that are bound to child buses but interested in parent bus-level events.

Consider the example above we may want to have project-specific functionality that reacts to the application-level events. 
All we need to do is to subscribe to the target topic within the *project bus*.
No hard reference to the project-level subscriber will be stored at application-level then, 
i.e. we just avoided memory leak on project re-opening.

*Options*

Broadcast configuration is defined per-topic. Following options are available:

*  _TO\_CHILDREN_ (default);

*  _NONE_;

*  _TO\_PARENT_;

# Nested messages

_Nested message_ is a message sent (directly or indirectly) during another message processing.
The IntelliJ Platform's Messaging infrastructure guarantees that all messages sent to particular topic will be delivered at the sending order.

*Example:*

Suppose we have the following configuration:

![Nested messages](img/nested-config.png)

Let's see what happens if someone sends a message to the target topic:

*  _message1_ is sent;

*  _handler1_ receives _message1_ and sends _message2_ to the same topic;

*  _handler2_ receives _message1_;

*  _handler2_ receives _message2_;

*  _handler1_ receives _message2_;

# Tips'n'tricks

## Relief listeners management

Messaging infrastructure is very light-weight, so, it's possible to reuse it at local sub-systems in order to relief
[Observers](https://en.wikipedia.org/wiki/Observer_pattern) construction. Let's see what is necessary to do then:

1. Define business interface to work with;

2. Create shared message bus and topic that uses the interface above (_shared_ here means that either _subject_ or _observers_ know about them);

Let's compare that with a manual implementation:

1. Define listener interface (business interface);

2. Provide reference to the _subject_ to all interested listeners;

3. Add listeners storage and listeners management methods (add/remove) to the _subject_;

4. Manually iterate all listeners and call target callback in all places where new event is fired;

## Avoid shared data modification from subscribers

We had a problem in a situation when two subscribers tried to modify the same document
([IDEA-71701](https://youtrack.jetbrains.com/issue/IDEA-71701)).

The thing is that every document change is performed by the following scenario:

1. _before change_ event is sent to all document listeners and some of them publish new messages during that;

2.  actual change is performed;

3.  _after change_ event is sent to all document listeners;

We had the following then:

1.  _message1_ is sent to the topic with two subscribers;
2.  _message1_ is queued for both subscribers;
3.  _message1_ delivery starts;
4.  _subscriber1_ receives _message1_;
5.  _subscriber1_ issues document modification request at particular range (e.g. _document.delete(startOffset, endOffset)_);
6.  _before change_ notification is sent to the document listeners;
7.  _message2_ is sent by one of the standard document listeners to another topic within the same message bus during _before change_ processing;
8.  the bus tries to deliver all pending messages before queuing _message2_;
9.  _subscriber2_ receives _message1_ and also modifies a document;
10.  the call stack is unwinded and _actual change_ phase of document modification operation requested by _subscriber1_ begins;

**The problem**  is that document range used by _subscriber1_ for initial modification request is invalid if _subscriber2_ has changed document's range before it.


