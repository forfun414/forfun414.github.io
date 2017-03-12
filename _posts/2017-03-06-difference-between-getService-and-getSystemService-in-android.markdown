---
layout: post
title: difference between getService and getSystemService in Android
date:   2017-03-06 08:00:00 -0500
categories: jekyll update
---

# Android getService和getSystemService区别

简单的说明下Android framework中使用非常频繁的两个接口的区别
```
ServiceManager.getService("xx")

Context.getSystemService(xxManager.class)
```
前者获取的是binder；

而后者获取到的其实是一个本地的manager对象，但里面封装了binder接口。

# 1 ServiceManager.getService
ServiceManager.getService获取到的就是一个IBinder，

也就是远程服务的代理(binder原理本身是比较复杂的话题，这里不做详述，有兴趣请单独学习专题)，

通过这个IBinder就可以构造出远程服务的接口，从而通过该接口完成到服务端的调用。

# 2 Context.getSystemService
Context.getSystemService获取的其实是一个manager对象。
```
Context.getSystemService
ContextImpl.getSystemService
SystemServiceRegistry.getSystemService
```

最终，SystemServiceRegistry为每个不同的service_name维护了一个CachedServiceFetcher，这个CachedServiceFetcher实现context到Manager对象的映射，也就是说它为不同的context提供不同的Manager。CachedServiceFetcher的实例其实是保存在Context对象中的，ContextImpl.mServiceCache里。

这也就是说每个Context获取到的manager对象，其实就保存在它自己的成员变量中(context中其实保存了所有service_name对应的manager对象)。
而SystemServiceRegistry.CachedServiceFetcher只是用来更方便的从ContextImpl中挑选选取对象，它本身并不保存数据。


综上，getSystemService获取到的是一个本地的manager对象，不过这个manager里面封装了ServiceManager.getService操作，从而简化了对service的操作。

## 所以，本质上getService和getSystemService并无实质差别，只是使用上的封装不一样而已。
