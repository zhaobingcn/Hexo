---
title: Zookeeper学习笔记六
date: 2016-08-03 14:35:52
tags: Zookeeper
categories: Zookeeper
---

本章开始学习下Curator一些典型的使用场景，可以为实际项目提供参考。
## 事件监听
Zookeeper原声支持通过注册Watcher来进行事件监听，但使用起来不是特别方便，需要反复注册Watcher，比较繁琐。Curator引入了Cache来实现对Zookeeper服务端事件的监听，Cache是Curator中对事件监听的包装，并且自动反复注册监听。Cache分为两类监听类型：节点监听和子节点监听。

## NodeCache
NodeCache用于监听指定Zookeeper数据节点本身的变化。

```
public class NodeCacheSample {
    static String path = "/zk-book/nodecache";
    static CuratorFramework client = CuratorFrameworkFactory.builder()
            .connectString("localhost:2181")
            .sessionTimeoutMs(5000)
            .retryPolicy(new ExponentialBackoffRetry(1000, 3)).build();
    public static void main(String[] args) throws Exception {
        client.start();
        final NodeCache cache = new NodeCache(client, path, false);
        cache.start(true);
        cache.getListenable().addListener(() -> {
            System.out.println("NodeCacheListener...");
            if (cache.getCurrentData() != null) {
                System.out.println("Node data update, new data: " +
                        new String(cache.getCurrentData().getData()));
            }
        });
        //创建节点会触发NodeCacheListener
        client.create()
                .creatingParentsIfNeeded()
                .withMode(CreateMode.EPHEMERAL)
                .forPath(path, "init".getBytes());
        Thread.sleep(1000);
        /**
         * 修改节点会触发NodeCacheListener
         * 但是只会输出"y"，所以猜测NodeCache不适用并发修改场景
         */
        client.setData().forPath(path, "x".getBytes());
        client.setData().forPath(path, "y".getBytes());
        Thread.sleep(1000);
        //该版本删除节点会触发NodeCacheListener
        client.delete().deletingChildrenIfNeeded().forPath(path);
        Thread.sleep(1000);
    }
}
```

## PathChildrenCache
PathChildrenCache用于监听指定Zookeeper数据节点的子节点变化情况，无法对二级子节点进行事件监听。
```
public class PathChildrenCacheSample {
    static String path = "/zk-book";
    static CuratorFramework client = CuratorFrameworkFactory.builder()
            .connectString("localhost:2181")
            .retryPolicy(new ExponentialBackoffRetry(1000, 3))
            .sessionTimeoutMs(5000).build();
    public static void main(String[] args) throws Exception {
        client.start();
        PathChildrenCache cache = new PathChildrenCache(client, path, true);
        cache.start(StartMode.POST_INITIALIZED_EVENT);
        cache.getListenable().addListener((client1, event) -> {
            switch (event.getType()) {
                case CHILD_ADDED:
                    System.out.println("CHILD_ADDED," + event.getData().getPath());
                    break;
                case CHILD_UPDATED:
                    System.out.println("CHILD_UPDATED," + event.getData().getPath());
                    break;
                case CHILD_REMOVED:
                    System.out.println("CHILD_REMOVED," + event.getData().getPath());
                    break;
                default:
                    //CONNECTION_RECONNECTED、INITIALIZED
                    System.out.println(event.getType());
                    break;
            }
        });
        client.create().withMode(CreateMode.PERSISTENT).forPath(path);
        Thread.sleep(1000);
        //新增子节点会触发PathChildrenCacheListener
        client.create().withMode(CreateMode.PERSISTENT).forPath(path + "/c1");
        Thread.sleep(1000);
        //删除子节点会触发PathChildrenCacheListener
        client.delete().forPath(path + "/c1");
        Thread.sleep(1000);
        client.delete().forPath(path);
        Thread.sleep(1000);
    }
}
```

## Master选举
在分布式系统中，经常会碰到这样的场景：对于一个复杂的任务，仅需要从集群中选举出一台进行处理即可。诸如此类的分布式问题，我们统称为Master选举，借助Zookeeper可以轻松实现。其思路为：选在一个根节点，例如/master_select，多台机器同时想该节点创建一个子节点/master_select/lock，利用Zookeeper的特性，最终最有一台机器能够创建成功，那台机器就成为Master。

```
public class RecipesMasterSelect {
    static String master_path = "/curator_recipes_master_path";
    static CuratorFramework client = CuratorFrameworkFactory.builder()
            .connectString("localhost:2181")
            .retryPolicy(new ExponentialBackoffRetry(1000, 3)).build();
    public static void main(String[] args) throws Exception {
        client.start();
        LeaderSelector selector = new LeaderSelector(client,
                master_path,
                new LeaderSelectorListenerAdapter() {
                    /**
                     * 一旦执行完takeLeadership方法，Curator就会立即释放Master权利，重新开始新一轮的Master选举
                     * @param client
                     * @throws Exception
                     */
                    public void takeLeadership(CuratorFramework client) throws Exception {
                        System.out.println("成为Master角色");
                        Thread.sleep(3000);
                        System.out.println("完成Master操作，释放Master权利");
                    }
                });
        selector.autoRequeue();
        selector.start();
        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

