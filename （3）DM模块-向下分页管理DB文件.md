# DM模块-向下分页管理DB文件

[toc]

上一节我们已经实现了一个通用的缓存框架，那么这一节我们将文件系统抽象成页面，**每次对文件系统的读写都是以页面Page为单位的**。所以，对下层模块需要缓存页面Page，就可以直接借用那个缓存的框架了。

但是首先，需要定义出页面的结构。注意**这个页面是存储在内存中的，与已经持久化到磁盘的抽象页面有区别。**

## 页面 Page

最底层的一层抽象，将存储的字节数组抽象成页。后面会将页再抽象一层分为普通页和效验页。

参考大部分数据库的设计，将默认数据页大小定为 8K。定义一个页面结构如下：

> mysql中页默认16KB，这里是在PageCache中设置为8KB

```java
public class PageImpl implements Page {
    private int pageNumber;//pageNumber 是这个页面的页号，该页号从 1 开始
    private byte[] data;//data 就是这一页实际包含的字节数据
    private boolean dirty;//dirty 标志着这个页面是否是脏页面，在缓存驱逐的时候，
    // 脏页面需要被写回磁盘。
    private Lock lock;
    private PageCache pc;// 
    // 可以快速对这个页面的缓存进行释放操作。
}
```

其中，pageNumber 是这个页面的页号，该页号从 1 开始。

data 就是这个页实际包含的字节数据。

dirty 标志着这个页面是否是脏页面，在缓存驱逐的时候，脏页面需要被写回磁盘。

这里保存了一个 PageCache的引用，用来方便在拿到 Page 的引用时可以快速对这个页面的缓存进行释放操作。

## 页面缓存 PageCache

从数据库中获得page缓存
页面缓存的具体实现类PageCacheImpl，需要继承上一节介绍的抽象缓存框架，并且实现 getForCache() 和 releaseForCache() 两个抽象方法。

需要区分继承父类的get(Page page)方法。

1 由于数据源就是文件系统，getForCache() 直接从数据库中读取，并包裹成 Page 即可。

2 get(Page page)是从缓存中读取资源，如果缓存中无，再调用getForCache()方法从数据库拿

```java
protected Page getForCache(long key) throws Exception {
    int pgno = (int)key;
    //根据页号算偏移量
    long offset = PageCacheImpl.pageOffset(pgno);
    ByteBuffer buf = ByteBuffer.allocate(PAGE_SIZE);
    fileLock.lock();
    try {
        fc.position(offset);
        fc.read(buf);
    } catch(IOException e) {
        Panic.panic(e);
    }
    fileLock.unlock();
    //this:PageCache的引用，用来方便在拿到 Page 的引用时
    //可以快速对这个页面的缓存进行释放操作。
    return new PageImpl(pgno, buf.array(), this);
}

// 页号从 1 开始
private static long pageOffset(int pgno) {
    return (pgno-1) * PAGE_SIZE;
}
```



### page缓存回源

需要区分继承父类的release(Page page)方法。
1 releaseForCache() 驱逐页面时，也只需要根据页面是否是脏页面，来决定是否需要写回数据库

2 release(Page page)是页面已经在缓存中，每调用一次，引用计数减1，当引用计数为0时，调用releaseForCache() 回源。

```java
protected void releaseForCache(Page pg) {
    if(pg.isDirty()) {
        flush(pg);
        pg.setDirty(false);
    }
}
```

flush()方法把写好的page格式数据写回数据库中

```java
private void flush(Page pg) {
    int pgno = pg.getPageNumber();
    long offset = pageOffset(pgno);//该页初始位置的偏移量
    fileLock.lock();
    try {
        ByteBuffer buf = ByteBuffer.wrap(pg.getData());
        fc.position(offset);
        fc.write(buf);
        fc.force(false);
    } catch(IOException e) {
        Panic.panic(e);
    } finally {
        fileLock.unlock();
    }
}
```



### 新建页面

PageCache 还使用了一个 AtomicInteger，来记录了当前打开的数据库文件有多少页。这个数字在数据库文件被打开时就会被计算，并在新建页面时自增。

```java
public int newPage(byte[] initData) {
    int pgno = pageNumbers.incrementAndGet();
    Page pg = new PageImpl(pgno, initData, null);
    flush(pg);  // 新建的页面需要立刻写回
    return pgno;
}
```

### 测试代码

```java
public class test {
    public static void main(String[] args) throws Exception {
        //初始化给了16页页面，页面大小是1 << 13
        PageCacheImpl pc=PageCache.open("cun/page",1 << 17);
        //新建一页，写数据,返回页号
        byte a[]=new byte[10];
        int pago=pc.newPage(a);
        System.out.println(pago);
        //从缓存中拿第 2 页
        Page pg=pc.getPage(2);
        pg.setDirty(true);
        //缓存中引用-1，引用为0时，调用releaseForCache（）回源
        pg.release();
        //从数据库中拿第2页
        pg=pc.getForCache(2);
        //不管引用是否为0，强制回源，且关闭dm文件
        pc.close();
    }
}
```



## 数据页管理 

> 操作Page都要通过PageCache，PageCache本身存储着n个的页，从PageCache里拿出指定类型数据页，然后通过指定的数据页给的方法去操作页。
>
> PageCache里的每个页都是PageImpl，这些PageImpl一个是PageOne，一些是PageX，一些未定义。
>
> PageOne、PageX里面封装了一些操作PageImpl（实际给的是接口类Page）的方法。
>
> 这就是PageCache和PageOne、PageX和Page、PageImpl的关系。

### 第一页（PageOne校验页）

数据库文件的第一页，通常用作一些特殊用途，比如存储一些元数据，用来启动检查什么的。MYDB 的第一页，只是用来做启动检查。具体的原理是，在每次数据库启动时，会生成一串随机字节，存储在 100 ~ 107 字节。在数据库正常关闭时，会将这串字节，拷贝到第一页的 108 ~ 115 字节。

**这样数据库在每次启动时，就会检查第一页两处的字节是否相同，以此来判断上一次是否正常关闭。**如果是异常关闭，就需要执行数据的恢复流程。

设置校验码

启动时设置初始字节：

```java
//启动时设置初始字节：
public static void setVcOpen(Page pg) {
    pg.setDirty(true);//对页面有修改时就设置为脏数据，然后flush到磁盘中
    setVcOpen(pg.getData());
}

//数据库启动时，在100-108设置随机字节
private static void setVcOpen(byte[] raw) {
    //LEN_VC 8  OF_VC 100
    System.arraycopy(RandomUtil.randomBytes(LEN_VC), 0, raw, OF_VC, LEN_VC);
}
```

关闭时拷贝字节：

```java
public static void setVcClose(Page pg) {
    pg.setDirty(true);
    setVcClose(pg.getData());
}

private static void setVcClose(byte[] raw) {
    System.arraycopy(raw, OF_VC, raw, OF_VC+LEN_VC, LEN_VC);
}
```

校验字节：

```java
public static boolean checkVc(Page pg) {
    return checkVc(pg.getData());
}

private static boolean checkVc(byte[] raw) {
    return Arrays.equals(Arrays.copyOfRange(raw, OF_VC, OF_VC+LEN_VC), Arrays.copyOfRange(raw, OF_VC+LEN_VC, OF_VC+2*LEN_VC));
}
```

每次打开DM文件都会首先校验第一页，如果校验不成功，再通过日志进行恢复。

```java
if(!dm.loadCheckPageOne()) {
     Recover.recover(tm, lg, pc);
}

// 校验PageOne
boolean loadCheckPageOne() {
     try {
            pageOne = pc.getPage(1);
     } catch (Exception e) {
            Panic.panic(e);
     }
     return PageOne.checkVc(pageOne);
}
```

#### 测试代码

```java
public class test {
    public static void main(String[] args) throws Exception {
        //初始化给了16页
        PageCacheImpl page=PageCache.create("cun/page",1 << 17);
        Page pg=page.getForCache(1);
        PageOne.setVcOpen(pg);
        PageOne.setVcClose(pg);
        //校验第一页
        boolean flag=PageOne.checkVc(pg);
        System.out.println(flag);
    }
}
```

### 普通页 PageX

（之前没文档误将FSO当成Data的开始位置，看好久好不懂，结果它是Data的结束位置）

一个普通页面以一个 2 字节无符号数起始，表示这一页指针的偏移（即这一页写到了哪个位置）。剩下的部分都是实际存储的数据。（其中数据Page.data()是以DataItem的格式存储的，后续会介绍到）

> PageX是解析管理普通Page的，本身不存储数据。

所以对普通页的管理，基本都是围绕着对 FSO（Free Space Offset）进行的。例如向页面插入数据：

将raw插入pg中，会返回指针偏移，这个表示数据写之前指针的位置。

```java
public static short insert(Page pg, byte[] raw) {
    pg.setDirty(true);
    short offset = getFSO(pg.getData());//offset新的插入点该在哪个位置
    System.arraycopy(raw, 0, pg.getData(), offset, raw.length);
    setFSO(pg.getData(), (short)(offset + raw.length));
    return offset;
}
```



在写入之前获取 FSO，来确定写入的位置，并在写入之后更新 FSO。FSO 的操作如下：

```java
//raw要写入的字节数组，ofData偏移量,把偏移量写入到raw字节数组的前两个字节
private static void setFSO(byte[] raw, short ofData) {
    System.arraycopy(Parser.short2Byte(ofData), 0, raw, OF_FREE, OF_DATA);
}

// 获取pg的FSO
public static short getFSO(Page pg) {
    return getFSO(pg.getData());
}

private static short getFSO(byte[] raw) {
    return Parser.parseShort(Arrays.copyOfRange(raw, 0, 2));
}

// 获取页面的空闲空间大小
public static int getFreeSpace(Page pg) {
    return PageCache.PAGE_SIZE - (int)getFSO(pg.getData());
}
```



#### 测试代码

```java
public class test {
    public static void main(String[] args) throws Exception {
        byte a[]=new byte[10];
        int pagX1=pc.newPage(a);
        System.out.println("写入该数据的页号"+pagX1);
        //从数据库中拿取一页普通页，继续填写
        Page pgx=pc.getPage(pagX1);
        byte[]b=new byte[128];

        long offset=PageX.insert(pgx,b);
        System.out.println("该页写入时的偏移量"+offset);
        System.out.println("该页写完时的偏移量"+PageX.getFSO(pgx));

        offset=PageX.insert(pgx,b);
        System.out.println("该页写入时的偏移量"+offset);
        System.out.println("该页写完时的偏移量"+PageX.getFSO(pgx));

        }

}
```



#### 根据日志进行恢复

剩余两个函数 recoverInsert() 和 recoverUpdate() 用于在数据库崩溃后重新打开时（即第一页的校验不对时），恢复例程直接插入数据以及修改数据使用。

PageX中两个方法的直接效果：

- recoverInsert：将给定的字节数组向给定的页从给定的位置开始插入（覆盖），可能更新FSO
- recoverUpdate：将给定的字节数组向给定的页从给定的位置开始插入（覆盖），不会更新FSO

这两个方法主要服务于`Recover`类，用于事务操作，做到类似于`mysql`里面的`redo`、`undo`效果

```java
//根据日志恢复插入操作
//将raw插入pg中的offset位置，并将pg的offset设置为较大的offset
public static void recoverInsert(Page pg, byte[] raw, short offset) {
    pg.setDirty(true);
    System.arraycopy(raw, 0, pg.getData(), offset, raw.length);
    //未插入之前的偏移量
    short rawFSO = getFSO(pg.getData());
    if(rawFSO < offset + raw.length) {
        setFSO(pg.getData(), (short)(offset+raw.length));
    }
}
//根据日志恢复更新操作
// 将raw插入pg中的offset位置，不更新offset,因为oldraw和newraw的大小相等
public static void recoverUpdate(Page pg, byte[] raw, short offset) {
    pg.setDirty(true);
    System.arraycopy(raw, 0, pg.getData(), offset, raw.length);
}
```



##### 测试代码

```java
short offset=PageX.insert(pgx,b);//插入前的偏移量
System.out.println("该页恢复更新前的偏移量"+PageX.getFSO(pgx));
byte[]c=new byte[256];
PageX.recoverUpdate(pgx,c,PageX.getFSO(pgx));
System.out.println("该页恢复更新后的偏移量"+PageX.getFSO(pgx));
```

