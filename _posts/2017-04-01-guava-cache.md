---
layout: post
---

## 简介与使用场景
Guava Cache是一种内存缓存。实际工作中，为防止业务瞬间QPS过高对DB带来的压力，我们通常会使用缓存，如redis，memcached，tair等成熟的缓存技术。但同时，业务瞬间QPS过高也有可能对缓存造成压力。
故，可以考虑本地内存缓存做一级缓存，redis等做二级缓存。

```
  真实业务场景：某个业务场景下，我们DB中的数据全部放入缓存，查询时仅查询tair，
  不查询DB，即以tair数据为准（不用“先查缓存，查不到再查DB”的策略的原因：
  业务初始状态下或一段较长时间内，数据基本为空，缓存数据均会miss，
  这时DB的QPS仍然很高）。
  后台增加同步任务定时同步数据至缓存，并支持手动触发 应急。
```
## 内存缓存需要考虑的问题
1. 缓存的数据不要超过内存总量
2. 缓存的回收机制，比如需要支持LRU回收算法；防止系统内存溢出
3. 如果再支持定时失效，定时回刷，那就更好了

## 现成解决方案：Guava Cache
### 1. 数据加载
Guava Cache支持“get-if-absent-compute”语义（查询-如果不存在-计算-放入缓存）。只需定义自己的load方法

```java
  LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
        .maximumSize(1000)
        .build(
            new CacheLoader<Key, Graph>() {
                public Graph load(Key key) throws AnyException {
                    return createExpensiveGraph(key);
                }
            });

...
try {
    return graphs.get(key);
} catch (ExecutionException e) {
    throw new OtherException(e.getCause());
}
```
#### 1.1 显示插入
同时，Guava Cache支持显示插入。可直接调用cache.put。

Cache.asMap()视图提供的任何方法也能修改缓存。

### 2. 缓存回收
Guava Cache支持三种基本的缓存回收方式：基于容量回收，定时回收和基于引用回收。

#### 2.1 基于容量的回收（size-based eviction）
1. 限定cache key值的上限：CacheBuilder.maximumSize(long)。缓存将尝试回收最近没有使用或很少使用的缓存项；
2. 限定cache总内存大小(最大总重，需要指定每个缓存项权重计算方法)：CacheBuilder.maximumWeight(long).weiger()
 
 ```java
  LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
        .maximumWeight(100000)
        .weigher(new Weigher<Key, Graph>() {
            public int weigh(Key k, Graph g) {
                return g.vertices().size();
            }
        })
        .build(
            new CacheLoader<Key, Graph>() {
                public Graph load(Key key) { // no checked exception
                    return createExpensiveGraph(key);
                }
            });
 ```
 
 ```
  【注意】：不管是基于size还是基于weight的回收，缓存均在size/weight将要达到阈值时就开始回收，而不是在到达阈值之后这在使用时应特别注意。
  
  基于weight的回收，权重是在缓存创建的时候计算的，因此要考虑权重计算的复杂度
 ```
 
#### 2.2 定时回收(Timed Eviction)
CacheBuilder提供两种定时回收的方法：
1. expireAfterAccess(long, TimeUnit)：缓存项在给定时间内没有被 **读/写** 访问，则回收。这种缓存的回收顺序和基于大小的一样
2. expiredAfterWrite(long, TimeUnit)：缓存项在给定时间内没有被写访问（创建或覆盖）则回收。
 ```
  如果认为缓存数据总是在特定时间后变得陈旧不可用，就应该使用expireAfterWrite
 ```

#### 2.3 基于引用的回收
通过使用弱引用的键、或弱引用的值、或软引用的值，Guava Cache可以把缓存设置为允许垃圾回收：

1. CacheBuilder.weakKeys()：使用弱引用存储键。当键没有其它（强或软）引用时，缓存项可以被垃圾回收。因为垃圾回收仅依赖恒等式（==），使用弱引用键的缓存用==而不是equals比较键。
2. CacheBuilder.weakValues()：使用弱引用存储值。当值没有其它（强或软）引用时，缓存项可以被垃圾回收。因为垃圾回收仅依赖恒等式（==），使用弱引用值的缓存用==而不是equals比较值。
3. CacheBuilder.softValues()：使用软引用存储值。软引用只有在响应内存需要时，才按照全局最近最少使用的顺序回收。考虑到使用软引用的性能影响，我们通常建议使用更有性能预测性的缓存大小限定（见上文，基于容量回收）。使用软引用值的缓存同样用==而不是equals比较值。

#### 2.4 显示清除

1. 清除单个key：Cache.invalidate(key)
2. 批量清除：Cache.invalidateAll(keys)
3. 清除所有key：Cache.invalidateAll()

#### 2.5 刷新
刷新和回收不一样，refresh(key)，刷新表示加载某个key的新值，这个过程可以是异步的。

```java
//有些键不需要刷新，并且我们希望刷新是异步完成的
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
        .maximumSize(1000)
        .refreshAfterWrite(1, TimeUnit.MINUTES)
        .build(
            new CacheLoader<Key, Graph>() {
                public Graph load(Key key) { // no checked exception
                    return getGraphFromDatabase(key);
                }

                public ListenableFuture<Key, Graph> reload(final Key key, Graph prevGraph) {
                    if (neverNeedsRefresh(key)) {
                        return Futures.immediateFuture(prevGraph);
                    }else{
                        // asynchronous!
                        ListenableFutureTask<Key, Graph> task=ListenableFutureTask.create(new Callable<Key, Graph>() {
                            public Graph call() {
                                return getGraphFromDatabase(key);
                            }
                        });
                        executor.execute(task);
                        return task;
                    }
                }
            });
```

1. CacheBuilder.refreshAfterWrite(long, TimeUnit)可以为缓存增加自动定时刷新功能。和expireAfterWrite相反，refreshAfterWrite通过定时刷新可以让缓存项保持可用，

```
但请注意：缓存项只有在被检索时才会真正刷新
（如果CacheLoader.refresh实现为异步，那么检索不会被刷新拖慢）。
因此，如果你在缓存上同时声明expireAfterWrite和refreshAfterWrite，
缓存并不会因为刷新盲目地定时重置，如果缓存项没有被检索，那刷新就不会真的发生，缓存项在过期时间后也变得可以回收。
```
