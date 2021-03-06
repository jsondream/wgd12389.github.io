---
layout: post
title: 分布锁——redis实现
category: 技术
tags: [tech]
keywords: lock, redisson
description: 分布锁——redis实现
---



## 分布式锁的场景   

首先在读文章之前，我们要考虑一个问题，为什么要用分布式锁，也就是什么场景下要用分布式锁？   

> 假如我们有一个抢购业务，之前是单机的时候我们可以用程序锁，扩展到了多个服务节点的时候，那我无法再继续使用lock sync等程序的锁来控制并发中可能会造成的超卖。   

这时候我们就该引入一个分布式锁来解决这个问题，当然上面的例子有更好的解决办法，这里仅仅提供一个分布式锁的场景引入。   

## 设计一个分布式锁的要素    

OK，我们知道什么情况下用分布式锁了之后，我们要考虑下，如果让我们设计一个分布式锁，要考虑哪些问题？    

> 第一点，既然是锁，那么我要确保这个锁在整个集群中唯一性   
> 第二点，我要确保我某个获取到锁的节点挂掉之后不会因为无法释放而产生死锁的问题   
> 第三点，我要确保我的锁不会被其他的节点误操作而错误的解锁   
> 第四点，我们要考虑我们的锁其他的竞争线程，如何在持有锁的节点释放之后快速的受到通知重新竞争锁   
> 第五点，就是锁的性能效率   
> 第六点，就是锁的可重入性（这里提一下，对于大多数程序和业务来讲是没必要实现这个功能的）   

ok，上面就是我们做一个分布式锁应该注意的地方，当然，这里说的情况并不是很全面，但是基本上已经足够大多数的业务使用了，那么我们带着上面的这些注意的点，一起去看一下怎么实现一个分布式锁。   

## 构建分布式锁    

### 锁的唯一性问题   

大家应该都知道redis里有个过期时间的概念，也就是expire这个api可以设置一个key的过期时间，那么利用这个功能我们可以设置一个最大值，避免死锁的问题。   

那么说道这里，大家可能跟我之前一样，会考虑到一个问题？    
**这个过期时间设置为多少好呢，如果设置太小了，会造成业务没操作完，锁就提前被其他线程获取了，如果设置太大了，又可能比死锁没好多少。**     

#### redisson是怎么解决这个问题的？       

redisson默认是设置一个key的过期时间为30秒，那么大家可能想，这也没区别啊！     
如果看过redisson源码的应该注意到他用了netty，那么他用netty干嘛了？他用netty做了这样的一个事，他给每个上锁的操作都加了一个事件。    

**什么样的事件？**     
如果我一个上锁操作，上锁失败了，就订阅锁，直到收到通知，否则就暂时等待，这里他是利用java用的信号量来实现的，如果有兴趣的可以看一下他的具体代码。      
那么如果上锁成功了呢？他会开启一个异步线程，等待通知，这个通知可以是这样的：***如果我收到的通知是，我工作完了，要释放锁了。那么这时候他就把这个异步线程从工作者队列中干掉。***    
那么，如果我没有收到通知呢？这一步其实就是redisson的关键实现    

![redisson锁的代码](http://7xpz5v.com1.z0.glb.clouddn.com/redis-lock-redisson-step-1)    

如果我没收到通知，我每隔离10s会调用一次这个事件，判断一下过期时间，然后给这个持有锁的线程的key，也就是当前锁，重新设置上为30秒的过期时间，也就说，即使我这台机器挂掉了，那我这个机器持有的锁最多会保留30s的“死锁”时间。    

#### 如果我有一堆远程调用，30s根本不够用呢？   
没关系，每隔10s你的过期时间都会更新为30s。也就是一直到你释放锁。当然，如果你害怕你的业务会发送阻塞而造成了锁的一直持有的"假死锁"情况，那怎么办？    
redisson提供了lockInterruptibly(long leaseTime, TimeUnit unit)的时间限制哈。也就是你在多少秒之内如果没完成任务也会自动释放这个锁。     

### 锁的标志    

**我怎么要确保我的锁不会被其他的节点误操作而错误的解锁呢?**       

这个其实很好解决，一般对于一个锁来讲，都是需要一个onwer的标示，对于大多数的做法：都是使用uuid+ThreadId，然后操作线程保留这个onwer标示，在set的时候吧这个owner的标示放到value中，解锁的时候判断这个owner标示。     

**note:**一般来讲这个owner标示还起着做重入的时候的作用.     

### 解锁后的快速通知     

这里其实是有两种做法：

* 第一种，类似本地锁的不断重试(自旋)。 
* 第二种方式，也就是redlock的实现RedisSon的做法，pub/sub的方式    

#### 自旋如何实现      

> 我原来做这一块的时候利用locksupport的park来做了短暂时间的暂停，再暂停之后不断的重试获取锁。    

但是这样就会有这样的几个问题：    
1. 锁的通知被释放的时候我无法及时的收到通知，并且这个能获取锁的机会有可能就看运气了，也就是说谁暂停完之后重试的时间正好是我释放的时间，也就是无法实现公平锁（按申请锁的顺序来获取锁）   
2. redis毕竟是网络的，无论是网络抖动的影响还是自身这种不断发请求来讲，都是很大的开销，性能上烂到爆    

但是上面的锁翩翩适用一种常见，业务操作比较简单短暂，不会出太多问题，耗时比较短，需要简单的锁模型     

#### pub/sub如何实现     

相信了解过redis的都知道它有个发布订阅的功能      
RedisSon是这么实现快速通知的：    

> 获取锁的线程会去redis中发布一个key，然后所有没有取到锁的就去订阅相应的channel，来接收锁释放的通知，获取锁的释放了之后就会去这里发布释放的通知。收到消息的就会继续重试获取锁的过程      

### 性能问题     

首先，大家都知道，一个分布式锁可以基于zk和redis来实现。   
但是rediss做分布式锁的效率要比zk高上很多很多倍，因为zk是基于文件系统的实现，而redis是基于内存的操作实现。   
而且zk做分布式锁的时候还会有可能因为网络抖动的问题发生锁被误释放的问题（这里我们暂时不讨论）   

###  可冲入特性      

我这里简单的说一下redisson是怎么实现的：   

redisson是吧获取锁的行为变成了一次hashset的操作   

![redisson锁的代码](http://7xpz5v.com1.z0.glb.clouddn.com/redis-lock-redisson-step-2)     

**这里就是核心的实现：**      
  
> 上面lua代码中第一个if就是先尝试获取锁，如果获取成功就返回，如果不成功就判断要获取锁的线程，和持有锁的线程是否是同一个线程，如果是，那就在value上加个1，代表重入了一次，最后那个return的pttl其实是一个else逻辑，也就是说我既没有获取锁，也不是持有锁的那个线程，也就意味我获取锁失败，那我就返回一个过期时间的值     

## Redisson的流程简述      

redisson的整个过程简介：利用lua在redis中的原子性，获取锁保证唯一性，在value中加上标示防止误解锁，不断的叠加expire来保证持有锁的时候不会被误拿到，利用redis的pub/sub来即使的通知锁的释放，利用Semaphore来实现没有获取锁的线程的等待。    

## 要思考的问题     

自旋锁 然后 自旋锁会导致饥饿 就开始使用阻塞，然后 阻塞会导致CPU等资源空置 就开始使用异步解耦(最常见的就是做完了，通知的方式)，其实整个过程跟IO的几种模式很像 从BIO到NIO到AIO的整个过程，然后大家看到发布订阅这种通知的模式了。。     
但如果发布订阅模式突然挂了,你的线程可能永远不会醒来了？这里我还没有完全的关注到这个点，日后有机会吧这个点看一下然后补充上来。     


