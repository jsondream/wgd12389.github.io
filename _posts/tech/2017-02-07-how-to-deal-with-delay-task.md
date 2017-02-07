---
layout: post
title: 细说延迟任务的处理
category: 技术
tags: [tech]
keywords: delayQueue,delayTask,rabbitMq,wheelTimer,quartz
description: 细说延迟任务的几种处理方式 
---


## 源起   

大家可能都遇到过类似的需求:   

* 生成订单60秒后,给用户发短信   
* 下单之后15分钟,如果用户不付款就关闭订单   

## 解决方式   

是的没错,我们用一种术语来描述上面的任务,***延时任务***.   

那么针对于类似这样的任务,一般我们都是怎么处理的呢?   

对于这种延时任务,我们一般有以下的4中解决方式:   

* 利用quartz等定时任务   
* delayQueue    
* wheelTimer    
* rabbitMq的延迟队列     

下面就让我们一起看一下这四种方式各自的优劣。   

### 利用quartz等定时任务      

相信目前还有很多的公司依然沿用着这种做法,那么利用quartz怎么解决这个延时任务的问题呢？   

具体的方式就是这样的,比如我们有个下单15分钟后用户不付款就关闭订单的任务.我的订单是存储在mysql的一个表里,表里会有各种状态和创建时间.   
利用quartz来设定一个定时任务,我们暂时设置为每5分钟扫描一次.扫描的条件为未付款并且当前时间大于创建时间超过15分钟.然后我们再去逐一的操作每一条数据.    

***优点：***  简单易用,可以利用quartz的分布式特性轻易的进行横向扩展。    

***缺点：***  需要扫表会增加程序负荷、任务执行不够准时。    

### 利用jdk自带的delayQueue      

上面我们已经说过了用quartz解决这个办法,现在我们这里引入了新的东东,就是jdk自带的*delayQueue*.   

那么究竟什么是*delayQueue*呢？   

> DelayQueue是java.util.concurrent中提供的一个很有意思的类。很巧妙，非常棒！但是java doc和Java SE 5.0的source中都没有提供Sample。在ScheduledThreadPoolExecutor源码时，发现DelayQueue的妙用。可以这么说，DelayQueue是一个使用优先队列（PriorityQueue）实现的BlockingQueue，优先队列的比较基准值是时间（关于DelayQueue的源码解析可以看我之前的文章[delayQueue原理理解之源码解析](http://www.jsondream.com/2016/08/14/delayQueue-source-read.html)）    

#### 怎么使用delayQueue呢？   

DelayQueue主要用于放置实现了Delay接口的对象，其中的对象只能在其时刻到期时才能从队列中取走。这种队列是有序的，即队头的延迟到期时间最短。如果没有任何延迟到期，那么久不会有任何头元素，并且poll()将返回null（正因为这样，你不能将null放置到这种队列中）   

也就是说我们只需要把我们需要延迟触发的任务构建完毕放到*delayQueue*中，然后构建一个消费者不断的去取到期的任务,进行处理就好.     

***优点：***  效率高,任务触发时间延迟低。    

***缺点：***  复杂度比quartz要高,自己要处理分布式横向扩展的问题,因为数据是放在内存里,需要自己写持久化的备案以达到高可用。   



### 利用wheelTimer      

netty中的Timer管理，使用了的Hashed time Wheel的模式，Time Wheel翻译为时间轮，是用于实现定时器timer的经典算法。   

#### HashWheelTimer的原理      

**时间轮算法的原理如图所示：**   

![时间轮算法的原理](http://7xpz5v.com1.z0.glb.clouddn.com/wheelTimer.png)

可以将 HashedWheelTimer 理解为一个 Set<Task>[] 数组, 图中每个槽位(slot)表示一个 Set<Task>   

HashedWheelTimer 有两个重要参数   

tickDuration:  每 tick 一次的时间间隔, 每 tick 一次就会到达下一个槽位    

ticksPerWheel: 轮中的 slot 数   

上图就是一个 ticksPerWheel = 8 的时间轮, 假如说 tickDuration = 100 ms, 则 800ms 可以走完一圈   

在 timer.start() 以后, 便开始 tick, 每 tick 一次, timer 会将记录总的 tick 次数 ticks   

我们加入一个新的超时任务时, 会根据超时的任务的超时时间与时间轮开始时间算出来它应该在的槽位.   


#### 怎么使用WheelTimer呢？   
    
在netty中已经有了时间轮算法的实现HashWheelTimer,HashWheelTimer的使用非常的简单:先new一个HashedWheelTimer，然后调用它的newTimeout方法传递相应的延时任务就ok了。   

下面是newTimeout的声明：       

```java

    /**
     * Schedules the specified {@link TimerTask} for one-time execution after
     * the specified delay.
     *
     * @return a handle which is associated with the specified task
     *
     * @throws IllegalStateException if this timer has been
     *                               {@linkplain #stop() stopped} already
     */
    Timeout newTimeout(TimerTask task, long delay, TimeUnit unit);

```   

这个方法需要一个TimerTask对象以知道当时间到时要执行什么逻辑，然后需要delay时间数值和TimeUnit时间的单位，像下面的例子中，我们在timer到期后会打印字符串，第一个任务是5秒后开始执行，第二个10秒后开始执行。   

```java   
public class HashWheelTimerTest {
    public static void main(String[] argv) {
        final Timer timer = new HashedWheelTimer();
        timer.newTimeout(new TimerTask() {
            public void run(Timeout timeout) throws Exception {
                System.out.println("timeout 5");
            }
        }, 5, TimeUnit.SECONDS);
        timer.newTimeout(new TimerTask() {
            public void run(Timeout timeout) throws Exception {
                System.out.println("timeout 10");
            }
        }, 10, TimeUnit.SECONDS);
    }
}

```   

***优点：***  效率高,根据楼主自己写的测试,在大量高负荷的任务堆积的情况下,HashWheelTimer基本要比delayQueue低上一倍的延迟率.netty中也有了时间轮算法的实现,实现难度低   

***缺点：***  内存占用相对较高,对时间精度要求相对不高.和delayQueue有着相同的问题,自己要处理分布式横向扩展的问题,因为数据是放在内存里,需要自己写持久化的备案以达到高可用。   

### rabbitMq的延迟队列      

大家都知道rabbitmq是一个消息队列,同时因为其天然的分布式特性的支持已经极高的消息处理效率深受大家的喜爱.那么大家应该不知道他也是可以用来处理我们的延时任务的.    

#### 如何使用rabbitMq的延迟队列    

* AMQP和RabbitMQ本身没有直接支持延迟队列功能，但是可以通过以下特性模拟出延迟队列的功能。   
* RabbitMQ可以针对Queue和Message设置 x-message-tt，来控制消息的生存时间，如果超时，则消息变为dead letter   
* lRabbitMQ的Queue可以配置x-dead-letter-exchange  和x-dead-letter-routing-key（可选）两个参数，用来控制队列内出现了deadletter，则按照这两个参数重新路由。   
* 结合以上两个特性，就可以模拟出延迟消息的功能    

具体实现可参照官方文档:    
[http://www.rabbitmq.com/ttl.html](http://www.rabbitmq.com/ttl.html)    
[http://www.rabbitmq.com/dlx.html](http://www.rabbitmq.com/dlx.html)   

***优点：***  高效,可以利用rabbitmq的分布式特性轻易的进行横向扩展,消息支持持久化增加了可靠性。     

***缺点：***  本身的易用度要依赖于rabbitMq的运维.因为要引用rabbitMq,所以复杂度和成本变高          
   




 