---
title: zookeeper学习笔记三【java客户端API的使用】
date: 2016-07-26 12:22:55
tags:
     - Zookeeper
categories: Zookeeper
---

Zookeeper提供了源生Java Api，下面我们新建个项目来测试，之后所有的测试代码都放于该项目中，项目地址为zookeeper-sample。

```
<!-- Zookeeper -->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.9</version>
    <!-- Multiple SLF4J bindings error-->
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
        <exclusion>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

## 创建会话，连接服务端
```
//Java API -> 创建连接 -> 创建一个最基本的ZooKeeper对象实例
public class ZooKeeperConstructorUsageSimple implements Watcher {
    private static CountDownLatch connectedSemaphore = new CountDownLatch(1);
    public static void main(String[] args) throws Exception{
        /**
         * Zookeeper客户端和服务端会话的建立是一个异步的过程
         * 也就是说在程序中，构造方法会在处理完客户端初始化工作后立即返回
         * 在大多数情况下此时并没有真正建立好一个可用的会话，此时在会话的生命周期中处于“CONNECTING”的状态
         */
        ZooKeeper zookeeper = new ZooKeeper("localhost:2181", 5000,
                new ZooKeeperConstructorUsageSimple());
        System.out.println(zookeeper.getState());
        try {
            //等待Watcher通知SyncConnected
            connectedSemaphore.await();
        } catch (InterruptedException e) {}
        System.out.println("ZooKeeper session established.");
    }
    /**
     * ZooKeeper_Constructor_Usage_Simples实现了Watcher接口，重写了process方法
     * 该方法负责处理来自Zookeeper服务端的Watcher通知，即服务端建立连接后会调用该方法
     * @param event
     */
    public void process(WatchedEvent event) {
        System.out.println("Receive watched event：" + event);
        if (Event.KeeperState.SyncConnected == event.getState()) {
            connectedSemaphore.countDown();
        }
    }
}

```

```
//Java API -> 创建连接 -> 创建一个最基本的ZooKeeper对象实例，复用sessionId和password
public class ZooKeeperConstructorUsageWithSidPassword implements Watcher {
    private static CountDownLatch connectedSemaphore = new CountDownLatch(1);
    public static void main(String[] args) throws Exception{
        ZooKeeper zookeeper = new ZooKeeper("localhost:2181", 5000,
                new ZooKeeperConstructorUsageWithSidPassword());
        connectedSemaphore.await();
        /**
         * 获取sessionId、password，目的是为了复用会话
         */
        long sessionId = zookeeper.getSessionId();
        byte[] password  = zookeeper.getSessionPasswd();
        //使用错误的sessionId和password连接
        zookeeperConnector wrong = new zookeeperConnector(1, "test".getBytes(), new CountDownLatch(1));
        wrong.connect();
        //使用正确的sessionId和password连接
        zookeeperConnector correct = new zookeeperConnector(sessionId, password, new CountDownLatch(1));
        correct.connect();
    }
    public void process(WatchedEvent event) {
        System.out.println("Receive watched event：" + event);
        if (KeeperState.SyncConnected == event.getState()) {
            connectedSemaphore.countDown();
        }
    }
    static class zookeeperConnector implements Watcher{
        private long sessionId;
        private byte[] password;
        private CountDownLatch connectedSemaphore;
        public zookeeperConnector(long sessionId, byte[] password, CountDownLatch connectedSemaphore){
            this.sessionId = sessionId;
            this.password = password;
            this.connectedSemaphore = connectedSemaphore;
        }
        public void connect() throws IOException, InterruptedException {
            new ZooKeeper("localhost:2181", 5000, this, sessionId, password);
            this.connectedSemaphore.await();
        }
        @Override
        public void process(WatchedEvent watchedEvent) {
            System.out.println("Receive watched event：" + watchedEvent);
            this.connectedSemaphore.countDown();
        }
    }
}
//从输出中截取出三条能代表结果的信息如下：
Receive watched event：WatchedEvent state:SyncConnected type:None path:null
Receive watched event：WatchedEvent state:Expired type:None path:null
Receive watched event：WatchedEvent state:SyncConnected type:None path:null
```

## 创建节点
创建节点的API分同步和异步两种方式，无论时同步还是异步接口，Zookeeper都不支持递归创建。

```
//ZooKeeper API创建节点，使用同步(sync)接口。
public class ZooKeeperCreateAPISyncUsage implements Watcher {
    private static CountDownLatch connectedSemaphore = new CountDownLatch(1);
    public static void main(String[] args) throws Exception{
        ZooKeeper zookeeper = new ZooKeeper("localhost:2181", 5000,
				new ZooKeeperCreateAPISyncUsage());
        connectedSemaphore.await();
        /**
         * 创建临时节点
         */
        String path1 = zookeeper.create("/zk-test-ephemeral-", "".getBytes(),
        		Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
        System.out.println("Success create znode: " + path1);
        /**
         * 创建临时顺序节点
         * Zookeeper会自动在节点后缀加上一个数字
         */
        String path2 = zookeeper.create("/zk-test-ephemeral-", "".getBytes(),
        		Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        System.out.println("Success create znode: " + path2);
        /**
         * 创建永久节点
         */
        String path3 = zookeeper.create("/persistent-node","bboyjing".getBytes(),
                Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        System.out.println("Success create znode: " + path3);
    }
    public void process(WatchedEvent event) {
        if (KeeperState.SyncConnected == event.getState()) {
            connectedSemaphore.countDown();
        }
    }
}
//在zookeeper客户端中查看结果
[zk: localhost:2181(CONNECTED) 7] ls /
[zookeeper, persistent-node]
[zk: localhost:2181(CONNECTED) 8] get /persistent-node
zhaobing
```

## 删除节点

删除节点API分同步和异步两种方式，只允许删除叶子节点。

```
// ZooKeeper API 删除节点，使用同步(sync)接口。
public class DeleteAPISyncUsage implements Watcher {
    private static CountDownLatch connectedSemaphore = new CountDownLatch(1);
    public static void main(String[] args) throws Exception {
        ZooKeeper zk = new ZooKeeper("localhost:2181", 5000, new DeleteAPISyncUsage());
    	connectedSemaphore.await();
        /**
         * 删除节点，需要注意的是至允许删除叶子节点
         */
        zk.delete("/persistent-node", -1);
    }
    @Override
    public void process(WatchedEvent event) {
        if (KeeperState.SyncConnected == event.getState()) {
            if (EventType.None == event.getType() && null == event.getPath()) {
                connectedSemaphore.countDown();
            }
        }
    }
}
```

## 读取节点数据

读取的数据包括子节点列表和节点数据，Zookeeper提供了不同的API，而且还能注册Watcher来订阅节点相关信息的变化。

### 读取子节点(getChildren)
```
// ZooKeeper API 获取子节点列表，使用同步(sync)接口。
public class ZooKeeperGetChildrenAPISyncUsage implements Watcher {
    private static CountDownLatch connectedSemaphore = new CountDownLatch(1);
    private static CountDownLatch watcherSemaphore = new CountDownLatch(1);
    private static ZooKeeper zk = null;
    public static void main(String[] args) throws Exception{
        /**
         * 声明node路径
         * 实例化Zookeeper
         */
    	String path = "/zk-book";
        zk = new ZooKeeper("localhost:2181", 500000, new ZooKeeperGetChildrenAPISyncUsage());
        connectedSemaphore.await();
        /**
         * 创建永久节点/zk-book
         * 创建临时节点/zk-book/c1
         */
        zk.create(path, "".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        zk.create(path + "/c1", "".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
        /**
         * 获取/zk-book下的子节点
         * 此时注册了默认的watch，如果继续在/zk-book下增加节点的话，会调用process方法，通知客户端节点变化了
         * 但是仅仅是发通知，客户端需要自己去再次查询
         * 另外需要注意的是watcher是一次性的，即一旦触发一次通知后，该watcher就失效了，需要反复注册watcher，
         * 即process方中的getChildren继续注册了watcher
         */
        List<String> childrenList = zk.getChildren(path, true);
        System.out.println(childrenList);
        zk.create(path+"/c2", "".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
        watcherSemaphore.await();
    }
    public void process(WatchedEvent event) {
        if (KeeperState.SyncConnected == event.getState()) {
            if (EventType.None == event.getType() && null == event.getPath()) {
                connectedSemaphore.countDown();
            } else if (event.getType() == EventType.NodeChildrenChanged) {
                try {
                    //收到子节点变更通知，重新主动查询子节点信息
                    System.out.println("ReGet Child:" + zk.getChildren(event.getPath(),true));
                    watcherSemaphore.countDown();
                } catch (Exception e) {}
            }
        }
    }
}
```

### 获取节点数据(getData)
```
// ZooKeeper API 获取节点数据内容，使用同步(sync)接口。
public class GetDataAPISyncUsage implements Watcher {
    private static CountDownLatch connectedSemaphore = new CountDownLatch(1);
	private static CountDownLatch nodeDataChangedSemaphore = new CountDownLatch(1);
    private static ZooKeeper zk = null;
    private static Stat stat = new Stat();
    public static void main(String[] args) throws Exception {
    	String path = "/zk-book";
    	zk = new ZooKeeper("localhost:2181", 5000, new GetDataAPISyncUsage());
        connectedSemaphore.await();
        /**
         * 新增节点并给节点赋值
         */
        zk.create( path, "123".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL );
        /**
         * 获取节点数据，传入旧的stat，会被服务端响应的新的stat替换
         */
        System.out.println(new String(zk.getData( path, true, stat )));
        System.out.println(stat.getCzxid()+","+stat.getMzxid()+","+stat.getVersion());
        /**
         * 虽然节点的值没有改版，但是版本号改变了，依然会触发process事件
         */
        zk.setData( path, "123".getBytes(), -1 );
        nodeDataChangedSemaphore.await();
    }
    @Override
    public void process(WatchedEvent event) {
        if (KeeperState.SyncConnected == event.getState()) {
            if (EventType.None == event.getType() && null == event.getPath()) {
                connectedSemaphore.countDown();
            } else if (event.getType() == EventType.NodeDataChanged) {
                try {
                    System.out.println(new String(zk.getData( event.getPath(), true, stat)));
                    System.out.println(stat.getCzxid()+","+ stat.getMzxid()+","+ stat.getVersion());
                    nodeDataChangedSemaphore.countDown();
                } catch (Exception e) {}
            }
        }
    }
}
```

#更新数据
```
// ZooKeeper API 更新节点数据内容，使用同步(sync)接口。
public class SetDataAPISyncUsage implements Watcher {
    private static CountDownLatch connectedSemaphore = new CountDownLatch(1);
    public static void main(String[] args) throws Exception {
    	String path = "/zk-book";
        ZooKeeper zk = new ZooKeeper("localhost:2181", 5000, new SetDataAPISyncUsage());
    	connectedSemaphore.await();
        zk.create( path, "123".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL );
        /**
         * version：-1，代表不需要根据版本号更新
         */
        Stat stat = zk.setData( path, "456".getBytes(), -1 );
        System.out.println(stat.getCzxid()+","+
                           stat.getMzxid()+","+
                           stat.getVersion());
        /**
         * 根据上一次更新的版本号更新，成功
         */
        Stat stat2 = zk.setData( path, "456".getBytes(), stat.getVersion());
        System.out.println(stat2.getCzxid()+","+
                           stat2.getMzxid()+","+
                           stat2.getVersion());
        /**
         * 根据上上次旧的版本跟新，失败抛异常
         */
        try {
            zk.setData( path, "456".getBytes(), stat.getVersion() );
        } catch ( KeeperException e ) {
            System.out.println("Error: " + e.code() + "," + e.getMessage());
        }
    }
    @Override
    public void process(WatchedEvent event) {
        if (KeeperState.SyncConnected == event.getState()) {
            if (EventType.None == event.getType() && null == event.getPath()) {
                connectedSemaphore.countDown();
            }
        }
    }
}
异步更新方式在项目代码示例中，不贴出来了。
```

### 检测节点是否存在

```
// ZooKeeper API 判断节点是否存在，使用同步(sync)接口。
public class ExistAPISyncUsage implements Watcher {
    private static CountDownLatch connectedSemaphore = new CountDownLatch(1);
    private static CountDownLatch lastSemaphore = new CountDownLatch(1);
    private static ZooKeeper zk;
    private static String path = "/zk-book";
    public static void main(String[] args) throws Exception {
    	zk = new ZooKeeper("localhost:2181", 5000, new ExistAPISyncUsage());
    	connectedSemaphore.await();
        /**
         * 通过exists接口检测是否存在指定节点，同事注册一个Watcher
         */
    	zk.exists( path, true );
        /**
         * 创建节点/zk-book，服务器会向客户端发送事件通知：NodeCreated
         * 客户端收到通知后，继续调用exists接口，注册Watcher
         */
    	zk.create( path, "".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT );
        /**
         * 更新节点数据，服务器会向客户端发送事件通知：NodeDataChanged
         * 客户端收到通知后，继续调用exists接口，注册Watcher
         */
    	zk.setData( path, "123".getBytes(), -1 );
        /**
         * 删除节点/zk-book
         * 客户端会收到服务端的事件通知：NodeDeleted
         */
    	zk.delete( path, -1 );
        lastSemaphore.await();
    }
    @Override
    public void process(WatchedEvent event) {
        try {
            if (KeeperState.SyncConnected == event.getState()) {
                if (EventType.None == event.getType() && null == event.getPath()) {
                    connectedSemaphore.countDown();
                } else if (EventType.NodeCreated == event.getType()) {
                    System.out.println("Node(" + event.getPath() + ")Created");
                    zk.exists( event.getPath(), true );
                } else if (EventType.NodeDeleted == event.getType()) {
                    System.out.println("Node(" + event.getPath() + ")Deleted");
                    zk.exists( event.getPath(), true );
                    System.out.println("Last semaphore");
                    lastSemaphore.countDown();
                } else if (EventType.NodeDataChanged == event.getType()) {
                    System.out.println("Node(" + event.getPath() + ")DataChanged");
                    zk.exists( event.getPath(), true );
                }
            }
        } catch (Exception e) {}
    }
}
异步方式我没有再写出来，自行查看API吧。
```