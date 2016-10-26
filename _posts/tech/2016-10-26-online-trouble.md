---
layout: post
title: 一次重大的线上事故的总结
category: 技术
tags: [tech]
keywords: 并发,事故,锁
description: 本文中总结了一次线上重大事故的发生,从起因,到发现,到解决,到总结问题
---

## 问题背景描述  

服务以及上线几个月,今天在客服查询用户的提现信息的时候,发现有的用户竟然出现了高达数千数万的提现请求.由于我们的用户体量并没有那么大,交易流水按理说也不应该有那么多,客服带着疑问跟我们报出了这个问题.   
于是,我便查询了最近的交易记录,发现有几个人是定向的而且多次的大额交易记录,然后我便查询了一下充值记录,发现用户的充值记录是根本没有那么多的,也就是说用户的账户余额发生了异常,那么问题出在哪了呢？    
 
## 发现问题  

带着上面的疑问,我仔细的检查了交易记录的数据,发现在交易记录中,有针对某个功能同一时间的大量请求,然后第一时间去查询用户的访问日志,发现存在某一ip同一时间内下针对某个接口地址的大量请求.  

例如:  
> 102.1.1.X 2016-10-23 16:66:01 /xxxxxxx/v1/money/opt/pee/  Post 
> 102.1.1.X 2016-10-23 16:66:01 /xxxxxxx/v1/money/opt/pee/  Post 
> 102.1.1.X 2016-10-23 16:66:01 /xxxxxxx/v1/money/opt/pee/  Post 
> 102.1.1.X 2016-10-23 16:66:01 /xxxxxxx/v1/money/opt/pee/  Post 
> 102.1.1.X 2016-10-23 16:66:01 /xxxxxxx/v1/money/opt/pee/  Post 
> 102.1.1.X 2016-10-23 16:66:01 /xxxxxxx/v1/money/opt/pee/  Post 
> 102.1.1.X 2016-10-23 16:66:01 /xxxxxxx/v1/money/opt/pee/  Post 

由此猜测,是被用户用程序盗刷了接口.    
那么追究其本质问题,就算是用户盗刷了接口,也不应该出现余额异常的问题,对应着这个接口的代码顺藤摸瓜屡下去.  

原代码是这个样子的(由于涉及到具体业务,这里用伪代码来代替):  

```java
    // 参数校验
    if(StringUtils.isEmty(xx)){
       throw new Exception(Error.param_error);
    } 
    // 用户余额校验 
    1.余额是否大于0
    2.余额是否充足
    // 检测是否查看过  
    boolean b = seeLogService.hasSee();
    if(b) throw new Exception(Error.has_see);
    // 更新查看日志
    seeLogService.update(xx);
    // 增加收入者用户余额并记录日志
    OrdersService.addUserGoldNum
    // 减少消费者用户余额并记录日志
    OrdersService.subtractUserGoldNum
```  

让我们仔细看一看上面的代码,会有什么问题？  
这里我们先带着我们想到的问题之处,去测试一下,首先我用`CyclicBarrier`做了一个并发的请求工具,工具类的代码如下:   

```java

import com.alibaba.fastjson.JSON;

import java.io.BufferedReader;
import java.io.DataOutputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Executors+CyclicBarrier实现的并发测试小例子<br>
 * 例子实现了并发测试中使用的集合点，集合点超时时间及思考时间等技术
 *
 * @author 王光东
 * @version 1.0
 * @date 2016年06月27日16:43:49
 */
public class FlushGeneratorPost {
    // 主线程停止的标志
    private static volatile boolean runFlag = true;
    // 栏杆
    private CyclicBarrier cyclicBarrier;
    // 线程数
    private static int threads;
    // 总数
    private static AtomicInteger totalCount = new AtomicInteger();
    // 已经开启的线程数
    private static AtomicInteger startedCount = new AtomicInteger();
    // 已经完成任务的线程数
    private static AtomicInteger finishCount = new AtomicInteger();
    // 正在进行的线程数
    private static AtomicInteger runCount = new AtomicInteger();
    // 成功的线程数
    private static AtomicInteger successCount = new AtomicInteger();
    // 失败的线程数
    private static AtomicInteger failCount = new AtomicInteger();
    // 请求的url地址
    private String url;
    // 集合点超时时间
    private long rendzvousWaitTime = 0;
    // 思考时间
    private long thinkTime = 0;
    // 次数
    private static int iteration = 0;

    /**
     * 初始值设置
     *
     * @param url               被测url
     * @param threads           总线程数
     * @param iteration         每个线程迭代次数
     * @param rendzvousWaitTime 集合点超时时间，如果不启用超时时间，请将此值设置为0.<br>
     *                          如果不启用集合点，请将此值设置为-1<br>
     *                          如果不启用超时时间，则等待所有的线程全部到达后，才会继续往下执行<br>
     * @param thinkTime         思考时间，如果启用思考时间，请将此值设置为0
     */
    public FlushGeneratorPost(String url, int threads, int iteration, long rendzvousWaitTime,
        long thinkTime) {
        totalCount.getAndSet(threads);
        FlushGeneratorPost.threads = threads;
        this.url = url;
        this.iteration = iteration;
        this.rendzvousWaitTime = rendzvousWaitTime;
        this.thinkTime = thinkTime;
    }

    // 过得线程数的信息
    public static ThreadCount getThreadCount() {
        return new ThreadCount(threads, runCount.get(), startedCount.get(), finishCount.get(),
            successCount.get(), failCount.get());
    }

    // 判断线程是否应该停止
    public static boolean isRun() {
        return finishCount.get() != threads;
    }

    // 优雅的停止线程
    public synchronized static void stop() {
        runFlag = false;
    }

    // 执行任务
    public void runTask() {
        List<Future<String>> resultList = new ArrayList<Future<String>>();
        // 线程池构造
        ExecutorService exeService = Executors.newFixedThreadPool(threads);
        cyclicBarrier = new CyclicBarrier(threads);//默认加载全部线程
        for (int i = 0; i < threads; i++) {
            resultList.add(
                exeService.submit(new TaskThread(i, url, iteration, rendzvousWaitTime, thinkTime)));
        }
        exeService.shutdown();
        for (int j = 0; j < resultList.size(); j++) {
            try {
                System.out.println(resultList.get(j).get());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        stop();
    }

    /**
     * 不同状态的线程数构造类
     */
    static class ThreadCount {
        public final int runThreads;
        public final int startedThreads;
        public final int finishedThreads;
        public final int totalThreads;
        public final int successCount;
        public final int failCount;

        public ThreadCount(int totalThreads, int runThreads, int startedThreads,
            int finishedThreads, int successCount, int failCount) {
            this.totalThreads = totalThreads;
            this.runThreads = runThreads;
            this.startedThreads = startedThreads;
            this.finishedThreads = finishedThreads;
            this.successCount = successCount;
            this.failCount = failCount;
        }
    }

    /**
     * 实际的业务线程类
     */
    private class TaskThread implements Callable<String> {
        private String url;
        private long rendzvousWaitTime = 0;
        private long thinkTime = 0;
        private int iteration = 0;
        private int iterCount = 0;
        private int taskId;

        /**
         * 任务执行者属性设置
         *
         * @param taskId            任务id号
         * @param url               被测url
         * @param iteration         迭代次数，如果一直执行则需将此值设置为0
         * @param rendzvousWaitTime 集合点超时时间，如果不需要设置时间，则将此值设置为0。如果不需要设置集合点，则将此值设置为-1
         * @param thinkTime         思考时间，如果不需要设置思考时间，则将此值设置为0
         */
        public TaskThread(int taskId, String url, int iteration, long rendzvousWaitTime,
            long thinkTime) {
            this.taskId = taskId;
            this.url = url;
            this.rendzvousWaitTime = rendzvousWaitTime;
            this.thinkTime = thinkTime;
            this.iteration = iteration;
        }

        @Override
        public String call() throws Exception {
            startedCount.getAndIncrement();
            runCount.getAndIncrement();
            while (runFlag && iterCount < iteration) {
                if (iteration != 0)
                    iterCount++;
                try {
                    if (rendzvousWaitTime > 0) {
                        try {
                            System.out.println("任务：task-" + taskId + " 已到达集合点...等待其他线程,集合点等待超时时间为："
                                + rendzvousWaitTime);
                            cyclicBarrier.await(rendzvousWaitTime, TimeUnit.MICROSECONDS);
                        } catch (InterruptedException e) {
                        } catch (BrokenBarrierException e) {
                            System.out.println(
                                "task-" + taskId + " 等待时间已超过集合点超时时间：" + rendzvousWaitTime
                                    + " ms,将开始执行任务....");
                        } catch (TimeoutException e) {
                        }
                    } else if (rendzvousWaitTime == 0) {
                        try {
                            System.out.println("任务：task-" + taskId + " 已到达集合点...等待其他线程");
                            cyclicBarrier.await();
                        } catch (InterruptedException e) {
                        } catch (BrokenBarrierException e) {
                        }
                    }
                    // 发送请求返回结果
                    Bean result = readContent(url);
                    System.out.println(
                        "线程：task-" + taskId + " 获取到的资源大小：" + result.getResult().length() + ",状态码："
                            + result.getState());
                    // 增加成功的值
                    successCount.getAndIncrement();
                    // 判断是否需要思考
                    if (thinkTime != 0) {
                        System.out.println("task-" + taskId + " 距下次启动时间：" + thinkTime);
                        Thread.sleep(thinkTime);
                    }
                } catch (Exception e) {
                    failCount.getAndIncrement();
                }
            }
            // 增加完成次数
            finishCount.getAndIncrement();
            // 减少运行的线程数量
            runCount.decrementAndGet();
            return Thread.currentThread().getName() + " 执行完成！";
        }
    }

    public static void main(String[] args) {
        final long startTime = System.currentTimeMillis();
        String baseUri = "http://localhost:8080/xxx/xx/xx/xx/xx";
        new Thread() {
            public void run() {   
                new FlushGeneratorPost(
                    baseUri, 20, 1,
                    0, 0).runTask(); //开启20个线程一次同时去请求这个接口
            }
        }.start();

        new Thread() {
            public void run() {
                while (isRun()) {
                    try {
                        Thread.sleep(5000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("isRun:" + FlushGeneratorPost.isRun());
                    System.out
                        .println("totalThreads:" + FlushGeneratorPost.getThreadCount().totalThreads);
                    System.out.println(
                        "startedThreads:" + FlushGeneratorPost.getThreadCount().startedThreads);
                    System.out.println("runThreads:" + FlushGeneratorPost.getThreadCount().runThreads);
                    System.out.println(
                        "finishedThread:" + FlushGeneratorPost.getThreadCount().finishedThreads);
                    System.out
                        .println("successCount:" + FlushGeneratorPost.getThreadCount().successCount);
                    System.out.println("failCount:" + FlushGeneratorPost.getThreadCount().failCount);
                    System.out.println();
                }
                System.out.println("\n\n 执行" + threads * iteration + "次请求一共花费了"
                    + (System.currentTimeMillis() - startTime) / 1000 + "秒");
            }
        }.start();
    }

    /**
     * httpUrlConnection的get请求
     *
     * @param uri
     * @return
     * @throws IOException
     */
    private static Bean readContent(String uri) throws IOException {
        String body = "xxx=111&xxx22=1&xxx33=2";
        URL postUrl = new URL(uri);
        // 打开连接
        HttpURLConnection connection = (HttpURLConnection) postUrl.openConnection();

        // 设置是否向connection输出，因为这个是post请求，参数要放在
        // http正文内，因此需要设为true
        connection.setDoOutput(true);
        // Read from the connection. Default is true.
        connection.setDoInput(true);
        // 默认是 GET方式
        connection.setRequestMethod("POST");
        connection.setConnectTimeout(3 * 1000);

        // Post 请求不能使用缓存
        connection.setUseCaches(false);

        connection.setInstanceFollowRedirects(true);

        // 配置本次连接的Content-type，配置为application/x-www-form-urlencoded的
        // 意思是正文是urlencoded编码过的form参数，下面我们可以看到我们对正文内容使用URLEncoder.encode
        // 进行编码
        connection.setRequestProperty("Content-Type", "application/x-www-form-urlencoded");
        connection.setRequestProperty("Accept", "application/json;charset=UTF-8");
        String Authorization = "xxxxxssss123sdffasdf";        connection.setRequestProperty("Authorization", Authorization);
        // 连接，从postUrl.openConnection()至此的配置必须要在connect之前完成，
        // 要注意的是connection.getOutputStream会隐含的进行connect。
        connection.connect();
        DataOutputStream out = new DataOutputStream(connection.getOutputStream());
        // The URL-encoded contend
        // 正文，正文内容其实跟get的URL中 '? '后的参数字符串一致
        //        content = "count=" + URLEncoder.encode(String.valueOf(1), "UTF-8");
        //        content +="&amount="+URLEncoder.encode(String.valueOf(10), "UTF-8");
        // DataOutputStream.writeBytes将字符串中的16位的unicode字符以8位的字符形式写到流里面
        out.writeBytes(body);

        out.flush();
        out.close();

        BufferedReader reader = new BufferedReader(new InputStreamReader(connection.getInputStream()));
        String line;

        StringBuilder sb = new StringBuilder();
        while ((line = reader.readLine()) != null) {
            sb.append(line);
        }


        int state = connection.getResponseCode();
        reader.close();
        connection.disconnect();

        return new Bean(state, sb.toString());




    }

    /**
     * 结果集实体类
     */
    public static class Bean {
        private int state;
        private String result;

        public Bean(int state, String result) {
            this.state = state;
            this.result = result;
        }

        public int getState() {
            return state;
        }

        public void setState(int state) {
            this.state = state;
        }

        public String getResult() {
            return result;
        }

        public void setResult(String result) {
            this.result = result;
        }
    }
}

```

然后利用我的这个工具去还原请求我们被盗刷的接口.我们去一点点的验证我们猜想的问题所在,首先,我用了一个账户余额充足的账户去测试,发现没什么问题.  
这个地方为什么没有问题呢?我在加钱的服务和减少金钱的服务中利用了数据库的悲观锁保证了用户余额的安全(这里我们先不吐槽悲观锁的问题哈).   
然后我又换了一个余额少一点的账户进行了重试,发现这次的就出问题了,加钱的用户正确加上了20*price的金额,而扣钱的用户账户为0,也就是说,假如我只有20快,但是我想花100快给a,这时候我余额变成0了(也就是花了20快),然后a的余额增加了100快,那等于系统因为bug原因陪了80快.   

这时候我们再回顾一下之前我们猜测的问题   

1. 首先我的最外层的金额检测,没有事务和锁的保证,所以在并发的时候,这一层的检测就变成了无效的检测.   
2. 检测是否查看过由于同事的疏忽,没有做并发时候的唯一性校验,即如果用户看过一次就不能再继续再看了,所以这一层的检测在并发的时候也会出问题.   
3. 由于并发请求使得1和2的校验都不成功的时候这时候就来到了第三步,这时候我们是要进行真正的金额变动和变动日志的记录了,这个时候我们看一下具体的代码实现.    

加钱服务的代码    

```/**
     * 用户加钱的api
     *
     * @param userId                  用户id
     * @param amount                  钻石金额
     * @param orderId                 订单id
     * @param orderType               订单类型
     * @param moneyLogOrderChangeType 加钱的理由
     * @param relationUserId          这个字段意味这次加钱是跟谁有关的
     * @return
     */
    public int addUserGoldNum(String userId, int amount, String orderId, OrderType orderType,
                              MoneyLogOrderChangeType moneyLogOrderChangeType, String relationUserId) {
        if (userId == null)
            throw new BusinessException(ErrorCode.NOT_FIND_USER_ACCOUNT);
        int currentAmount = userInfoDAO.queryGoldNumByUserId(userId);
        if (amount <= 0)
            return currentAmount;
        int newAmount = currentAmount + amount;
        LOG.info(
                "<Important!> add BEGIN!! [userId]= " + userId + " [current amount]= " + currentAmount
                        + " [new ammount]= " + newAmount);
        userInfoDAO.addGoldNumByUserId(userId, amount);
        // 获得用户在内存中信息
        UserInfo userInfo = UserCache.getInstance().loadUserInfo(userId);
        if (userInfo != null) {
            userInfo.setGoldNum(newAmount);
            UserCache.getInstance().updateUserInfo(userInfo);
        } else
            throw new BusinessException(ErrorCode.NOT_FIND_USER);
        LOG.info("<Important!> add COMPLETE!! [userId]= " + userId + " [current amount]= "
                + currentAmount + " [new ammount]= " + newAmount);

                userService.updateUserInfo(userInfo);
        //更新data日志
        DailyUserDataLogCache.incrDiamonds(userId, amount);
        //记录到money log
        MoneyLog moneyLog = new MoneyLog();
        moneyLog.setUserId(userId);
        moneyLog.setAmountType(MoneyLogAmountType.ADD.getType());
        moneyLog.setChangeAmount(amount);
        moneyLog.setChangeType(moneyLogOrderChangeType.getType());
        moneyLog.setChangeReason(moneyLogOrderChangeType.getReason());
        moneyLog.setOrderId(orderId);
        moneyLog.setRelationId(relationUserId);
        moneyLog.setType(orderType.getType());
        moneyLogService.save(moneyLog);

        return newAmount;

    }
```

减钱服务的代码    

```
public int subtractUserGoldNum(String userId, int amount, String orderId, OrderType orderType,
                                   MoneyLogOrderChangeType moneyLogOrderChangeType, String relationUserId) {
        if (userId == null)
            throw new BusinessException(ErrorCode.NOT_FIND_USER_ACCOUNT);
        int currentAmount = userInfoDAO.queryGoldNumByUserId(userId);
        int newAmount = currentAmount - amount;
        if (newAmount < 0)
            throw new BusinessException(ErrorCode.INSUFFICIENT_MONEY);
        LOG.info("<Important!> subtract BEGIN!! [userId]= " + userId + " [current amount]= "
                + currentAmount + " [new ammount]= " + newAmount);
        userInfoDAO.subtractGoldNumByUserId(userId, amount);
        // 获得用户在内存中信息
        UserInfo userInfo = UserCache.getInstance().loadUserInfo(userId);
        if (userInfo != null) {
            userInfo.setGoldNum(newAmount);
            UserCache.getInstance().updateUserInfo(userInfo);
        } else
            throw new BusinessException(ErrorCode.NOT_FIND_USER);
        LOG.info("<Important!> subtract COMPLETE!! [userId]= " + userId + " [current amount]= "
                + currentAmount + " [new ammount]= " + newAmount);

        //记录到money log
        MoneyLog moneyLog = new MoneyLog();
        moneyLog.setUserId(userId);
        moneyLog.setAmountType(MoneyLogAmountType.SUBTRACT.getType());
        moneyLog.setChangeAmount(amount);
        moneyLog.setChangeType(moneyLogOrderChangeType.getType());
        moneyLog.setChangeReason(moneyLogOrderChangeType.getReason());
        moneyLog.setOrderId(orderId);
        moneyLog.setRelationId(relationUserId);
        moneyLog.setType(orderType.getType());
        moneyLogService.save(moneyLog);

        return newAmount;
    }
```

我们可以看到,按照常理来说,我们的扣钱服务中有了对余额的检测,如果余额不够会抛出业务异常,让数据回滚,那么案例来说我们的程序应该是没问题的啊?但是仔细查询消费日志表的时候会发现,正常我加钱和扣钱都会记录一条记录,也就是我加钱和扣钱的记录数量是相等的,但是在我们并发请求了之后,我的数据库中的记录数是不对等的,我的扣钱记录比加钱记录少,这是为什么呢？      
其实我们之前已经说过,无论是加钱服务还是减钱服务都是有悲观锁来保证的,那么这个悲观锁是怎么回事呢,其实就是一个for update语句,在事务提交了之后会自动释放锁,但是由于我们的项目是一个编程式事务,而这个服务的加钱和扣钱直接在view层调用了,所以这时候这两个服务是两个事物,所以即使扣钱服务发生了异常,那么我们之前的钱已经给收入者加过了,这时候是无法回滚的.  
然后再看一下我们的业务,重新整理思考一下逻辑,在并发请求的调用中,给用户a加钱服务调用完之后,我们的需要调用扣钱服务给b扣费,这时候我们发现用户余额不足了,而抛出异常,但是给a加的钱并没有还原回去,然后b的余额也只是0而已.    

## 解决问题  

其实我们在上面的原子服务中已经做了很多的检测,然而因为疏忽的问题造成了现在的问题,要解决这个问题有几种办法也都很简单.   

我们先去除view层的余额校验(这里去除他是因为没啥用,属于一个优化代码)   

1. 在检测用户是否查看过的地方增加锁和唯一性校验,保证用户只偷看一次(这种方法其实治标不治本,也只是针对于这个接口能保证没有问题)   
2. 把扣钱服务在加钱服务之前调用,这样扣钱服务发送异常的时候,就会熔断,不会继续走加钱服务(这种方式代码改动量最小,不过保不准哪个同事继续会出现这样的疏忽).     
3. 抽象出一层组合的服务层,吧扣钱和加钱放在一个事务之中,如果有其他的业务就继续组合,让业务处于同一个事务下,既可以保证数据安全,又能保证业务正确.还可以规范化整个开发中的代码调用(这种方式也是我比较推荐的,而且在日后做服务化和架构梳理的时候也会比较方便,而且来了新人也不容易出问题)  

## 总结问题  

我们上面已经把问题找到并解决了问题,不过已经异常的数据还是让本宝宝在10.24程序员节日的时候忙活了很久很久,可坑坏本宝宝了,于是乎楼主便整理了这个大事件中所暴漏出来的问题.   

1. 接口没做签名和加密(sign,base64没做,客户端未做混淆,用户可以轻易的请求到服务接口)     
2. 相同的请求数据没做过滤(例如加上接口调用会话id来过滤)       
3. 接口没做组合服务(这也是整个事件过程中比较重要的一个地方,因为代码未做规范化,所以才会暴漏了这么严重的问题,如果统一了服务调用就不会出现这个问题了,就像加钱和扣钱的原子服务以及处理了很多会发生的问题了)     
4. 没做并发测试,测试点不足(不提了,可能很多公司都有这个通病吧,哎,以后要重点注意了)    
5. 用户金钱安全性保证不够(给不了你心爱的用户安全感,拿什么说爱他们)   
6. 未做风控检测(考虑每天跑跑定时任务,做一些业务检测)     

