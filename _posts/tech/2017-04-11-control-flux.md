---
layout: post
title: 流量整形浅析
category: 技术
tags: [tech]
keywords: tpsLimit,tokenBucket
description: 流量整形浅析 
---


## 漏斗算法   

它有点像我们生活中用到的漏斗，液体倒进去以后，总是从下端的小口中以固定速率流出，漏斗算法也类似，不管突然流量有多大，漏斗都保证了流量的常速率输出，也可以类比于调用量，比如，不管服务调用多么不稳定，我们只固定进行服务输出，比如每10毫秒接受一次服务调用。既然是一个桶，那就肯定有容量，由于调用的消费速率已经固定，那么当桶的容量堆满了，则只能丢弃了，漏斗算法如下图：     
![image.png](http://upload-images.jianshu.io/upload_images/584578-d9f142130c2f915d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)    

#### 缺点    

漏斗算法其实是悲观的，因为它严格限制了系统的吞吐量，从某种角度上来说，它的效果和并发量限流很类似。漏斗算法也可以用于大多数场景，但由于它对服务吞吐量有着严格固定的限制，如果在某个大的服务网络中只对某些服务进行漏斗算法限流，这些服务可能会成为瓶颈。其实对于可扩展的大型服务网络，上游的服务压力可以经过多重下游服务进行扩散，过多的漏斗限流似乎意义不大。    

#### 实现：    

```java
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * <p>
 * 对请求做限速流控
 * </p>
 *
 * @author wangguangdong
 * @version 1.0
 * @Date 17/3/3
 */
public class LimitRequestByTime {
    long limit = 1000;

    Map<Long, AtomicLong> map = Collections.synchronizedMap(new LinkedHashMap<>(8));

    public boolean limitReq() {
        // 统计当前秒数
        long currentTimeMillis = System.currentTimeMillis() / 1000;
        map.putIfAbsent(currentTimeMillis, new AtomicLong(0));
        // 获取秒速流控
        AtomicLong currentAtomicLong = map.get(currentTimeMillis);
        return !(currentAtomicLong.incrementAndGet() >= limit);
    }
    public static void main(String[] args) {
        LimitRequestByTime limitRequestByTime = new LimitRequestByTime();
        // 统计所有的请求次数
        Map<Long, AtomicLong> totalMap = new ConcurrentHashMap<>();

        Map<Long, AtomicLong> successTotalMap = new ConcurrentHashMap<>();
        for (int i = 0; i < 100000000; i++) {
            // 统计当前这一秒的请求书
            long currentTimeMillis = System.currentTimeMillis() / 1000;
            totalMap.putIfAbsent(currentTimeMillis, new AtomicLong(0));
            // 自增加1
            totalMap.get(currentTimeMillis).incrementAndGet();

            successTotalMap.putIfAbsent(currentTimeMillis, new AtomicLong(0));
            if (limitRequestByTime.limitReq()) {
                successTotalMap.get(currentTimeMillis).incrementAndGet();
            }
        }

        for (Map.Entry<Long, AtomicLong> total : totalMap.entrySet()) {
            Long totalKey = total.getKey();
            System.out.println(String
                .format("在%d这一秒一共发送了%d次请求，通过的请求数量为%d", totalKey, totalMap.get(totalKey).get(),
                    successTotalMap.get(totalKey).get()));
        }
    }
}
```    

## 令牌桶算法    

令牌桶算法从某种程度上来说是漏桶算法的一种改进，漏桶算法能够强行限制请求调用的速率，而令牌桶算法能够在限制调用的平均速率的同时还允许某种程度的突发调用。在令牌桶算法中，桶中会有一定数量的令牌，每次请求调用需要去桶中拿取一个令牌，拿到令牌后才有资格执行请求调用，否则只能等待能拿到足够的令牌数，大家看到这里，可能就认为是不是可以把令牌比喻成信号量，那和前面说的并发量限流不是没什么区别嘛？其实不然，令牌桶算法的精髓就在于“拿令牌”和“放令牌”的方式，这和单纯的并发量限流有明显区别，采用并发量限流时，当一个调用到来时，会先获取一个信号量，当调用结束时，会释放一个信号量，但令牌桶算法不同，因为每次请求获取的令牌数不是固定的，比如当桶中的令牌数还比较多时，每次调用只需要获取一个令牌，随着桶中的令牌数逐渐减少，当到令牌的使用率（即使用中的令牌数/令牌总数）达某个比例，可能一次请求需要获取两个令牌，当令牌使用率到了一个更高的比例，可能一次请求调用需要获取更多的令牌数。同时，当调用使用完令牌后，有两种令牌生成方法，第一种就是直接往桶中放回使用的令牌数，第二种就是不做任何操作，有另一个额外的令牌生成步骤来将令牌匀速放回桶中。如下图：

![image1.png](http://upload-images.jianshu.io/upload_images/584578-fbca461472c52ae4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)    

#### 代码实现     

```java

import java.io.BufferedWriter;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.OutputStreamWriter;
import java.util.Random;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

import com.google.common.base.Preconditions;
/**
 * <p>
 *     令牌桶算法
 * </p>
 *
 * @author wangguangdong
 * @version 1.0
 * @Date 17/3/23
 */
public class TokenBucket  {

    // 默认桶大小个数 即最大瞬间流量是64M
    private static final int DEFAULT_BUCKET_SIZE = 1024 * 1024 * 64;

    // 一个桶的单位是1字节
    private int everyTokenSize = 1;

    // 瞬间最大流量
    private int maxFlowRate;

    // 平均流量
    private int avgFlowRate;

    // 队列来缓存桶数量：最大的流量峰值就是 = everyTokenSize*DEFAULT_BUCKET_SIZE 64M = 1 * 1024 * 1024 * 64
    private ArrayBlockingQueue<Byte> tokenQueue = new ArrayBlockingQueue<Byte>(DEFAULT_BUCKET_SIZE);

    private ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();

    private volatile boolean isStart = false;

    private ReentrantLock lock = new ReentrantLock(true);

    private static final byte A_CHAR = 'a';

    public TokenBucket() {
    }

    public TokenBucket(int maxFlowRate, int avgFlowRate) {
        this.maxFlowRate = maxFlowRate;
        this.avgFlowRate = avgFlowRate;
    }

    public TokenBucket(int everyTokenSize, int maxFlowRate, int avgFlowRate) {
        this.everyTokenSize = everyTokenSize;
        this.maxFlowRate = maxFlowRate;
        this.avgFlowRate = avgFlowRate;
    }

    public void addTokens(Integer tokenNum) {

        // 若是桶已经满了，就不再家如新的令牌
        for (int i = 0; i < tokenNum; i++) {
            tokenQueue.offer(A_CHAR);
        }
    }

    public TokenBucket build() {

        start();
        return this;
    }

    /**
     * 获取足够的令牌个数
     *
     * @return
     */
    public boolean getTokens(byte[] dataSize) {

        Preconditions.checkNotNull(dataSize);
        Preconditions.checkArgument(isStart, "please invoke start method first !");

        int needTokenNum = dataSize.length / everyTokenSize + 1;// 传输内容大小对应的桶个数

        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            boolean result = needTokenNum <= tokenQueue.size(); // 是否存在足够的桶数量
            if (!result) {
                return false;
            }

            int tokenCount = 0;
            for (int i = 0; i < needTokenNum; i++) {
                Byte poll = tokenQueue.poll();
                if (poll != null) {
                    tokenCount++;
                }
            }

            return tokenCount == needTokenNum;
        } finally {
            lock.unlock();
        }
    }


    public void start() {

        // 初始化桶队列大小
        if (maxFlowRate != 0) {
            tokenQueue = new ArrayBlockingQueue<Byte>(maxFlowRate);
        }

        // 初始化令牌生产者
        TokenProducer tokenProducer = new TokenProducer(avgFlowRate, this);
        scheduledExecutorService.scheduleAtFixedRate(tokenProducer, 0, 1, TimeUnit.SECONDS);
        isStart = true;

    }


    public void stop() {
        isStart = false;
        scheduledExecutorService.shutdown();
    }


    public boolean isStarted() {
        return isStart;
    }

    class TokenProducer implements Runnable {

        private int avgFlowRate;
        private TokenBucket tokenBucket;

        public TokenProducer(int avgFlowRate, TokenBucket tokenBucket) {
            this.avgFlowRate = avgFlowRate;
            this.tokenBucket = tokenBucket;
        }

        @Override
        public void run() {
            tokenBucket.addTokens(avgFlowRate);
        }
    }

    public static TokenBucket newBuilder() {
        return new TokenBucket();
    }

    public TokenBucket everyTokenSize(int everyTokenSize) {
        this.everyTokenSize = everyTokenSize;
        return this;
    }

    public TokenBucket maxFlowRate(int maxFlowRate) {
        this.maxFlowRate = maxFlowRate;
        return this;
    }

    public TokenBucket avgFlowRate(int avgFlowRate) {
        this.avgFlowRate = avgFlowRate;
        return this;
    }

    private String stringCopy(String data, int copyNum) {

        StringBuilder sbuilder = new StringBuilder(data.length() * copyNum);

        for (int i = 0; i < copyNum; i++) {
            sbuilder.append(data);
        }

        return sbuilder.toString();

    }

    public static void main(String[] args) throws IOException, InterruptedException {

        tokenTest();
    }

    private static void arrayTest() {
        ArrayBlockingQueue<Integer> tokenQueue = new ArrayBlockingQueue<Integer>(10);
        tokenQueue.offer(1);
        tokenQueue.offer(1);
        tokenQueue.offer(1);
        System.out.println(tokenQueue.size());
        System.out.println(tokenQueue.remainingCapacity());
    }

    private static void tokenTest() throws InterruptedException, IOException {
        TokenBucket tokenBucket = TokenBucket.newBuilder().avgFlowRate(512).maxFlowRate(1024).build();

        BufferedWriter bufferedWriter = new BufferedWriter(new OutputStreamWriter(new FileOutputStream("/tmp/ds_test")));
        String data = "xxxx";// 四个字节
        for (int i = 1; i <= 1000; i++) {

            Random random = new Random();
            int i1 = random.nextInt(100);
            boolean tokens = tokenBucket.getTokens(tokenBucket.stringCopy(data, i1).getBytes());
            TimeUnit.MILLISECONDS.sleep(100);
            if (tokens) {
                bufferedWriter.write("token pass --- index:" + i1);
                System.out.println("token pass --- index:" + i1);
            } else {
                bufferedWriter.write("token rejuect --- index" + i1);
                System.out.println("token rejuect --- index" + i1);
            }

            bufferedWriter.newLine();
            bufferedWriter.flush();
        }

        bufferedWriter.close();
    }

}
```

## 漏斗算法和令牌桶的异同点    

漏斗算法会限制平均的qps，对每个时间段的流控都是一样的，如果突然一瞬间的大流量进来，那么有可能会有大量请求被拦截住，
令牌桶算法的话除了能够限制数据的平均传输速率外，还允许一定时间内的大流量涌入，相当于漏斗算法的升级版本。



 