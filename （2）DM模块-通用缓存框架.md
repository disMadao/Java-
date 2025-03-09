# DM模块-通用缓存框架

[toc]

## DM模块作用

DM模块是上层模块和文件系统之间的一个抽象层，**向下从数据库直接读写文件（以Page的格式），向上提供数据的包装（以DataItem的格式）**；另外就是**日志**功能，发生错误时可以根据日志恢复。
可以注意到，无论是向上还是向下，DM 都提供了一个缓存的功能，用内存操作来保证效率
更详细的说

1) 向下分页管理 DB 文件，并提供**HashMap缓存，key是页号，value是page**；
2) 向上抽象 DB 文件为 DataItem 供上层模块使用，并提供**HashMap缓存，key是uid,value是DataItem**；
3) 管理日志文件，保证在发生错误时可以根据日志进行恢复；
后续，DM模块进行日志文件的恢复，需要通过TM模块来判断事务的状态来决定是redo还是undo，所以**DM模块是建立在TM模块之上**的。



## 缓存框架 AbstractCache

由于从缓存中拿数据比从数据库中拿要快很多，所以这里建立一个通用的缓存框架

AbstractCache，后续DM模块从数据库拿的Page缓存和传给上层模块的DataItem缓存都是继承在这个通用缓存框架之上的。

> AbstractCache类  实现了一个引用计数策略的缓存

### 引用计数策略

一个引用计数策略的缓存：除了普通的缓存功能，还需要另外维护一个计数。每引用一次，计数加一，当引用归零时，缓存就会驱逐这个资源。

除此以外，为了应对多线程场景，还需要记录哪些资源正在从数据源获取中

```java
private HashMap<Long, T> cache;                     // 缓存里的数据
private HashMap<Long, Integer> references;          // 资源的引用个数
private HashMap<Long, Boolean> getting;             // 正在从数据源被获取的资源
```

其他数据：

```java
private int maxResource;                            // 缓存的最大缓存资源数
private int count = 0;                              // 缓存中元素的个数
private Lock lock;
```



### 为啥不选用LRU策略？

LRU 策略中，资源驱逐不可控，上层模块无法感知。

只有上层模块主动释放引用，缓存在确保没有模块在使用这个资源了，才会去驱逐资源。同样，在缓存满了之后，引用计数法无法自动释放缓存，此时应该直接报错。

> 这里作者设定了一个特殊的并发错误场景，但得看这发生的频率和应用场景吧，直观来看的话，感觉不如LRU，可能是LRU实现过于复杂了吧。

### MySQL中

mysql中实际是使用LRU的，只是做了两点改进为了避免两个问题

- 将LRU链表划分为young和 old 两个区域，加入缓冲池的页，优先插入old区域；当页被访问时，才进入young区域，目的是==为了解决预读失效问题==（利用空间局部性失败）。
- 当 页被访问 且 old区域停留超过指定阈值 时，才会将页插入到young区域，否则还是插入到old 区域 ，目的是==为了解决批量数据访问，大量热数据淘汰的问题==。



## 主要源码 AbstractCache

### 获取缓存

1  获取数据要用一个死循环先判断getting.get()里面有无被其他线程正在从数据源请求这个资源，有，等待，无，再判断缓存中有无这个资源
2  判断缓存中有无这个资源，有从缓存中拿资源，引用加1，无，在getting中注册，表示我正在请求资源，再从数据库中拿走资源并放入缓存中，引用加1，最后，从getting中注销资源。

```java
protected T get(long key) throws Exception {
    //在通过 get() 方法获取资源时，首先进入一个死循环，来无限尝试从缓存里获取。
    // 首先就需要检查这个时候是否有其他线程正在从数据源获取这个资源，如果有，就过会再来看看
    while(true) {
        lock.lock();
        if(getting.containsKey(key)) {
            // 请求的资源正在被其他线程获取
            lock.unlock();
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
                continue;
            }
            continue;
        }
        //如果资源在缓存中，就可以直接获取并返回了，记得要给资源的引用数 +1。
        // 否则，如果缓存没满的话，就在 getting 中注册一下，该线程准备从数据源获取资源了。
        if(cache.containsKey(key)) {
            T obj = cache.get(key);
            references.put(key, references.get(key) + 1);
            lock.unlock();
            return obj;
        }        
        if(maxResource > 0 && count == maxResource) {
            lock.unlock();
            throw Error.CacheFullException;
        }
        // 如果资源不在缓存中，且缓冲没满，就在 getting 中注册一下，该线程准备从数据源获取资源了。
        count ++;
        getting.put(key, true);
        lock.unlock();
        break;
    }
    //从数据源获取资源就比较简单了，直接调用那个抽象方法即可，
    // 获取完成记得从 getting 中删除 key。
    T obj = null;
    try {
        //从数据库中获取资源
        obj = getForCache(key);
    } catch(Exception e) {
        lock.lock();
        count --;
        getting.remove(key);
        lock.unlock();
        throw e;
    }
    //成功从数据库获取资源后，移除getting，把资源放入缓存，引用计数器加1
    lock.lock();
    getting.remove(key);
    cache.put(key, obj);
    references.put(key, 1);
    lock.unlock();
    return obj;
}
```

### 从数据库获取资源、释放资源

这里还留了两个方法给子类实现，分别是从当资源不在缓存时的从数据库获取资源方法，和当资源被驱逐时的将资源写回数据库的方法。

后面章节中从数据库拿的Page缓存和DM模块传给上层模块的DataItem缓存就是继承了本章节所介绍的通用缓存框架，需要对这两个方法进行重写。

```java
/**
* 当资源不在缓存时的获取行为
*/
protected abstract T getForCache(long key) throws Exception;
/**
* 当资源被驱逐时的写回行为
*/
protected abstract void releaseForCache(T obj);
```

释放缓存
释放一个缓存直接从 references 中减 1，如果已经减到 0 了，就可以回源，并且删除缓存中所有相关的结构了

```java
protected void release(long key) {
    lock.lock();
    try {
        int ref = references.get(key)-1;
        if(ref == 0) {
            T obj = cache.get(key);
            releaseForCache(obj);
            references.remove(key);
            cache.remove(key);
            count --;
        } else {
            references.put(key, ref);
        }
    } finally {
        lock.unlock();
    }
}
```



### 安全关闭

以防万一，缓存应当还有以一个安全关闭的功能，在关闭时，不管引用是否为0，都可以将缓存中所有的资源强行回源。

```java
protected void close() {
    lock.lock();
    try {
        Set<Long> keys = cache.keySet();
        for (long key : keys) {
            T obj = cache.get(key);
            releaseForCache(obj);
            references.remove(key);
            cache.remove(key);
        }
    } finally {
        lock.unlock();
    }
}
```

## 共享内存数组 SubArray

在 Java 中，当你执行类似 subArray 的操作时，只会在底层进行一个复制，无法同一片内存。
于是，我写了一个 SubArray 类，来（松散地）规定这个数组的可使用范围，后面会用到。
最终目的：实现两个数组共享内存

```java
public class SubArray {
    public byte[] raw;//有效位 data大小 data
    public int start;
    public int end;

    public SubArray(byte[] raw, int start, int end) {

        this.raw = raw;
        this.start = start;
        this.end = end;

    }
}
```



