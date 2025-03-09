# DM模块-页面索引与向上管理DataItem

[toc]

本章将介绍一个实现简单的页面索引。并且实现了 DM 层对于上层的抽象：DataItem。

## 页面索引

页面索引，==缓存了每一页的空闲空间==。用于在上层模块进行插入操作时，能够快速找到一个合适空间的页面，而无需从磁盘或者缓存中检查每一个页面的信息。

### 大致思路

PageIndex将一页的空间划分成了 40 个区间。在启动时，就会遍历所有的页面信息，获取页面的空闲空间，安排到这 40 个区间中。insert 在请求一个页时，会首先将所需的空间向上取整，映射到某一个区间，随后取出这个区间的任何一页，都可以满足需求。

> **为什么要这么划分？**
>
> 这种划分是为了出现空间需求时，能快速找出合适的页分配空间，不然就需要将所有页排序，直接按空闲空间排序会浪费时间。这就相当与是==桶排序==。可以节约大量时间。

```java
public class PageIndex {
    // 将一页划成40个区间
    private static final int INTERVALS_NO = 40;
    //一页中一个区间有多大
    private static final int THRESHOLD = PageCache.PAGE_SIZE / INTERVALS_NO;
    //一个lists代表1页，被分为40个ArrayList，每个区间装的PageInfo类（PageInfo类存储有页号，剩余空间大小）
	//lists存储的是（一个区间，空余容量还剩一个区间的页面集合）（二个区间，空余容量还剩二个区间的页面集合...）
    private List<PageInfo>[] lists;
}
```

PageInfo实际保存了某页的页号，和该页空闲的区间大小

```java
public class PageInfo {
    public int pgno;
    public int freeSpace;
	//页号和空闲空间大小
    public PageInfo(int pgno, int freeSpace) {
        this.pgno = pgno;
        this.freeSpace = freeSpace;
    }
}
```

### 从 PageIndex 中获取页面

首先将所需的空间向上取整，映射到某一个区间，随后通过上面的List<PageIndex>数组 取出至少包含这个区间大小空闲空间的任何一页，都可以满足需求。

PageIndex 的实现也很简单，一个 List 类型的数组。从 PageIndex 中获取页面也很简单，算出区间号，直接取即可。返回的 PageInfo 中包含页号和空闲空间大小的信息。

```java
public PageInfo select(int spaceSize) {
    lock.lock();
    try {
        int number = spaceSize / THRESHOLD;//选择spaceSize需要多大的区间才放得下
        if(number < INTERVALS_NO) number ++;//区间若没有超过最大容量，则向上取整
        while(number <= INTERVALS_NO) {
            if(lists[number].size() == 0) {//没有这么大区间的页数
                number ++;//没有正好这么大区间的页数，就把更大容量的给它
                continue;
            }
            return lists[number].remove(0);//把该页数取走，填完后再重新插入
            //同一个页面是不允许并发写的
        }
        return null;
    } finally {
        lock.unlock();
    }
}
```



可以注意到，被选择的页，会直接从 PageIndex 中移除，这意味着，同一个页面是不允许并发写的。在上层模块使用完这个页面后，需要将其重新插入 PageIndex：

```java
public void add(int pgno, int freeSpace) {
    lock.lock();
    try {
        int number = freeSpace / THRESHOLD;//该页面还剩多少个区间
        //lists[1]放还剩下一个区间的页号和剩余空间
        //lists[2]放还剩下二个区间的页号和剩余空间
        lists[number].add(new PageInfo(pgno, freeSpace));
    } finally {
        lock.unlock();
    }
}
```

在 DataManager 被创建时，需要获取所有页面的剩余空间，并填充 PageIndex：

```java
//填入每一页的页号和该页的剩余空间
public void fillPageIndex() {
    int pageNumber = pc.getPageNumber();
    for(int i = 2; i <= pageNumber; i ++) {
        Page pg = null;
        try {
            pg = pc.getPage(i);
        } catch (Exception e) {
            Panic.panic(e);
        }
        pIndex.add(pg.getPageNumber(), PageX.getFreeSpace(pg));
        pg.release();
    }
}

// 获取页面的空闲空间大小
public static int getFreeSpace(Page pg) {
    return PageCache.PAGE_SIZE - (int)getFSO(pg.getData());
}
```



### 向上管理 DataItem 数据

DataItem 是 DM 层向上层提供的数据抽象。上层模块通过uid，向 DM 请求到对应的 DataItem，再获取到其中的数据。

DataItem 中保存的数据，结构如下：

[ValidFlag] [DataSize] [Data]
ValidFlag 1字节，0为合法，1为非法，删除一个 DataItem，只需要简单地将其有效位设置为 0
DataSize  2字节，标识Data的长度
上层模块在获取到 DataItem 后，可以通过 data() 方法获取到[data]部分，该方法返回的数组是数据共享的，而不是拷贝实现的，所以使用了 SubArray。实际就是都共用的这个DataItem里的数据。

```java
public SubArray data() {
    return new SubArray(raw.raw, raw.start+OF_DATA, raw.end);
}
```

### 对DataItem 进行修改

在上层模块试图对 DataItem 进行修改时，需要遵循一定的流程：

在修改之前需要调用DataItem 的 before() 方法，想要撤销修改时，调用DataItem的 unBefore() 方法，在修改完成后，调用DataItem的 after() 方法。整个流程，主要是为了保存前相数据，并及时落日志。DM 会保证对 DataItem 的修改是原子性的。

after()方法，主要就是调用 dm 中的一个方法，对修改操作落日志

> before为raw和oldRaw服务，后面的MVCC和回溯机制就依靠这个设计。

```java
//在修改之前需要调用 before() 方法，
//把raw移到oldRaw里面
@Override
public void before() {
    wLock.lock();
    pg.setDirty(true);
    System.arraycopy(raw.raw, raw.start, oldRaw, 0, oldRaw.length);
}
// 想要撤销修改时，调用 unBefore() 方法，oldraw移到raw里面
@Override
public void unBefore() {
    System.arraycopy(oldRaw, 0, raw.raw, raw.start, oldRaw.length);
    wLock.unlock();
}
//在修改完成后，调用after() 方法，主要就是调用 dm 中的一个方法，对修改操作落日志，不赘述。
@Override
public void after(long xid) {
    dm.logDataItem(xid, this);
    wLock.unlock();
}
// 为xid生成update日志
public void logDataItem(long xid, DataItem di) {
    byte[] log = Recover.updateLog(xid, di);
    logger.log(log);
}
```



## DM层的实现 DataManager

DataManager 是 DM 层直接对外提供方法的类，同时，也实现了 DataItem 对象的缓存。缓存中DataItem 存储的 key，是由页号和页内偏移组成的一个 8 字节无符号整数，**页号和偏移各占 4 字节**。

> 这个key就是代码里常出现的uid。

### 获取DataItem

DataItem缓存的具体实现类，需要继承前几节介绍的通用缓存框架。

1 从数据库中获取DataItem ，getForCache()，只需要从uid 中解析出页号，从 pageCache 中获取到页面Page，再根据偏移，解析出 DataItem 即可。

2 从缓存中获取DataItem，是继承父类的get(uid)方法。

需要注意的是，Page和DataItem共用一个缓存PageCache，Page的key是页号，DataItem的key是uid（页号和offset通过移位组成的值）。

> 将这个DataManager里面的`getForCache` 和 PageCacheImpl里面的 getForCache区分，PageCacheImpl里面的是直接走数据库，这个是调用PageCache的方法，先走PageCache缓存，没有再走PageCache的getForCache走数据库。

```java
protected DataItem getForCache(long uid) throws Exception {
    short offset = (short)(uid & ((1L << 16) - 1));
    uid >>>= 32;
    //从 key 中解析出页号
    int pgno = (int)(uid & ((1L << 32) - 1));
    //从 pageCache 中获取到页面，走到pc继承的AbstractCache的get方法
    //缓存的hashmap里面有会直接返回，没有就走数据库
    Page pg = pc.getPage(pgno);
    return DataItem.parseDataItem(pg, offset, this);
}
```

### 创建DM实现类

DM校验（见PageOne节）
从已有文件创建 DataManager 和从空文件中创建 DataManager 的流程稍有不同，除了 PageCache 和 Logger 的创建方式有所不同以外，从空文件创建首先需要对第一页进行初始化，而从已有文件创建，则是需要对第一页进行校验，来判断是否需要执行恢复流程。并重新对第一页生成随机字节。

```java
public static DataManager create(String path, long mem, TransactionManager tm) {
    PageCache pc = PageCache.create(path, mem);    
    Logger lg = Logger.create(path);
    DataManagerImpl dm = new DataManagerImpl(pc, lg, tm);
    //初始化PageOne
    dm.initPageOne();
    return dm;
}

public static DataManager open(String path, long mem, TransactionManager tm) {
    PageCache pc = PageCache.open(path, mem);
    Logger lg = Logger.open(path);
    DataManagerImpl dm = new DataManagerImpl(pc, lg, tm);
    // 校验PageOne
    // 在打开已有文件时时读入PageOne，并验证正确性
    if(!dm.loadCheckPageOne()) {
        Recover.recover(tm, lg, pc);
    }
    //填入每一页的页号和该页的剩余空间
    dm.fillPageIndex();
    PageOne.setVcOpen(dm.pageOne);
    dm.pc.flushPage(dm.pageOne);
    return dm;
}
```

### DataItem 的增删改查

DM 层提供了三个功能供上层使用，分别是读、插入和修改。

修改是通过读出的 DataItem 实现的（此外，对DataItem执行修改需要严格按照before()和after()顺序），于是 DataManager 只需要提供 read() 和 insert() 方法。

读取DataItem层需要判断[validFlag]位是否有效。

**read——两层缓存体现！！！**

> 调用父类AbstractCache<DataItem>的get方法
>
> 先从这个AbstractCache<DataItem>里面的hashmap里面找，这就是第一层缓存。
>
> 没有的话，会在这个get方法里面调用getForCache方法，这个getForCache方法会被当前类DataManager实现，这个实现中会先调用DM里面的pc这个页面缓存，没有再通过pc的getForCache走数据库。

```java
public DataItem read(long uid) throws Exception {
    //以PageX管理页面的时候FSO后面的DATA其实就是一个个的DataItem包
    DataItemImpl di = (DataItemImpl)super.get(uid);
    //[ValidFlag]是否为0
    if(!di.isValid()) {
        di.release();
        return null;
    }
    return di;
}
```

insert() 方法，在 pageIndex 中获取一个足以存储插入内容的页面的页号，获取页面后，首先需要写入插入日志，接着才可以通过 pageX 插入数据，并返回插入位置的偏移。最后需要将插入完成后的页面信息重新插入 pageIndex。

**insert**

> 从DM里的pIndex中取出合适的空闲大小的页，（没有就新建一个页刷入数据库和空闲链表）；
>
> 做写日志；
>
> 将取出的页，在放回去（更新了剩余空闲空间）；

```java
public long insert(long xid, byte[] data) throws Exception {
    //page里的data是dataItem
    byte[] raw = DataItem.wrapDataItemRaw(data);//打包成dateItem的格式
    if(raw.length > PageX.MAX_FREE_SPACE) {
        throw Error.DataTooLargeException;
    }
    // 尝试获取可用页，5次都获取不到就报错
    PageInfo pi = null;
    for(int i = 0; i < 5; i ++) {
        pi = pIndex.select(raw.length);
        if (pi != null) {
            break;
        } else {
            //没有满足条件的数据页，新建一个数据页并写入数据库文件
            int newPgno = pc.newPage(PageX.initRaw());
            //将这新页加入空闲空间管理链表
            pIndex.add(newPgno, PageX.MAX_FREE_SPACE);
        }
    }
    if(pi == null) {
        throw Error.DatabaseBusyException;
    }
    //System.out.println(pi.pgno);
    Page pg = null;
    int freeSpace = 0;
    try {
        pg = pc.getPage(pi.pgno);
        // 首先做日志  raw dataItem page里的data
        byte[] log = Recover.insertLog(xid, pg, raw);
        logger.log(log);
        // 再执行插入操作
        short offset = PageX.insert(pg, raw);
        pg.release();
        //返回插入位置的偏移
        return Types.addressToUid(pi.pgno, offset);
    } finally {
        // 将取出的pg重新插入pIndex
        if(pg != null) {
            //获取pg当下的空闲空间，通过pg这个数据结构，无需走数据库
            pIndex.add(pi.pgno, PageX.getFreeSpace(pg));
        } else {
            pIndex.add(pi.pgno, freeSpace);
        }
    }
}
```



DataManager 正常关闭时，需要执行缓存和日志的关闭流程，不要忘了设置第一页的字节校验：

```java
public void close() {
    super.close();
    logger.close();
    PageOne.setVcClose(pageOne);
    pageOne.release();
    pc.close();
}
```

### 获得DataItem的流程 

这里应该特指DM里的getForCache方法。

==需要注意的是，DM中DataItem和Page共用一个缓存pc==，Page的key是页号，DataItem的缓存是uid，，这个缓存pc是一个PageCachImpl，继承了AbstractCache<Page>，这里称其为DataItem在DM中的缓存2。

> DataManager中继承了AbstractCache<DataItem>，这里称其为DataItem在DM中的缓存1。

获得一个DataItem的流程大致如下（例如前面的read方法）：

1 缓存1中有DataItem，从缓存1中拿

2 缓存1中无DataItem，但是缓存2中有存放DataItem的页面Page，从缓存中拿Page，从Page中拿DataItem，把该DataItem写入缓存。

3 缓存中无DataItem，且无存放DataItem的页面Page，从数据库中拿该Page，从Page拿DataItem，把该DataItem写入缓存。



### 测试代码

```java
public class test {
    public static void main(String[] args) throws Exception {
        TransactionManagerImpl tm=TransactionManager.open("cun/tm");
        //开启一个事务
        long xid=tm.begin();
        //1<<17设置页面的大小，除以一页的大小就是有几页
        DataManager dm=DataManager.open("cun/dm",1 << 20,tm);
        //dm.read(1);
        byte[]b=new byte[1024];
        long uid=dm.insert(3,b);
        dm.close();
        //提交事务
        tm.commit(xid);
    }
}
```



## 梳理DM

### 每个子类的功能。

1、AbstractCache：引用计数法的缓存框架，留了两个从数据源获取数据和释放缓存的抽象方法给具体实现类去实现。

> 继承同一父类的不同子类可以实现同名不同方法，这是多态的一种实现方式：覆盖。
>
> 另一种是重载：允许同名不同参数的函数。

2、PageImpl：数据页的数据结构，包含页号、是否脏数据页、数据内容、所在的PageCache缓存。
3、PageOne：校验页面，用于启动DM的时候进行文件校验。
4、PageX：每个数据页的管理器。initRaw()新建一个数据页并设置FSO值，FSO后面存的其实就是一个个DataItem数据包
5、PageCacheImpl：数据页的缓存具体实现类，除了重写获取 和释放两个方法外，还完成了所有数据页的统一管理：
    1）获取数据库中的数据页总数；getPageNumber()
    2）新建一个数据页并写入数据库文件；newPage(byte[] initData)
    3）从缓存中获取指定的数据页；getPage(int pgno)
    4）删除指定位置后面的数据页；truncateByBgno(int maxPgno)
6、PageIndex：方便DataItem的快速定位插入，其实现原理可以理解为HashMap那种数组+链表结构（实际实现是 List+ArrayList），先是一个大小为41的数组 存的是区间号（区间号从1>开始），然后每个区间号数组后面跟一个数组存满足空闲大小的所有数据页信息（PageInfo）。

> 可以理解为，为了快速获取合适大小的页，实现的桶排序的空闲空间链表。

7、Recover：日志恢复策略，主要维护两个日志：updateLog和insertLog，重做所有已完成事务 redo，撤销所有未完成事务undo
8、DataManager：统揽全局的类，主要方法也就是读写和修改，全部通过DataItem进行。

### 打开DM的流程

首先从DataManager进去创建DM（打开DM就不谈了，只是多了个检验PageOne 和更新PageIndex），需要执行的操作是：
1）新建PageCache，DM里面有 页面缓存 和 DataItem缓存 两个实现；DataItem缓存也是在PageCache中获取的，DataItem缓存不存在的时候就去PageCache缓存获取，PageCache缓存没有才去数据库文件中获取；

​	1、read操作：从DataItem缓存（AbstractCache<DataItem>）中读，没有就直接从数据库文件中读

​	2、insert操作：从DM里的PageIndex中获取页的空闲块，将数据插入其所在的页。

2）新建日志，
3）构建DM管理器；
4）初始化校验页面1： dm.initPageOne()nnnDataManager的所有功能（主要功能就是CRUD，进行数据的读写修改都是靠DataItem进行操作的 ，所以PageX管理页面的时候FSO后面的DATA其实就是一个个的DataItem包）：
   1、初始化校验页面1：
    initPageOne() 和 启动时候进行校验：loadCheckPageOne()
   2、读取数据 read(long uid)：
    从DataItem缓存中读取一个DataItem数据包并进行校验，如果DataItem缓存中没有就会调用 DataManager下的getForCache(long uid)从PageCache缓存中读取DataItem数据包并加入DataItem缓存（其实PageCache缓存和DataItem缓存都是共用的一个cache Map存的，只是key不一样，page的key是页号，DataItem的key是uid，页号+偏移量），如果PgeCache也没有就去数据库文件读取。
   3、插入数据 insert(long xid, byte[] data)：
    先把数据打包成DataItem格式，然后在 pageIndex 中获取一个足以存储插入内容的页面的页号； 获取页面后，需要先写入插入日志Recover.insertLog(xid, pg, raw)，接着才可以通过 pageX 在目标数据页插入数据PageX.insert(pg, raw)，并返回插入位置的偏移。如果在pageIndex中没有空闲空间足够插入数据了，就需要新建一个数据页pc.newPage(PageX.initRaw())。最后需要将页面信息重新插入 pageIndex。
4、修改数据就是先读取数据，然后修改DataItem内容，再插入DataItem数据。但是在修改数据操作的前后需要调用DataItemImp.after()进行解写锁并记录更新日志，这里需要依赖DataManager里面的logDataItem(long xid, DataItem di)方法；
5、释放缓存：
释放DataItem的缓存，实质上就是释放DataItem所在页的PageCache缓存

