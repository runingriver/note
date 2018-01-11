---
title: 【ZooKeeper】基于Curator的开发(四)
date: 2017/05/12 11:12:22
toc: false
list_number: false
categories:
- zookeeper
tags:
- zookeeper
---

# Curator开发介绍
当前基于zookeeper的client开发有zookeeper提供的原生jar包，也有第三方ZKClient jar包，但是更多的是使用Curator的jar包。

# 封装Curator
通常简单使用Zookeeper的场景,直接使用Curator即可,无需进行更多的封装。但是，在较复杂的场景下，对Curator进行进一步封装显得很有必要。
一方面，不同环境使用不同的zk集群，或者spring下使用，针对业务需要的封装以方便使用；另一方面，将业务逻辑和zookeeper的执行逻辑分开！
下面主要描述如何结合spring封装Curator，封装类为`ZkHelper`。

## 基本配置
1. 引入下面两个jar包：
```
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.9</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>4.0.0</version>
</dependency>
```

2. 引入配置，配合spring根据不同环境，使用不同的zookeeper的连接地址：
可以通过下面两种方式：
```
//1. 属性注入
@Value("${zookeeper.address}")
String connectionUrl;
//2. 后构造函数中初始化
@PostConstruct
public void init() {
    connectionString = FileProperty.getPropertyValues("zookeeper.address");
}
```
## ZkHelper封装示例
请勿用于实际环境，为缩减篇幅，忽略了日志和异常处理等情况，删除了一些封装方法：
- - - 
代码较长，置于文尾….
- - -
上面封装仅做参考，目的：
1. 统一管理zk对象；
2. 统一开启关闭zk；
3. 封装PathChild和NodeCache的逻辑。
4. 进一步封装Curator操作zk数据树的方法。
所有的**目的**就是做到对业务代码无侵入，与业务逻辑分开。


## 使用示例
以设置某个路径下的所有节点变化的监听为例：
1. 获取实例： `ZkHelper.ZKClient zkClient = zkHelper.getDefaultZKClient();`
2. 设置监听：`zkClient.setPathChildListener(“zk_path…”, nodeListener());`
实现具体业务逻辑
```
private PathChildrenCacheListener nodeListener() {
    return new PathChildrenCacheListener() {
        @Override
        public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
            if (event == null || event.getData() == null || event.getData().getPath() == null) {
                /*do something*/
                return;
            }
            switch (event.getType()) {
                case CHILD_ADDED: /*do something*/ break;
                case CHILD_REMOVED: /*do something*/ break;
                case CONNECTION_RECONNECTED: /*do something*/ break;
                case CONNECTION_LOST: /*do something*/ break;
                default:break;
            }
        }
    };
}
```
ok，所有操作完成，但是如果你还想动态监听：可以关闭`zkClient.closePathChildrenCache(“zk_listener_path...”);`，然后重新设置！

这里，也许在设置`NodeCacheListener`时，需要区别不同的listener，可以采用下面方法进一步封装：

```
public static abstract class NodeDataListener implements NodeCacheListener {
    private String nodePath;
    public NodeDataListener(String nodePath) {
        this.nodePath = nodePath;
    }
    public String getNodePath() {
        return nodePath;
    }
}
```
注意的点：
1. curator实现了重复监听，但是系统启动的时候会默认触发一下监听的事件！
2. 临时节点删除的通知的时间间隔,依赖于client和zk server配置的session timeout和connection timeout时间!

# 示例代码
```
@Component
public class ZkHelper {
    //用于同时多个zk集群的使用和操作
    private volatile Map<String, ZKClient> clientMap = Maps.newConcurrentMap();
    @Value("${zookeeper.address}")
    String connectionUrl;
    @PreDestroy
    public void closeZkClient() {
        for (ZKClient zkClient : clientMap.values()) {
            try {
                zkClient.close();
            } catch (Exception e) {
                //...
            }
        }
    }
    /**
     * 获取默认连接的zk对象
     */
    public ZKClient getDefaultZKClient() {
        ZKClient zkClient = getZKClient(connectionUrl);
        return zkClient;
    }
    /**
     * 根据连接地址来实例化并使用zk
     */
    public ZKClient getZKClient(String address) {
        Preconditions.checkArgument(StringUtils.isNotBlank(address), "address is illegal.");
        if (clientMap.containsKey(address)) {
            return clientMap.get(address);
        }
        synchronized (this) {
            if (clientMap.containsKey(address)) {
                return clientMap.get(address);
            } else {
                ZKClient zkClient = new ZKClient(address);
                clientMap.put(address, zkClient);
                return zkClient;
            }
        }
    }
    /**
     * 具体对业务逻辑的封装
     */
    public static class ZKClient {
        private String connectString;
        private CuratorFramework client;
        private static final int TIMEOUT = 5000;
        //注册,仅用作关闭!
        private Map<String, NodeCache> nodeCacheMap = Maps.newConcurrentMap();
        private Map<String, PathChildrenCache> pathChildrenCacheMap = Maps.newConcurrentMap();
        public ZKClient(String address) {
            this.connectString = address;
            client = CuratorFrameworkFactory.newClient(address, TIMEOUT, TIMEOUT, new ExponentialBackoffRetry(TIMEOUT, 3));
            client.start();
            this.setConnWatcher(new ConnectionStateListener() {
                @Override
                public void stateChanged(CuratorFramework client, ConnectionState newState) {
                    //...
                }
            });
        }
        public CuratorFramework getZkCuratorClient() {
            return client;
        }
        public boolean createNode(String path, CreateMode mode, byte[] data) {
            try {
                ZKPaths.mkdirs(client.getZookeeperClient().getZooKeeper(), path, false);
                client.create().creatingParentsIfNeeded().withMode(mode).forPath(path, data);
                return true;
            } catch (Exception e) {
                //...
            }
            return false;
        }
        public boolean createNode(String path, CreateMode mode) {
            //...
        }
        public String getData(String path) {
            //...
        }
        public boolean setData(String path, byte[] data) {
            try {
                if (!checkNodeExisted(path)) {
                    ZKPaths.mkdirs(client.getZookeeperClient().getZooKeeper(), path, true);
                }
                client.setData().forPath(path, data);
            } catch (Exception e) {
                return false;
            }
            return true;
        }
        /**
         * 给指定路径节点设置一个监控,监控zk节点的变化,只监控一次
         */
        public void setNodeWatcher(String path, CuratorWatcher curatorWatcher) {
            Preconditions.checkArgument(checkNodeExisted(path), "the node does not exist.");
            try {
                client.getData().usingWatcher(curatorWatcher).forPath(path);
            } catch (Exception e) {
                //...
            }
        }
        public boolean deleteNode(String path) {
            if (!checkNodeExisted(path)) {
                return false;
            }
            try {
                client.delete().forPath(path);
            } catch (Exception e) {
                //...
                return false;
            }
            return true;
        }
        /**
         * 给指定路径节点的所有子节点设置一个监控,只监控一次
         */
        public void setChildrenWatcher(String path, CuratorWatcher curatorWatcher) {
            Preconditions.checkArgument(checkNodeExisted(path), "the node does not exist.");
            try {
                client.getChildren().usingWatcher(curatorWatcher).forPath(path);
            } catch (Exception e) {
                //...
            }
        }
        public boolean checkNodeExisted(String path) {
            if (StringUtils.isBlank(path)) {
                return false;
            }
            Stat stat = null;
            try {
                stat = client.checkExists().forPath(path);
            } catch (Exception e) {
                //...
            }
            return stat == null ? false : true;
        }
        public boolean checkAndCreatePath(String path) {
            if (checkNodeExisted(path)) {
                return true;
            }
            try {
                ZKPaths.mkdirs(client.getZookeeperClient().getZooKeeper(), path, true);
            } catch (Exception e) {
                //...
                return false;
            }
            return true;
        }
        /**
         * 给该连接设置一个监控,连接每发生一次变化,都会回调,重复监听
         * Curator具有连接中断重连机制!
         */
        public void setConnWatcher(ConnectionStateListener connWatcher) {
            client.getConnectionStateListenable().addListener(connWatcher);
        }
        /**
         * 监听Node节点中数据改变事件,自动重复监听
         */
        public boolean setNodeCacheListener(String path, NodeCacheListener listener) {
            if (!checkNodeExisted(path)) {
                return false;
            }
            NodeCache nodeCache = new NodeCache(client, path);
            nodeCacheMap.put(path, nodeCache);
            try {
                nodeCache.start();
                nodeCache.getListenable().addListener(listener);
            } catch (Exception e) {
                return false;
            }
            return true;
        }
        /**
         * 关闭指定path的NodeCache
         */
        public void closeNodeCache(String path) {
            if (nodeCacheMap.containsKey(path)) {
                try {
                    nodeCacheMap.get(path).close();
                } catch (IOException e) {
                    //...
                }
            }
        }
        public void closePathChildrenCache(String path) {
            if (pathChildrenCacheMap.containsKey(path)) {
                PathChildrenCache childrenCache = pathChildrenCacheMap.get(path);
                closePathChildrenCache(childrenCache);
            }
        }
        private void closePathChildrenCache(PathChildrenCache childrenCache) {
            try {
                childrenCache.close();
                //清空缓存值
                childrenCache.clear();
                //清除map中的listener对象
                childrenCache.getListenable().clear();
            } catch (IOException e) {
                //...
            }
        }
        /**
         * 关闭ZKClient对象
         */
        public void close() {
            for (NodeCache nodeCache : nodeCacheMap.values()) {
                try {
                    nodeCache.close();
                } catch (IOException e) {
                    //...
                }
            }
            for (PathChildrenCache childrenCache : pathChildrenCacheMap.values()) {
                closePathChildrenCache(childrenCache);
            }
            try {
                client.close();
            } catch (Exception e) {
                //...
            }
        }
        public boolean setPathChildListener(String path, PathChildrenCacheListener listener) {
            if (!checkNodeExisted(path)) {
                //...
                return false;
            }
            PathChildrenCache childrenCache = new PathChildrenCache(this.getZkCuratorClient(), path, false);
            pathChildrenCacheMap.put(path, childrenCache);
            try {
                childrenCache.start();
                childrenCache.getListenable().addListener(listener);
            } catch (Exception e) {
                //...
                return false;
            }
            return true;
        }
    }
}
```

## 参考
1. https://zookeeper.apache.org/doc/trunk/zookeeperAdmin.html
2. http://www.cnblogs.com/LiZhiW/tag/ZooKeeper/