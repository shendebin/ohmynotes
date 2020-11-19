
# 深入理解MyBatis缓存
### 简介
缓存是MyBatis中非常重要的特性。在应用程序和数据库都是单节点的情况下，合理使用缓存能够减少数据库IO，显著提升系统性能。但是在分布式环境下，如果使用不当，则可能会带来数据一致性问题。MyBatis提供了一级缓存和二级缓存，其中一级缓存基于SqlSession实现，而二级缓存基于Mapper实现。下面我们就来学习一下MyBatis缓存的使用，并分析MyBatis缓存的实现原理。
### MyBatis缓存实现类
MyBatis通过Cache接口定义缓存对象的行为，Cache接口代码如下：

```java
public interface Cache {

  /**
   * @return The identifier of this cache
   */
  String getId();

  /**
   * @param key Can be any object but usually it is a {@link CacheKey}
   * @param value The result of a select.
   */
  void putObject(Object key, Object value);

  /**
   * @param key The key
   * @return The object stored in the cache.
   */
  Object getObject(Object key);

  /**
   * As of 3.3.0 this method is only called during a rollback 
   * for any previous value that was missing in the cache.
   * This lets any blocking cache to release the lock that 
   * may have previously put on the key.
   * A blocking cache puts a lock when a value is null 
   * and releases it when the value is back again.
   * This way other threads will wait for the value to be 
   * available instead of hitting the database.
   *
   * 
   * @param key The key
   * @return Not used
   */
  Object removeObject(Object key);

  /**
   * Clears this cache instance
   */  
  void clear();

  /**
   * Optional. This method is not called by the core.
   * 
   * @return The number of elements stored in the cache (not its capacity).
   */
  int getSize();
  
  /** 
   * Optional. As of 3.2.6 this method is no longer called by the core.
   *  
   * Any locking needed by the cache must be provided internally by the cache provider.
   * 
   * @return A ReadWriteLock 
   */
  ReadWriteLock getReadWriteLock();

}

```

Cache接口的默认实现类是PerpetualCache，通过一个HashMap实例存放缓存对象。需要注意的是，PerpetualCache类重写了Object类的equals()方法，当两个缓存对象的Id相同时，即认为缓存对象相同。另外，PerpetualCache类还重写了Object类的hashCode()方法，仅以缓存对象的Id作为因子生成hashCode。除了基础的PerpetualCache类之外，MyBatis中为了对PerpetualCache类的功能进行增强，提供了一些缓存的装饰器类，使用了装饰器模式。

![Mybatis缓存装饰器类](/resources/ca1.jpg "图片Title")

这些缓存装饰器类功能如下。
- BlockingCache：阻塞版本的缓存装饰器，能够保证同一时间只有一个线程到缓存中查找指定的Key对应的数据。
- FifoCache：先入先出缓存装饰器，FifoCache内部有一个维护具有长度限制的Key键值链表（LinkedList实例）和一个被装饰的缓存对象，Key值链表主要是维护Key的FIFO顺序，而缓存存储和获取则交给被装饰的缓存对象来完成。
- LoggingCache：为缓存增加日志输出功能，记录缓存的请求次数和命中次数，通过日志输出缓存命中率。
- LruCache：最近最少使用的缓存装饰器，当缓存容量满了之后，使用LRU算法淘汰最近最少使用的Key和Value。
- ScheduledCache：自动刷新缓存装饰器，当操作缓存对象时，如果当前时间与上次清空缓存的时间间隔大于指定的时间间隔，则清空缓存。清空缓存的动作由getObject()、putObject()、removeObject()等方法触发。
- SerializedCache：序列化缓存装饰器，向缓存中添加对象时，对添加的对象进行序列化处理，从缓存中取出对象时，进行反序列化处理。
- SoftCache：软引用缓存装饰器，SoftCache内部维护了一个缓存对象的强引用队列和软引用队列，缓存以软引用的方式添加到缓存中，并将软引用添加到队列中，获取缓存对象时，如果对象已经被回收，则移除Key，如果未被回收，则将对象添加到强引用队列中，避免被回收，如果强引用队列已经满了，则移除最早入队列的对象的引用。
- SynchronizedCache：线程安全缓存装饰器，为了保证线程安全，对操作缓存的方法使用synchronized关键字修饰。
- TransactionalCache：事务缓存装饰器，该缓存与其他缓存的不同之处在于，TransactionalCache增加了两个方法，即commit()和rollback()。当写入缓存时，只有调用commit()方法后，缓存对象才会真正添加到TransactionalCache对象中，如果调用了rollback()方法，写入操作将被回滚。
- WeakCache：弱引用缓存装饰器，功能和SoftCache类似，只是使用不同的引用类型。

缓存装饰器的使用示例：
```java
Cache cache = new PerpetualCache("com.xxx.test");
cache = new LruCache(cache);
cache = new SerializedCache(cache);
cache = new ScheduledCache(cache);
cache = new BlockingCache(cache);
cache = new LoggingCache(cache);
for (int i = 0; i < 10000; i++) {
    cache.putObject(i, i);
}
System.out.println(cache.getSize());
```

另外，MyBatis提供了一个CacheBuilder类，通过Builder模式创建缓存对象。
```java
Cache cache2 = new CacheBuilder("com.xxx.test2")
        .implementation(PerpetualCache.class)
        .addDecorator(LruCache.class)
        .clearInterval(600L)
        .size(1024)
        .readWrite(true)
        .blocking(true)
        .build();
for (int i = 0; i < 10000; i++) {
    cache2.putObject(i, i);
}
System.out.println(cache2.getSize());
```

### 一级缓存的实现

一级缓存使用PerpetualCache实例实现，在BaseExecutor类中维护了两个PerpetualCache属性，其中，localCache属性用于缓存MyBatis查询结果，localOutputParameterCache属性用于缓存存储过程调用结果。这两个属性在BaseExecutor构造方法中进行初始化，代码如下

![一级缓存](/resources/ca2.jpg "图片Title")

MyBatis通过CacheKey对象来描述缓存的Key值，从源码可以看出，缓存的Key与下面这些因素有关：
  1.  Mapper的Id
  2.  查询结果的偏移量及查询的条数。
  3.  具体的SQL语句及SQL语句中需要传递的所有参数。
  4.  MyBatis主配置文件中，通过environment标签配置的环境信息对应的Id属性值。

在BaseExecutor类的query()方法中，首先根据缓存Key从localCache属性中查找是否有缓存对象，如果查找不到，则调用queryFromDatabase()方法从数据库中获取数据，然后将数据写入localCache对象中。如果localCache中缓存了本次查询的结果，则直接从缓存中获取

需要注意的是，如果localCacheScope属性设置为STATEMENT，则每次查询操作完成后，都会调用clearLocalCache()方法清空缓存。除此之外，MyBatis会在执行完任意更新语句后清空缓存。
在分布式环境下，务必将MyBatis的localCacheScope属性设置为STATEMENT，避免其他应用节点执行SQL更新语句后，本节点缓存得不到刷新而导致的数据一致性问题。

### 二级缓存的实现

MyBatis二级缓存在默认情况下是关闭的，因此需要通过设置cacheEnabled参数值为true来开启二级缓存
SqlSession将执行Mapper的逻辑委托给Executor组件完成，而Executor接口有几种不同的实现，分别为SimpleExecutor、BatchExecutor、ReuseExecutor。另外，还有一个比较特殊的CachingExecutor，CachingExecutor用到了装饰器模式，在其他几种Executor的基础上增加了二级缓存功能。

我们看下Configuration类中创建Executor的代码：
![CachingExecutor](/resources/ca3.jpg "图片Title")


CachingExecutor在普通Executor的基础上增加了二级缓存功能，我们可以重点关注一下CachingExecutor类的实现。下面是CachingExecutor类的属性信息：

```java
public class CachingExecutor implements Executor {

  private Executor delegate;
  private TransactionalCacheManager tcm = new TransactionalCacheManager();
}
```

CachingExecutor类中维护了一个TransactionalCacheManager实例，TransactionalCacheManager用于管理所有的二级缓存对象。

```java
public class TransactionalCacheManager {

  private Map<Cache, TransactionalCache> transactionalCaches = new HashMap<Cache, TransactionalCache>();

  public void clear(Cache cache) {
    getTransactionalCache(cache).clear();
  }

  public Object getObject(Cache cache, CacheKey key) {
    return getTransactionalCache(cache).getObject(key);
  }
  
  public void putObject(Cache cache, CacheKey key, Object value) {
    getTransactionalCache(cache).putObject(key, value);
  }

  public void commit() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.commit();
    }
  }

  public void rollback() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.rollback();
    }
  }

  private TransactionalCache getTransactionalCache(Cache cache) {
    TransactionalCache txCache = transactionalCaches.get(cache);
    if (txCache == null) {
      txCache = new TransactionalCache(cache);
      transactionalCaches.put(cache, txCache);
    }
    return txCache;
  }

}

```
在TransactionalCacheManager类中，通过一个HashMap对象维护所有二级缓存实例对应的TransactionalCache对象，在TransactionalCacheManager类的getObject()方法和putObject()方法中都会调用getTransactionalCache()方法获取二级缓存对象对应的TransactionalCache对象，然后对TransactionalCache对象进行操作。在getTransactionalCache()方法中，首先从HashMap对象中获取二级缓存对象对应的TransactionalCache对象，如果获取不到，则创建新的TransactionalCache对象添加到HashMap对象中。


接下来以查询操作为例介绍二级缓存的工作机制。下面是CachingExecutor的query()方法的实现
![CachingExecutor](/resources/ca5.jpg "图片Title")
在CachingExecutor的query()方法中，首先调用createCacheKey()方法创建缓存Key对象，然后调用MappedStatement对象的getCache()方法获取MappedStatement对象中维护的二级缓存对象。然后尝试从二级缓存对象中获取结果，如果获取不到，则调用目标Executor对象的query()方法从数据库获取数据，再将数据添加到二级缓存中。当执行更新语句后，同一命名空间下的二级缓存将会被清空

CachingExecutor的update()方法中会调用flushCacheIfRequired()方法确定是否需要刷新缓存,在flushCacheIfRequired()方法中会判断<select|update|delete|insert>标签的flushCache属性，如果属性值为true，就清空缓存。<select>标签的flushCache属性值默认为false，而<update|delete|insert>标签的flushCache属性值默认为true

MyBatis除了提供内置的一级缓存和二级缓存外，还支持使用第三方缓存（例如Redis、Ehcache）作为二级缓存。MyBatis官方提供了一个mybatis-redis模块，该模块用于整合Redis作为二级缓存。在分布式系统中直接使用mybatis缓存的场景很少，一般查出数据都要进行一定业务组装之后再进行缓存，理解mybatis缓存的原理，可以避免踩坑。


