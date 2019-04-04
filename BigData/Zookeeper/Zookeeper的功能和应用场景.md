# Zookeeper相关知识

## 1. Zookeeper概念

Zookeeper是一个分布式协调服务，为用户的分布式应用程序提供协调服务

1. 其本身就是一个分布式程序(只要有半数以上节点存活，zk就能正常工作)
2. Zookeeper所提供的服务：主从协调、服务器节点动态上下线、统一配置管理、分布式共享锁、统一名称服务等
3. Zookeeper在底层只提供了两个功能：管理(存储，读取)用户程序提交的数据；并为用户程序提供数据节点监听服务。

export ： 扩充环境变量到子shell;   source : 扩充环境变量到其父shell

## 2. Zookeeper结构和命令

### Zookeeper特性

1. 一个leader，多个follower组成的集群；
2. 全局数据一致。每个server保存一份相同的数据副本，client无论连接到哪个server，数据都是一致的；
3. 分布式读写。更新请求转发，由leader实施；
4. 更新请求顺序执行，来自同一个client的更新请求按其发送顺序依次执行；
5. 数据更新原子性，一次数据更新要么成功，要么失败；
6. 实时性，在一定时间范围内，client能读到最新数据。

### Zookeeper数据结构

1. 层次化的目录结构；
2. 每个节点在Zookeeper中叫做znode，并且其中有一个唯一的路径标识；
3. 节点Znode可以包含数据和子节点(Ephemeral类型的节点不能有子节点)；
4. 客户端应用可以在节点上设置监听器；

![1554349085389](.\image\%5CUsers%5CAdministrator%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5C1554349085389.png)

### 节点类型

Znode有两种类型：

1. 短暂(ephemeral)，断开连接后自己删除 

2. 持久(persistent)，断开连接后不会删除

Zonde有四种形式的目录节点（默认是persistent）

1. Persistent 
2. persistent_sequential(持久序列/test00000019)
3. ephemeral
4. ephemeral_sequential

创建Zonde时设置顺序标识，Znode名称后会附加一个值，顺序好是一个单调递增的计数器，由父节点维护。

在分布式系统中，顺序号可以被用于为所有的事件进行全局排序，这样客户端可以通过顺序号推断事件的顺序。

## 3. Zookeeper的应用

### 分布式锁的实现

算法实现流程：

1. 在Zookeeper指定节点（locks）下创建临时顺序节点node_n；
2. 获取locks下所有子节点children
3. 对子节点自增序号从小到大进行排序
4. 判断本节点是不是第一个子节点，若是，则获取锁；若不是，监听比该节点小的那个节点的删除事件
5. 监听事件生效，则回到第二步重新进行判断，直到获取到锁

[分布式锁的实现]: https://blog.csdn.net/zhailuxu/article/details/80658016	"分布式锁的实现"

```Java
public class ZookeeperDistributedLock {

    private ZooKeeper zk;

    private String root = "/locks";

    private String lockName;

    private ThreadLocal<String> nodeId = new ThreadLocal<>();

    private CountDownLatch connectedSignal = new CountDownLatch(1);

    private final static int sessionTimeout = 3000;

    private final static byte[] data = new byte[0];

    public ZookeeperDistributedLock(String config, String lockName){
        this.lockName = lockName;

        try {
            zk = new ZooKeeper(config, sessionTimeout, new Watcher() {
                @Override
                public void process(WatchedEvent watchedEvent) {
                    if (watchedEvent.getState() == Event.KeeperState.SyncConnected){
                        connectedSignal.countDown();
                    }
                }
            });

            connectedSignal.await();
            Stat stat = zk.exists(root, false);
            if (null == stat){
                zk.create(root, data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            }

        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
    }

    class LockWatcher implements  Watcher{

        private CountDownLatch latch = null;

        public LockWatcher(CountDownLatch latch){
            this.latch = latch;
        }

        @Override
        public void process(WatchedEvent watchedEvent) {
            if (watchedEvent.getType() == Event.EventType.NodeDeleted){
                latch.countDown();
            }
        }
    }

    public void lock(){

        try {
            String myNode = zk.create(root + "/" + lockName, data, ZooDefs.Ids.OPEN_ACL_UNSAFE,
                    CreateMode.EPHEMERAL);
            System.out.println(Thread.currentThread().getName() + myNode + " 节点创建");
            List<String> subNodes = zk.getChildren(root, false);
            List<String> sortedNodes = subNodes.stream()
                                                .sorted().collect(Collectors.toList());
            if (myNode.equals(root + "/" + sortedNodes.get(0))){
                System.out.println(Thread.currentThread().getName() + "线程获得到了锁 ： " + myNode);
                this.nodeId.set(myNode);
                return;
            }

            CountDownLatch latch = new CountDownLatch(1);
            String preNode = myNode.substring(myNode.lastIndexOf("/") + 1);
            String waitNode = sortedNodes.get(Collections.binarySearch(sortedNodes, preNode) - 1);

            Stat stat = zk.exists(waitNode, new LockWatcher(latch));
            if (stat != null){
                System.out.println(Thread.currentThread().getName() + "等待 " + root + "/" + preNode + "释放锁");
                latch.await(); //等待
                nodeId.set(myNode);
                latch = null;
            }

        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }


    public void unlock(){
        System.out.println(Thread.currentThread().getName() + nodeId.get() + "锁释放了");
            try {
                if (nodeId != null){
                    zk.delete(nodeId.get(), -1);
                }
                nodeId.remove();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (KeeperException e) {
                e.printStackTrace();
            }
    }
}

```

