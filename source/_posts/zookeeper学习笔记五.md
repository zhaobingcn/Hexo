---
title: zookeeper学习笔记五
date: 2016-07-29 12:23:05
tags:
     - Zookeeper
categories: Zookeeper
---

Zookeeper提供了ALC的权限控制机制，简单来说就是通过设置Zookeeper服务器上数据节点的ACL，来控制客户端对该数据节点的访问权限。Zookeeper提供了多种权限控制模式，这里选择digest来了解下API的使用方法。

## 创建带权限信息的节点

```
//使用含权限信息的ZooKeeper会话创建数据节点
public class ZNodeForFoo implements Watcher{
    private static CountDownLatch connectedSemaphore = new CountDownLatch(1);
    public static void main(String[] args) throws Exception {
        String path = "/zk-book-auth_test";
        ZooKeeper zookeeper = new ZooKeeper("localhost:2181", 50000, new ZNodeForFoo());
		connectedSemaphore.await();
        //添加带权限信息的节点
        zookeeper.addAuthInfo("digest", "foo:true".getBytes());
        zookeeper.create( path, "init".getBytes(), Ids.CREATOR_ALL_ACL, CreateMode.EPHEMERAL );
    }
    @Override
    public void process(WatchedEvent watchedEvent) {
        if (Event.KeeperState.SyncConnected == watchedEvent.getState()) {
            connectedSemaphore.countDown();
        }
    }
}
```

## 使用不同的权限信息访问节点

```
//使用不同的权限信息的ZooKeeper会话访问含权限信息的数据节点
public class GetFooNodeByAuth implements Watcher{
    private static CountDownLatch noAuthSemaphore = new CountDownLatch(1);
    private static CountDownLatch wrongAuthSemaphore = new CountDownLatch(1);
    private static CountDownLatch rightAuthSemaphore = new CountDownLatch(1);
    public static void main(String[] args) throws Exception {
        String path = "/zk-book-auth_test";
        try{
            ZooKeeper noAuthZK = new ZooKeeper("localhost:2181", 50000, new GetFooNodeByAuth());
            noAuthSemaphore.await();
            //使用不包含权限信息的客户端访问节点，抛出异常
            noAuthZK.getData( path, false, null );
        }catch(Exception e){
            e.printStackTrace();
        }
        try{
            ZooKeeper wrongAuthZK = new ZooKeeper("localhost:2181", 50000, new GetFooNodeByAuth());
            wrongAuthSemaphore.await();
            wrongAuthZK.addAuthInfo("digest", "bar:true".getBytes());
            //使用错误的权限信息访问节点，抛出异常
            wrongAuthZK.getData( path, false, null );
        }catch (Exception e){
            e.printStackTrace();
        }
        ZooKeeper rightAuthZK = new ZooKeeper("localhost:2181", 50000, new GetFooNodeByAuth());
        rightAuthSemaphore.await();
        rightAuthZK.addAuthInfo("digest", "foo:true".getBytes());
        //使用正确的权限信息获取节点
        System.out.println(new String(rightAuthZK.getData( path, false, null )));
    }
    @Override
    public void process(WatchedEvent watchedEvent) {
        if (Event.KeeperState.SyncConnected == watchedEvent.getState()) {
            if(noAuthSemaphore.getCount() > 0){
                noAuthSemaphore.countDown();
                return;
            }
            if(wrongAuthSemaphore.getCount() > 0){
                wrongAuthSemaphore.countDown();
                return;
            }
            if(rightAuthSemaphore.getCount() > 0){
                rightAuthSemaphore.countDown();
                return;
            }
        }
    }
}
```

## 删除节点的权限控制

```
//删除节点的权限控制
public class DeleteNodeByAuth implements Watcher{
    private static CountDownLatch createSemaphore = new CountDownLatch(1);
    private static CountDownLatch deleteChildNoAuthSemaphore = new CountDownLatch(1);
    private static CountDownLatch deleteChildAuthSemaphore = new CountDownLatch(1);
    private static CountDownLatch deleteNoAuthSemaphore = new CountDownLatch(1);
    public static void main(String[] args) throws Exception {
        String path = "/zk-book-auth_test";
        String pathChild = "/zk-book-auth_test/child";
        ZooKeeper createZK = new ZooKeeper("localhost:2181",5000,new DeleteNodeByAuth());
        createSemaphore.await();
        createZK.addAuthInfo("digest", "foo:true".getBytes());
        createZK.create( pathChild, "initChild".getBytes(), Ids.CREATOR_ALL_ACL, CreateMode.PERSISTENT );

        try {
            ZooKeeper deleteChildNoAuthZK = new ZooKeeper("localhost:2181",50000,new DeleteNodeByAuth());
            deleteChildNoAuthSemaphore.await();
            deleteChildNoAuthZK.delete( pathChild, -1 );
        } catch ( Exception e ) {
           System.out.println( "删除节点失败: " + e.getMessage() );
        }

        ZooKeeper deleteChildAuthZK = new ZooKeeper("localhost:2181",50000,new DeleteNodeByAuth());
        deleteChildAuthSemaphore.await();
        deleteChildAuthZK.addAuthInfo("digest", "foo:true".getBytes());
        deleteChildAuthZK.delete( pathChild, -1 );
        System.out.println( "成功删除节点：" + pathChild );
        /**
         * 删除权限作用的范围是子节点，所有不包含权限信息的客户端可以删除/zk-book-auth_test节点
         */
        ZooKeeper deleteNoAuthZK = new ZooKeeper("localhost:2181", 50000, new DeleteNodeByAuth());
        deleteNoAuthSemaphore.await();
        deleteNoAuthZK.delete( path, -1 );
        System.out.println( "成功删除节点：" + path );
    }
    @Override
    public void process(WatchedEvent watchedEvent) {
        if (Event.KeeperState.SyncConnected == watchedEvent.getState()) {
            if(createSemaphore.getCount() > 0){
                createSemaphore.countDown();
                return;
            }
            if(deleteChildNoAuthSemaphore.getCount() > 0){
                deleteChildNoAuthSemaphore.countDown();
                return;
            }
            if(deleteChildAuthSemaphore.getCount() > 0){
                deleteChildAuthSemaphore.countDown();
                return;
            }
            if(deleteNoAuthSemaphore.getCount() > 0){
                deleteNoAuthSemaphore.countDown();
                return;
            }
        }
    }
}
```