---
layout: post
title: "ZooKeeper(二)编程例子"
tagline: "ZooKeeper(二)编程例子"
description: "ZooKeeper(二)编程例子"
tags: [zookeeper, 分布式]
---
{% include JB/setup %}

###ZooKeeper的java例子

    public class ZookeepTest {
        private static Map<Watcher.Event.EventType,IFunc> map;
        public static void main(String[] args) throws IOException, KeeperException, InterruptedException {
            map = Maps.newHashMap();
            map.put(Watcher.Event.EventType.NodeChildrenChanged,new DataChangedFunc());
            map.put(Watcher.Event.EventType.NodeCreated,new NodeCreateFunc());
            ZooKeeper zk = new ZooKeeper("127.0.0.1:2181",50*1000,new Watcher() {

                @Override
                public void process(WatchedEvent watchedEvent) {

                    System.out.println("回调watcher实例： 路径" + watchedEvent.getPath() + " 类型："
                            + watchedEvent.getType());


                }

            });
            System.out.println("---------------------");

            // 创建一个节点mztt，数据是mydata,不进行ACL权限控制，节点为永久性的(即客户端shutdown了也不会消失)

            zk.exists("/mztt3", true);

            zk.create("/mztt3", "mydata".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,

                    CreateMode.PERSISTENT);

            System.out.println("---------------------");

            // 在mztt下面创建一个childone znode,数据为childone,不进行ACL权限控制，节点为永久性的

            zk.exists("/mztt/childone", true);

            zk.create("/mztt/childone", "childone".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,

                    CreateMode.PERSISTENT);

            System.out.println("---------------------");

            // 删除/mztt/childone这个节点，第二个参数为版本，－1的话直接删除，无视版本

            zk.exists("/mztt/childone", true);

            zk.delete("/mztt/childone", -1);

            System.out.println("---------------------");

            zk.exists("/mztt", true);

            zk.delete("/mztt", -1);

            System.out.println("---------------------");

            // 关闭session

            zk.close();

            //3类事件触发wather后就不再作用，也就是所谓的（一次作用），
            // 但是，如何永久监听呢？这需要我们再程序逻辑上进行控制，网上有更好的办法，但是，在简单的应用中，
            // 可以再wather方法里面再设置监听，这个方法很笨，但是，很有效，达到了预期的效果。

        }
    }


* exist方法只能一次注册一次监听
* 如何做到永久监听

###Exist()方法做到永久监听的解决办法

    private synchronized void initNodes(List<String> nodes) {
        // 根据zk节点，判断是否需要处理
    }

    private void syncNodes() {
        try {
            List<String> nodes = zookeeper.getChildren(ArbitrateConstants.NODE_NID_ROOT, new AsyncWatcher() {

                public void asyncProcess(WatchedEvent event) {
                    syncNodes();// 继续关注node节点变化
                }
            });

            initNodes(nodes);
        } catch (KeeperException e) {
            syncNodes();
        } catch (InterruptedException e) {
            // ignore
        }
    }