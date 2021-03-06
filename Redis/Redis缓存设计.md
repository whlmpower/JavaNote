# 缓存设计
## 1.1缓存的收益和成本
1.收益：     
加速读写    
降低后端负载    
2.成本：    
数据不一致性     
代码维护成本     
运维成本      
3.缓存使用场景    
开销大量计算       
加速请求响应     

## 1.2缓存更新策略    
1.LRU/LFU/FIFO 算法剔除    
使用场景：maxmemory-policy 这个配置作为内存最大值后对于数据的剔除      
一致性：最差劲     
维护成本：只需要配置maxmemory和对应的策略即可      
2.超时剔除      
使用场景：设置缓存超时时间     
一致性：一段时间窗口内存在一致性问题     
维护成本：只需要设置expire过期时间      
3.主动更新    
使用场景：需要在获取真实数据后，立即进行缓存更新，可以利用消息系统或者其他方式通知缓存更新。     
一致性：一致性最高   
维护成本：开发者需要自己来完成更细     
4.最佳实践     
低一致性业务建议配置最大内存和淘汰策略的方式使用     
高一致性业务可以结合使用超时剔除和主动更新，这样即使主动更新出了问题，也能保证数据过期时间后删除脏数据      

## 1.3缓存粒度控制    
通用性    
空间占用    
代码维护     

## 1.4穿透优化
缓存穿透是指查询一个根本不存在的数据，缓存层和存储层都不会命中。    
解决缓存穿透问题：     
1.缓存空对象    
缓存空对象会有两个问题：第一空值做了缓存，意味着缓存层中存了更多的键，需要更多的内存空间，比较有效的办法是设置一个较短的过期时间，让其自动删除。第二，缓存层和存储层的数据会有一段时间窗口的不一致，可以使用消息系统或其他方式清除掉缓存层中的空对象。     
```Java
String get (String key){
    String cacheValue = cache.get(key);
    if(StringUtils.isBlank(cacheValue)){
        String storageValue = storage.get(key);
        cache.set(key, storageValue);
        if(storageValue == null){
            cache.expire(key, 60 * 5);
        }
        return storageValue;
    }else {
        return cacheValue;
    }
}
``` 
2.布隆过滤器拦截    
在访问缓存层和存储层之前，将存在的key用布隆过滤器提前保存起来，做第一层拦截。将所有推荐数据的用户做成布隆过滤器，如果布隆过滤器认为该用户id 不存在，就不会访问存储层，在一定程度上保护了存储层。      


## 1.5无底洞优化   
键值数据库由于通常采用哈希函数将key映射到各个节点上，批量操作通常需要从不同的节点上获取，相对比单机批量操作只涉及一次网络操作，分布式批量操作会涉及多次网络时间。    
如何高效地在分布式缓存中批量操作是一个难点。    
常见的IO优化思路：  
命令本身的优化，例如优化SQL语句等    
减少网络通信的次数     
降低接入成本，例如客户端使用长连 连接池、NIO等     
Redis批量获取n个字符串，有三种实现办法：    
客户端n次get: n 次网络 + n 次get 命令本身     
客户端1次pipeline get： 1 次网络 + n 次 get命令    
客户端1次mget： 1次网络加一次mget命令本身      
结合Redis Cluster的一些特性对四种分布式的批量操作方式进行说明     
1.串行命令   
2.串行IO     
Redis Cluster使用CRC16算法计算出散列值，在取对16383的余数就可以算出slot值，同时提到过
Smart客户端会保存slot和节点的对应关系，有了这两个数据可以将属于同一个节点的key进行归档，得到每一个节点的key的子列表，之后对每个节点执行mget或者Pipeline操作，它的操作时间是node次网络时间+n次命令时间，网络次数是node的个数     
3.并行IO    
此方案试讲方案2的最后一步改成多线程执行，网络次数虽然还是节点的个数，但是使用多线程网络时间变成O(1),操作时间为： 
max_slow(node网络时间) + n 次命令时间     
4.hash_tag 实现
Redis Cluster的hash_tag功能，可以将多个key强制分配到一个节点上，它的操作时间=1次网络时间+n次命令时间    

## 1.6雪崩优化   
缓存层承载着大量的请求，有效地保护了存储层，如果缓存层由于某些原因不能提供服务，所有的服务请求都会达到存储层，存储层的调用量会暴增，造成存储层也级联宕机的情况。    
预防和解决缓存雪崩的问题     
1.保证缓存层的服务高可用，Redis Sentinel 和 Redis Cluster 都实现了高可用    
2.依赖隔离组件为后端限流并降级。 对重要的资源进行分离，让每种资源都单独运行在自己的线程池中----Hystrix   
3.提前演练     

## 1.7热点key 重建优化    
在缓存失效瞬间，有大量的线程来重建缓存，造成后端负载加大，甚至可能让应用崩溃。指定如下目标：    
减少缓存重建次数       
数据尽可能一致     
较少的潜在风险    
1.互斥锁     
只允许一个线程重建缓存，其他线程等待重建缓存的线程执行完，重新从缓存获取数据即可   
使用Redis的setnx命令实现上述功能     
```Java
String get(String key){
    String value = redis.get(key);
    if(value == null){
        String mutexKey = "mutext:key" + key;
        if(redis.set(mutexKey, "1", "ex 180", "nx")){
            value = db.get(key);
            redis.setex(key, timeout, value);
            redis.delete(mutexKey);
        }
        else{
            Thread.sleep(50);
            get(key);
        }
    }
    return value;
}
``` 
(1)Redis 获取数据，如果值不为空，则直接返回值     
(2.1) 如果set(nx, ex)结果为true，说明此时没有其他线程重建缓存，那么当前的线程完成缓存的建立。   
(2.2) 如果set(nx, ex)结果为false，说明此时已经有其他线程正在执行缓存构建，当前线程将休息指定时间后，重新执行函数，直到获取到数据。     
2.永不过期
为每一个value设置一个逻辑过期时间，当发现超过逻辑过期时间后，会启用单独的线程去构建缓存。    

优缺点比较：    
1.互斥锁，构建缓存出现问题或者时间较长，可能会出现死锁或者线程池阻塞的风险，但是这种方法能较好地降低后端存储负载，并在一致性上做的比较好。    
2.永远不过期，这种方案由于没有设置真正的过期时间，实际上已经不存在热点key产生的一系列危害，但是会存在数据不一致的情况，同时代码复杂度会增加。     

参考书籍： Redis开发与运维



