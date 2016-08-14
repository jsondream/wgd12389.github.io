---
layout: post
title: delayQueue原理理解之源码解析
category: 技术
tags: [tech]
keywords: delayQueue,delay,currentPackage
description: delayQueue原理理解之源码解析
---


### 内部结构  

- 可重入锁  
- 用于根据delay时间排序的优先级队列  
- 用于优化阻塞通知的线程元素leader  
- 用于实现阻塞和通知的Condition对象  

### delayed和PriorityQueue  

在理解delayQueue原理之前我们需要先了解两个东西,delayed和PriorityQueue.  

- delayed是一个具有过期时间的元素  
- PriorityQueue是一个根据队列里元素某些属性排列先后的顺序队列  

delayQueue其实就是在每次往优先级队列中添加元素,然后以元素的delay/过期值作为排序的因素,以此来达到先过期的元素会拍在队首,每次从队列里取出来都是最先要过期的元素  

### offer方法  

1. 执行加锁操作  
2. 吧元素添加到优先级队列中  
3. 查看元素是否为队首  
4. 如果是队首的话，设置leader为空，唤醒所有等待的队列   
5. 释放锁  

代码如下：  

```
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
```  

### take方法  

1. 执行加锁操作  
2. 取出优先级队列元素q的队首  
3. 如果元素q的队首/队列为空,阻塞请求  
4. 如果元素q的队首(first)不为空,获得这个元素的delay时间值  
5. 如果first的延迟delay时间值为0的话,说明该元素已经到了可以使用的时间,调用poll方法弹出该元素,跳出方法   
6. 如果first的延迟delay时间值不为0的话,释放元素first的引用,避免内存泄露    
7. 判断leader元素是否为空,不为空的话阻塞当前线程  
8. 如果leader元素为空的话,把当前线程赋值给leader元素,然后阻塞delay的时间,即等待队首到达可以出队的时间,在finally块中释放leader元素的引用  
9. 循环执行从1~8的步骤  
10. 如果leader为空并且优先级队列不为空的情况下(判断还有没有其他后续节点),调用signal通知其他的线程  
11. 执行解锁操作  

```
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null)
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }  
```  

### get点  

整个代码的过程中并没有使用上太难理解的地方,但是有几个比较难以理解他为什么这么做的地方  

#### leader元素的使用  

大家可能看到在我们的DelayQueue中有一个Thread类型的元素leader,那么他是做什么的呢,有什么用呢？  

让我们先看一下元素注解上的doc描述:  


> Thread designated to wait for the element at the head of the queue.   
> This variant of the [Leader-Follower pattern](http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to minimize unnecessary timed waiting.  
> when a thread becomes the leader, it waits only for the next delay to elapse, but other threads await indefinitely.
> The leader thread must signal some other thread before returning from take() or poll(...), unless some other thread becomes leader in the interim.   
> Whenever the head of the queue is replaced with an element with an earlier expiration time, the leader field is invalidated by being reset to null, and some waiting thread, but not necessarily the current leader, is signalled.  
> So waiting threads must be prepared to acquire and lose leadership while waiting.  

上面主要的意思就是说用leader来减少不必要的等待时间,那么这里我们的DelayQueue是怎么利用leader来做到这一点的呢:    

这里我们想象着我们有个多个消费者线程用take方法去取,内部先加锁,然后每个线程都去peek第一个节点.  
如果leader不为空说明已经有线程在取了,设置当前线程等待    

```
if (leader != null)
   available.await();
```

如果为空说明没有其他线程去取这个节点,设置leader并等待delay延时到期,直到poll后结束循环    

```
     else {
         Thread thisThread = Thread.currentThread();
         leader = thisThread;
         try {
              available.awaitNanos(delay);
         } finally {
              if (leader == thisThread)
                  leader = null;
         }
     }
```

#### take方法中为什么释放first元素  

```
first = null; // don't retain ref while waiting
```

我们可以看到doug lea后面写的注释,那么这段代码有什么用呢？  

想想假设现在延迟队列里面有三个对象。  
- 线程A进来获取first,然后进入 else 的else ,设置了leader为当前线程A  - 线程B进来获取first,进入else的阻塞操作,然后无限期等待  
- 这时在JDK 1.7下面他是持有first引用的  
- 如果线程A阻塞完毕,获取对象成功,出队,这个对象理应被GC回收,但是他还被线程B持有着,GC链可达,所以不能回收这个first.  - 假设还有线程C 、D、E.. 持有对象1引用,那么无限期的不能回收该对象1引用了,那么就会造成内存泄露.     


