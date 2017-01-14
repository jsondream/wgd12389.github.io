---
layout: post
title: netty集群方案
category: 技术
tags: [tech]
keywords: netty,cluster
description: netty cluster 
---


## 集群构建的思路  

1. 服务的注册   
2. 服务的提供   
3. 服务的发现   
4. 负载均衡策略   

### 服务的注册  

要实现服务的注册，首先我们要有一个注册中心(这里我们选择主流的zookeeper作为注册中心)，然后，我们在每一个netty服务启动的时候,吧本地的一些服务信息，比如ip地址+端口号注册到zookeeper上。    

代码实现如下:    

```java
CuratorFramework cf;

public void register(String nodeName, String ip) throws Exception {
        String path = ZK_CLUSTER_PATH + "/" + nodeName;
        Stat stat = cf.checkExists().forPath(path);
        if (stat == null) {
           // 如果父节点不存在就先创建父节点
           cf.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(path);
        }
        System.out.println("获取本机ip:" + ip);
        // 创建对应的临时节点
        cf.create().withMode(CreateMode.EPHEMERAL).forPath(path + "/" + ip, ip.getBytes());
    }
```

这里大家应该可以看见，我们关键的地方就是在在于对应的ip节点的创建，那么大家可以看见我们创建的是EPHEMERAL节点，这个节点是个临时节点。   

#### 什么是临时节点？   

 服务启动后创建临时节点， 服务断掉后临时节点就不存在了
 
#### 为什么使用临时节点？   

 正常的思路可能是注册的时候，我们像zk注册一个正常的节点，然后在服务下线的时候删除这个节点，但是这样的话会有一个弊端。    
 比如我们的服务泵机了，无法去删除临时节点，那么这个节点就会被我们错误的提供给了客户端。    
 另外我们还要考虑持久化的节点创建之后删除之类的问题，问题会更加的复杂化，所以我们使用了临时节点。   

### 服务的提供   

ok我们上面知道了我们服务注册的流程，那么我们需要吧注册的服务提供给客户端。   

这里其实关键的问题就是如何知道哪些服务是可用的。   
这里有两种方式：   

1. 每次都去zk里去查询   
2. 本地缓存一个列表   

这里我们先说一下，第一种方式每次都要走网络的调用，效率太低，不予考虑。那么大家可能想了一下，用第一种方式的，如果我的服务下线，或者新服务上线，我怎么办，这里就引入我们的下一个步骤了：**服务的发现**，利用我们的服务发现机制来更新本地的缓存，这样的话，我们每次都是在本地中去取服务的列表，节省了很大的网络开销。   

### 服务的发现  
    
上面我们说到`服务的列表变更去更新本地的缓存`，那么关键的是我们怎么去实现呢。   
了解zk的朋友应该都知道，zk有一个watch机制，就是针对某个节点进行监听，一点这个节点发生了变化就会受到zk的通知。   
我们就是利用zk的这个watch来进行服务的上线和下线的通知，也就是我们的服务发现功能。   

代码实现如下:   

```java
    CuratorFramework cf;

    public void subscribe(String nodeName) throws Exception {
        //订阅某一个服务
        final String path = ZK_CLUSTER_PATH + "/" + nodeName;
        Stat stat = cf.checkExists().forPath(path);
        if (stat == null) {
            // 如果父节点不存在就先创建父节点
            cf.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(path);
        }
        PathChildrenCache cache = new PathChildrenCache(cf, path, true);
        // 在初始化的时候就进行缓存监听
        cache.start(PathChildrenCache.StartMode.POST_INITIALIZED_EVENT);
        cache.getListenable()
            .addListener((CuratorFramework client, PathChildrenCacheEvent event) -> {
                // 重新获取子节点
                List<String> children = client.getChildren().forPath(path);
                // 排序一下子节点
                Collections.sort(children);
                // 子节点重新缓存起来
                data.put(nodeName, children);
            });
    }
```
    
### 负载均衡策略   

ok,在我们解决了服务的注册和发现问题之后，那么我们究竟提供给客户端那台服务呢，这时候就需要我们做出选择，为了让客户端能够均匀的连接到我们的服务器上（比如有个100个客户端，2台服务器，每台就分配50个），我们需要使用一个`负载均衡`的策略。   

这里我们使用轮询的方式来为每个请求的客户端分配ip。具体的代码实现如下：   

```java

public class RoundRobinLoadBalance {

    public static final String NAME = "roundrobin";

    private static final AtomicInteger sequences = new AtomicInteger();

    public static String doSelect(List<String> callList) {
        // 取模轮循
        return callList.isEmpty() ?
            null :
            callList.get(sequences.getAndIncrement() % callList.size());
    }

}

```

### 测试下效果   

我们已经完成了整个集群的搭建了，那么我们编写一个测试程序，来测一下我们的效果。   

完整的测试代码如下:   


```java
public static List<String> dataList = new ArrayList<>();

    static {

        dataList.add("19.21.2.1");
        dataList.add("12.34.33.12");
        dataList.add("21.1.235.22");
        dataList.add("6.12.36.233");
        dataList.add("71.12.36.233");
    }

    /**
     * Zookeeper info
     */

    public static void main(String[] args) throws Exception {
        String ZK_ADDRESS = "127.0.0.1:2181";
        String Node = "IM";
        String path = "/zk_cluster/IM";
        // Connect to zk
        CuratorFramework client =
            CuratorFrameworkFactory.newClient(ZK_ADDRESS, new RetryNTimes(10, 5000));
        client.start();

        List<String> children = client.getChildren().forPath(path);

        assert children.isEmpty();
        // test
        ZkList zkList = new ZkList(client);
        // 先订阅
        zkList.subscribe(Node);
        // 添加节点
        dataList.stream().forEach(ip -> {
            try {
                zkList.register(Node, ip);
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
        // 初始化一个统计的map,来检测每个节点被分配的次数
        Map<String, AtomicInteger> map = new ConcurrentHashMap<>();
        dataList.stream().forEach(ip -> {
            try {
                map.putIfAbsent(ip, new AtomicInteger(0));
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
        //Map
        for (int i = 0; i < 20000; i++) {
            String node = zkList.selectNode(Node);
            map.get(node).incrementAndGet();
        }

        for (Map.Entry<String, AtomicInteger> m : map.entrySet()) {
            System.out.println(String.format("[%s]一共被执行了:%s次", m.getKey(), m.getValue().get()));
        }

    }
```   

看一下我们的效果

```
[12.34.33.12]一共被执行了:4000次
[21.1.235.22]一共被执行了:4000次
[6.12.36.233]一共被执行了:4000次
[19.21.2.1]一共被执行了:4000次
[71.12.36.233]一共被执行了:4000次
```

我们在更改一下我们的测试程序，模拟一下节点上线和下线,吧我们去查询节点的部分来改一下，具体的代码如下：      

```java
        for (int i = 0; i < 20001; i++) {
            if (i == 10000) {
                zkList.unRegister(Node, dataList.get(0));
            }
            if (i == 15000) {
                zkList.register(Node, dataList.get(0));
            }
            String node = zkList.selectNode(Node);
            map.get(node).incrementAndGet();
            System.out.println(node);
        }
```

实际的效果如下：   

```java
[12.34.33.12]一共被执行了:4285次
[21.1.235.22]一共被执行了:4284次
[6.12.36.233]一共被执行了:4284次
[19.21.2.1]一共被执行了:2863次
[71.12.36.233]一共被执行了:4285次
```

ok大家能看到我们的请求基本被均匀的分配到了每个节点，无论有服务上线还是下线都不会过多的影响到服务的分配。

## Note   

大家可以看到上面做那种服务注册的时候，按照我们现在的逻辑，先去注册一个临时节点，
如果这个时候我服务挂了，我瞬间重启，zk一般对临时节点的对应session断开的话默认会有3次的重试判断的，那么我在这3次重试判断完成之前吧我的服务重启完了，那么我这时候去注册当前的临时节点的话，又会因为节点存在而无法创建，然后过一会zk检测完原来的节点了，发现原来的服务挂掉了，吧节点删除了，那么也就是说我这个节点除非再次重启，否则无法正常的注册成功了。   

### 方式一    

如果，我在注册的时候判断节点是否存在，不存在的话正常注册，存在的话删除原来的节点并且重新注册。
但是感觉这么做的话，感觉在某些并发情况下，可能会产生其他的新问题

### 方式二    

这个时候我们可以提供两种服务注册失败时候的策略：   

* 第一种就是服务注册失败抛出异常，让用户知道服务没有注册成功。   
* 第二种方式就是定时重试，如果注册失败了就过一点时间之后继续重新尝试注册，到一定次数之后在选择失败或者其他策略。   


## 总结       

让我们再看一下整个过程的步骤：   

1. 我们在每一个netty服务启动的时候,吧本地的一些服务信息，比如ip地址+端口号注册到zookeeper上    
2. 有一个对外提供netty集群列表的服务(可以是常规的http服务)，在本地缓存一个服务列表，我们的这个服务去监听zk上对应的节点变化，如果有变化就改变自己的缓存服务列表。    
3. 客户端每次去对外提供的服务请求，服务端根据相应的策略去选择节点提供给客户端(这里我们使用轮询的方式)，客户端根据服务端提供的节点地址去建立连接进行业务通信。  
 