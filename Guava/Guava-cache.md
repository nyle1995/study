# Guava Cache 

官方文档地址：https://github.com/google/guava/wiki/CachesExplained#applicability

## 使用

### 1. 构造
```
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .maximumSize(1000)
       .expireAfterWrite(10, TimeUnit.MINUTES)
       .removalListener(MY_LISTENER)
       .build(
           new CacheLoader<Key, Graph>() {
            // 必须实现
             public Graph load(Key key) throws AnyException {
               return createExpensiveGraph(key);
             }

             // 可以实现loadAll， reload
           });
```

### 2. 获取

1. get
```
会要求你try住  load中的异常
try {
  return graphs.get(key);
} catch (ExecutionException e) {
  throw new OtherException(e.getCause());
}
```

2. graphs.getUnchecked(key); 


3. 批量获取

```
getAll(Iterable<? extends K> keys) throws ExecutionException;

这里需要在CacheBuilder定义CacheLoader时候，实现 loadAll
```

4. 自定义load方法

```
V get(K, Callable<V>);

cache.get(key, new Callable<Value>() {
    @Override
    public Value call() throws AnyException {
      return doThingsTheHardWay(key);
    }
});
```
### 3. 换出

#### a. 根据maxsize 或者 maxweight失效
```
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
        .maximumSize(100)
       .build(
           new CacheLoader<Key, Graph>() {
             public Graph load(Key key) { // no checked exception
               return createExpensiveGraph(key);
             }
           });
或者

       .maximumWeight(100000)
       .weigher(new Weigher<Key, Graph>() {
          public int weigh(Key k, Graph g) {
            return g.vertices().size();
          }
        })

不能同时设置maxsize 和 maxweight
换出时采用LRU策略，accessQueue
```

这里有两点注意的  
1. weight 只有在第一次set到cache的时候计算，不会根据value，key变化而变化 
2. 不是严格按照maxsize设置的  
        因为LoadingCache 是类似ConcurrentHashmap1.6版本的，分成很多个Segment维护。  
        初始化的时候，会给每个Segment维护一个maxSize, 换出是每个Segment维护的

#### b. 根据时间过期或者刷新

```
expireAfterAccess(long, TimeUnit)
expireAfterWrite(long, TimeUnit)
refreshAfterWrite(long, TimeUnit)
```

#### c. 根据对象引用失效过期

```
CacheBuilder.weakKeys()
CacheBuilder.weakValues()
CacheBuilder.softValues()
```

#### d. 换出监听方法
```
 CacheBuilder
    .removalListener(new RemovalListener<String, InnerValue>() {
        @Override
        public void onRemoval(RemovalNotification<String, InnerValue> notification) {
            System.out.println("remove " + notification.getKey());
        }
    })
```

### 4. 清理
清理时机失效meta：  
1. 当进行写操作的时候
2. 或者如果一直没有写操作（写操作后64次读操作）,会由读操作来cleanup

如果希望有一个后台线程定时清理，可以单独其一个线程定时调用Cache.cleanUp()

### 5. 源码阅读

基本结构和ConcurrentHashMap1.6版本的类似，使用Segment，之后再下分为不同的节点

![img](http://m10.music.126.net/21180228005704/48082f7fc96c1565eb38a3c9e9691bd2/music-app/mix_1521824224293634)

1. 引用被gc失效
```
为了支持虚指针，软指针回收  
Entry在hashMap中就是只需要保存Key，Value  
这里Key，Value都可能被回收调，所以都用Reference对象

并且维护gc后的队列，来用于gc  
keyReferenceQueue  
valueReferenceQueue  

而需要通过value找到对应的节点，所以每个value 需要维护一个指向entry的指针
```

2. 支持失效时间和LRU换出

```
维护3个队列
- Queue<ReferenceEntry<K, V>> writeQueue;  最近写的elements
- Queue<ReferenceEntry<K, V>> accessQueue; 最近访问的elements
- Queue<ReferenceEntry<K, V>> recencyQueue 最近读取的队列

前两个队列都是双向链表（最后连成一个环）实现
每一个Entry都维护前后节点，最新访问和最新写入的element放在最后
非线程安全
只有加锁之后才能操作

recencyQueue 是一个 ConcurrentLinkedQueue。  
有读操作的时候，会add到这个queue
当需要clear，或者写入的时候，会把recencyQueue中的数据添加到accessQueue队列中，再进行处理  
```

3. 加载过程  

```
调用load的时候，会加锁，然后生成一个LoadingValueReference临时节点放在table里  
而如果refrsh的时候，会先返回之前的值  

load结束之后，会entry中的value修改为真实的reference

- 其他线程  
而load过程中，其他的线程get时会调用waitForLoadingValue  
相当于调用了 Future.get，这个future是load返回的Future，在Future结束之后会把线程换气

V waitForLoadingValue(ReferenceEntry<K, V> e, K key, ValueReference<K, V> valueReference)
        throws ExecutionException {
   ***
    V value = valueReference.waitForValue();
    ***
}

public V waitForValue() throws ExecutionException {
      return getUninterruptibly(futureValue);
}

public static <V> V getUninterruptibly(Future<V> future) throws ExecutionException {
    ***
          return future.get();
   ***
}
```


###### refresh的问题

refresh会调用异步loadAsync，会调用CacheLoader的reload, 默认reload也是同步调用load，所以没有异步逻辑

可以实现reload，使用其他的线程来返回ListenableFuture
```
new CacheLoader<String, InnerValue>() {
    @Override
    public InnerValue load(String key) throws Exception {
        InnerValue result = new InnerValue();
        result.value = "" + key + "_" + System.currentTimeMillis();
        return result;
    }

    @Override
    public ListenableFuture<InnerValue> reload(String key, InnerValue oldValue) throws Exception {
        return service.submit(new Callable<InnerValue>() {
            @Override
            public InnerValue call() throws Exception {
                Thread.sleep(1000);
                return load(key);
            }
        });
    }
});
```
这样调用refresh的时候，不会卡住主线程。

注意： 
1. 如果调用refresh后，有其他线程设置了值的话，refresh线程在执行结束后，不会再次覆盖该值
2. 如果refresh后，有其他线程删除了这个值，或者这个entry已经被gc了，refresh线程在执行结束后，还是会把这个值添加到table里

#### 其他
Spring5 的本地cache，改用了[Caffeine](https://baijiahao.baidu.com/s?id=1565651081655610&wfr=spider&for=pc)





