# VM模块-记录的版本与事务隔离

[toc]

## VM模块介绍

VM 模块是事务和数据版本的控制中心。其实现了 MVCC 以消除读写阻塞。同时实现了两种隔离级别。

 

DM 层向上层提供了数据项（Data Item）的概念，VM 通过管理所有的数据项，向上层提供了记录（Entry）的概念。上层模块通过 VM 操作数据的最小单位，就是记录。VM 则在其内部，为每个记录，维护了多个版本（Version）。每当上层模块对某个记录进行修改时，VM 就会为这个记录创建一个新的版本。

**就是将DataItem再封装一层，变成Entry。**

> DataItem是更底层的数据结构，Entry是向上层提供的。

![image-20250110011425051](.\assets\image-20250110011425051.png)

所以，VM模块是基于TM和DM模块基础上的。

在第四章中，为了保证数据的可恢复，VM 层传递到 DM 的操作序列需要满足以下两个规则：

> 规定1：正在进行的事务，不会读取其他任何未提交的事务产生的数据。
> 规定2：正在进行的事务，不会修改其他任何未提交的事务修改或产生的数据。

由于 2PL 和 MVCC，我们可以看到，这两个条件都被很轻易地满足了。

## 相关知识回顾

 如果数据库中的事务都是串行执行的，这种方式可以保障事务的执行不会出现异常和错误，但带来的问题是串行执行会带来性能瓶颈；
而事务并发执行，如果不加以控制则会引发诸多问题，包括死锁、更新丢失等等。

### 脏读、不可重复读、幻读

首先来定义数据库的冲突，暂时不考虑插入操作，只看更新操作（U）和读操作（R），两个操作只要满足下面三个条件，就可以说这两个操作相互冲突

```java
这两个操作是由不同的事务执行的
这两个操作操作的是同一个数据项
这两个操作至少有一个是更新操作
```

并发事务带来的问题
读-读：即并发事务相继读取同一记录；
因为读取记录并不会对记录造成任何影响，所以同个事务并发读取同一记录也就不存在任何安全问题，所以允许这种操作。

写-写；即并发事务相继对同一记录做出修改；
如果允许并发事务都读取同一记录，并相继基于旧估对这一记录做出修改，那么就会出现前一个事务所做的修改被后面事务的修改覆盖，即出现提交覆盖的问题。

另外一种情况，并发事务相继对同一记录做出修改，其中一个事务提交之后之后另一个事务发生回滚，这样就会出现已提交的修改因为回滚而丢失的问题，即回滚覆盖问题

写-读或读-写：即两个并发事务对同一记录分别进行读操作和写操作。
如果一个事务读取了另一个事务尚未提交的修政记录，那么就出现了脏读的问题；

如果我们加以控制使得一个事务只能读取其他已提交事务的修改的数据，那么这个事务在另一个事务提交修改前后读取到的数据是不一样的，这就意味看发生了不可重复读；

如果一个事务根据一些条件查询到一些记录，之后另一事物向表中插入了一些记录，原先的事务以相同条件再次查询时发现得到的结果跟第一次查词得到的结果不一致，这就意味着发生了幻读。

==不可重复读和幻读的区别?==
不可重复读的重点是修改:
同样的条件, 你读取过的数据, 再次读取出来发现值不一样了

幻读的重点在于新增或者删除
同样的条件, 第1次和第2次读出来的**记录数**不一样

当然, 从总的结果来看, 似乎两者都表现为两次读取的结果不一致.
**但如果你从控制的角度来看, 两者的区别就比较大**
对于前者, 只需要锁住满足条件的记录（行锁）
对于后者, 要锁住满足条件及其相近的记录（行锁+间隙锁）

### 2PL两段锁协议

把获取锁和释放锁分为两个不同的阶段的协议称为**两阶段锁协议**（2-phase locking）。
两阶段锁协议规定:

在加锁阶段，一个事务可以获得锁但是不能释放锁；
在解锁阶段事务只可以释放锁，并不能获得新的锁。

> 就是加锁解锁只分为两段，不能加锁解锁多段穿插着进行。

两段锁协议只是保证了并发事务的可串行化，不能保证不会发生脏读！例子：

```sql
TxnB XLock(x)
TxnB Write(x)
TxnB XUnlock(x)
TxnA SLock(x)
TxnA Read(x)
TxnA SUnlock(x)
TxnA Commit
TxnB Abort
```

从上面的代码知道是因为写锁在提交（或终止）之前就释放了。

解决这个问题，涉及2pl最常见的一个变体——==Strict 2PL==，**说的是事务的写锁只有在事务结束的时候才会释放，而不是在2PL的放锁阶段。**

> ==我看完代码了，应该是没有实现2pl的。==
>
> 只有dataitem的before和after里面是对数据的并发控制的加锁。
>
> 但是在VM实现类里面的setMax方法里面是既有dataitem.before()，也有dateitiem.after（）。
>
> strcuct-2pl应该是只有在事务提交或者终止的时候才会释放锁的，但项目这两方法里面没有释放锁的代码。

两阶段锁协议能够保证事务可串行化执行，解决事务并发问题，不可避免地导致了事务间的相互阻塞，甚至可能导致死锁。为了提高事务处理的效率，降低阻塞概率，实现了 MVCC。

### MVCC

MySQL 是通过MVCC（多版本并发控制）来实现读-写并发控制，又是通过两阶段锁来实现写-写并发控制的。即 MVCC+2PL 实现

==MySQL 通过 MVCC，降低了事务的阻塞概率。==譬如，T1 想要更新记录 X 的值，于是 T1 需要首先获取 X 的锁，接着更新，也就是创建了一个新的 X 的版本，假设为 x3。假设 T1 还没有释放 X 的锁时，T2 想要读取 X 的值，这时候就不会阻塞，MYDB 会返回一个较老版本的 X，例如 x2。这样最后执行的结果，就等价于，T2 先执行，T1 后执行，调度序列依然是可串行化的。如果 X 没有一个更老的版本，那只能等待 T1 释放锁了。所以只是降低了概率。

（这个项目MVCC的实现和MySQL中类似。）

## 记录的存储结构，Entry类

对于一条记录来说，Mysql使用 Entry 类维护了其结构，存储在dataItem中的data部分。虽然理论上，MVCC 实现了多版本，但是在实现中，VM模块 并没有提供 Update 操作，对于字段的更新操作由后面章节介绍的表和字段管理（TBM模块）实现。**所以在 VM 的实现中，一条记录只有一个版本**。

> 更新是通过先删除（覆盖版本号）再插入实现的。

我们规定，一条 Entry 中存储的数据格式如下：

```java
[XMIN]   [XMAX]    [data]
 8个字节  8个字节
```

XMIN：创建该版本的事务编号
XMAX：删除（覆盖）该版本的事务编号
DATA 就是这条记录持有的数据(即DM模块封装)，根据这个结构，在创建记录时调用的 wrapEntryRaw() 方法如下

```java
//创建记录
public static byte[] wrapEntryRaw(long xid, byte[] data) {
    byte[] xmin = Parser.long2Byte(xid);
    byte[] xmax = new byte[8];
    return Bytes.concat(xmin, xmax, data);
}
```

同样，如果要获取记录中持有的数据，也就需要按照这个结构来解析：

```java
// 以拷贝的形式返回内容
public byte[] data() {
    dataItem.rLock();
    try {
        SubArray sa = dataItem.data();
        byte[] data = new byte[sa.end - sa.start - OF_DATA];
        System.arraycopy(sa.raw, sa.start+OF_DATA, data, 0, data.length);
        return data;
    } finally {
        dataItem.rUnLock();
    }
}
```

## VM类主要相关操作

### 删除版本号

这里以拷贝的形式返回数据，如果需要修改的话，需要对 DataItem 执行 before() 方法，这个在设置 XMAX 的值中体现了，

这样，这个版本对每一个 XMAX 之后的事务都是不可见的，也就等价于删除了

```java
public void setXmax(long xid) {
    dataItem.before();
    try {
        SubArray sa = dataItem.data();
        System.arraycopy(Parser.long2Byte(xid), 0, sa.raw, sa.start+OF_XMAX, 8);
    } finally {
        dataItem.after(xid);
    }
}
```

before() 和 after() 是在 DataItem 一节中就已经确定的数据项修改规则。

### 开启事务

begin() 每开启一个事务，计算当前活跃的事务的结构，将其存放在 activeTransaction 中，level是隔离级别，0代表读已提交，1代表可重复读。

```java
public long begin(int level) {
    lock.lock();
    try {
        long xid = tm.begin();
        //xid 隔离级别  当前事务创建时活跃的事务，这类似快照Read View
        Transaction t = Transaction.newTransaction(xid, level, activeTransaction);
        activeTransaction.put(xid, t);
        return xid;
    } finally {
        lock.unlock();
    }
}
```

### 提交事务

commit() 方法提交一个事务，主要就是 free 掉相关的结构，并且释放持有的锁，并修改 TM 状态，并把该事务从activeTransaction中移除。

```java
public void commit(long xid) throws Exception {
    lock.lock();
    Transaction t = activeTransaction.get(xid);
    lock.unlock();
    try {
        if(t.err != null) {
            throw t.err;
        }
    } catch(NullPointerException n) {
        Panic.panic(n);
    }

    lock.lock();
    activeTransaction.remove(xid);
    lock.unlock();

    lt.remove(xid);
    tm.commit(xid);
}
```

### 插入数据

insert() 则是将数据包裹成 Entry，不是插入原有的dateitem，无脑交给 DM 插入即可：

```java
public long insert(long xid, byte[] data) throws Exception {
    lock.lock();
    //xid插入时还活跃的快照
    Transaction t = activeTransaction.get(xid);
    lock.unlock();
    if(t.err != null) {
        throw t.err;
    }
    byte[] raw = Entry.wrapEntryRaw(xid, data);
    return dm.insert(xid, raw);
}
```

### 读取数据

read() 方法读取一个 entry，根据隔离级别判断下可见性即可。

关于读取逻辑：

> 和DM层的逻辑类似。
>
> 先读取父类AbstractCache<Entry>中的缓存，没有的话就丢给DM层read方法。

```java
//xid 是事务，Uid 是数据
public byte[] read(long xid, long uid) throws Exception {
    lock.lock();
    //当前事务xid读取时的快照数据
    Transaction t = activeTransaction.get(xid);
    lock.unlock();
    if(t.err != null) {
        throw t.err;
    }

    Entry entry = null;
    try {
        //通过uid找要读取的事务dataItem
        entry = super.get(uid);
    } catch(Exception e) {
        if(e == Error.NullEntryException) {
            return null;
        } else {
            throw e;
        }
    }
    try {
        if(Visibility.isVisible(tm, t, entry)) {
            return entry.data();
        } else {
            return null;
        }
    } finally {
        entry.release();
    }
}
```

## 隔离级别的实现 Visibility

上面提到，如果一个记录的最新版本被加锁，当另一个事务想要修改或读取这条记录时，MYDB 就会返回一个较旧的版本的数据。这时就可以认为，最新的被加锁的版本，对于另一个事务来说，是不可见的。于是版本可见性的概念就诞生了。

### 事务隔离级别

`可重复读`，这是`mysql`的默认事务隔离级别。

当前项目支持两种隔离级别：读已提交和可重复读。可通过设置Transaction的level字段调整。0代表读已提交，1代表可重复读。

SQL定义的四种隔离级别如下，MySQL全部支持，需要自己设定，回见上面的VM里的开启事务那节。

| 隔离级别 | 脏读   | 不可重复读 | 幻读   |
| -------- | ------ | ---------- | ------ |
| 读未提交 | 可能   | 可能       | 可能   |
| 读提交   | 不可能 | 可能       | 可能   |
| 可重复读 | 不可能 | 不可能     | 可能   |
| 串行化   | 不可能 | 不可能     | 不可能 |



### 读已提交

即事务在读取数据时, ==只能读取已经提交事务产生的数据==。

X**MAX 这个变量，也就解释了为什么 DM 层不提供删除操作，当想删除一个版本时，只需要设置其 XMAX，这样，这个版本对每一个 XMAX 之后的事务都是不可见的，也就等价于删除了。** 

为了得到版本链实现MVCC，不能真的删除一个版本。

#### 解决思路

如此，在读提交下，**版本对事务的可见性**逻辑如下：

版本A对事务Ti可见的逻辑：版本A由事务Ti本身创建且尚未删除，或版本A由其他事务（这个事务并不要求在Ti之前）创建，且版本A并未被比Ti或Ti更早的事务删除。

(XMIN == Ti and                             // 由Ti创建且
    XMAX == NULL                            // 还未被删除
)
or                                          // 或
(XMIN is commited and                       // 由一个已提交的事务创建且
    (XMAX == NULL or                        // 尚未删除或
    (XMAX != Ti and XMAX is not commited)   // 由一个未提交的事务删除
))
若条件为 true，则版本对 Ti 可见。那么获取 Ti 适合的版本，只需要从最新版本开始，依次向前检查可见性，如果为 true，就可以直接返回。

**以下方法判断某个记录e对事务 t 是否可见：**

```java
private static boolean readCommitted(TransactionManager tm, Transaction t, Entry e) {
    long xid = t.xid;
    long xmin = e.getXmin();
    long xmax = e.getXmax();
    if(xmin == xid && xmax == 0) return true;
    if(tm.isCommitted(xmin)) {
        if(xmax == 0) return true;
        if(xmax != xid) {
            if(!tm.isCommitted(xmax)) {
            //由一个未提交事务覆盖，表明最新数据还未提交不能读，可以读这个次新
                return true;
            }
        }
    }
    return false;
}
```

#### 测试代码

```java
public class rc {
    public static void main(String[] args) throws Exception {
        TransactionManagerImpl tm= TransactionManager.create("cun/tm");
        DataManager dm=DataManager.create("cun/dm",1 << 20,tm);
        VersionManager vm=VersionManager.newVersionManager(tm,dm);
        byte b[]=new byte[1122];
        long xid1=vm.begin(0);
        long uid1=vm.insert(xid1,b);
        //vm.commit(xid1);
        long xid2=vm.begin(0);
        byte[] output=vm.read(xid2,uid1);
        System.out.println(output);
    }
}
```

### 可重复读

不可重复度，会导致一个事务在执行期间对同一个数据项的读取得到不同结果。如下面的结果，加入 X 初始值为 0：

T1 begin
R1(X) // T1 读得 0
T2 begin
U2(X) // 将 X 修改为 1
T2 commit
R1(X) // T1 读的 1
可以看到，T1 两次读 X，读到的结果不一样。如果想要避免这个情况，就需要引入更严格的隔离级别，即可重复读

####  解决思路

T1 在第二次读取的时候，读到了已经提交的 T2 修改的值（这个修改事务为原版本的XMAX，为新版本的XMIN），导致了这个问题。于是我们可以规定：

==事务只能读取它`开始时`, 就已经结束的那些事务产生的数据版本==
这条规定，增加于，事务需要忽略：

- 在本事务后开始的事务的数据版本;

- 本事务开始时还是 active 状态的事务的数据版本

对于第一条，只需要比较事务 ID，即可确定。而对于第二条，则需要在事务 Ti 开始时，记录下当前活跃的所有事务 SP(Ti)，如果记录的某个版本，XMIN 在 SP(Ti) 中，也应当对 Ti 不可见。

于是，可重复读的判断逻辑如下：

**版本A对事务Ti可见的逻辑**：版本A由事务Ti本身创建且尚未删除，或版本A由其他事务（这个创建的事务并必须要求在Ti之前且要处于提交状态）创建，且版本A并未被比Ti或Ti更早的事务删除。

(XMIN == Ti and                 // 由Ti创建且
 (XMAX == NULL or               // 尚未被删除
))
or                              // 或
(XMIN is commited and           // 由一个已提交的事务创建且
 XMIN < XID and                 // 这个事务小于Ti且
 XMIN is not in SP(Ti) and      // 这个事务在Ti开始前提交且
 (XMAX == NULL or               // 尚未被删除或
  (XMAX != Ti and               // 由其他事务删除但是
   (XMAX is not commited or     // 这个事务尚未提交或
XMAX > Ti or                    // 这个事务在Ti开始之后才开始或
XMAX is in SP(Ti)               // 这个事务在Ti开始前还未提交
))))
于是，需要提供一个结构，来抽象一个事务，以保存快照数据（该事务创建时还活跃着的事务）：

```java
public class Transaction {
    public long xid;
    public int level;
    public Map<Long, Boolean> snapshot;
    public Exception err;
    public boolean autoAborted;
    //事务id  隔离级别  快照
    public static Transaction newTransaction(long xid, int level, Map<Long, Transaction> active) {
        Transaction t = new Transaction();
        t.xid = xid;
        t.level = level;
        if(level != 0) {//隔离级别为可重复读,读已提交不需要快照信息
            t.snapshot = new HashMap<>();
            for(Long x : active.keySet()) {
                t.snapshot.put(x, true);
            }
        }
        return t;
    }

    public boolean isInSnapshot(long xid) {
        if(xid == TransactionManagerImpl.SUPER_XID) {
            return false;
        }
        return snapshot.containsKey(xid);
    }
}
```


构造方法中的 active，保存着当前所有 active 的事务。于是，可重复读的隔离级别下，一个版本是否对事务可见的判断如下：

```java
private static boolean repeatableRead(TransactionManager tm, Transaction t, Entry e) {
    long xid = t.xid;
    long xmin = e.getXmin();
    long xmax = e.getXmax();
    if(xmin == xid && xmax == 0) return true;

    //创建事务xmin在当前事务xid开始前已经提交
    if(tm.isCommitted(xmin) && xmin < xid && !t.isInSnapshot(xmin)) {
        //版本还未被删除
        if(xmax == 0) return true;
        //被其他事务删除
        if(xmax != xid) {
            //删除事务：xmax事务此时还未提交 or 在xid之后创建的 or xid创建时还活跃（快照）
            if(!tm.isCommitted(xmax) || xmax > xid || t.isInSnapshot(xmax)) {
                return true;
            }
        }
    }
    return false;

}
```

#### 测试代码

```java
public class rr {
    public static void main(String[] args) throws Exception {
        TransactionManagerImpl tm= TransactionManager.create("cun/tm");
        DataManager dm=DataManager.create("cun/dm",1 << 20,tm);
        VersionManager vm=VersionManager.newVersionManager(tm,dm);
        byte b[]=new byte[1122];
        long xid1=vm.begin(1);
        long xid2=vm.begin(1);
        long uid1=vm.insert(xid1,b);
        byte[] output=vm.read(xid2,uid1);
        System.out.println(output);
        vm.commit(xid1);
        output=vm.read(xid2,uid1);
        System.out.println(output);
    }
}
```



### 版本跳跃问题-可重复读要解决

版本跳跃问题，考虑如下的情况，假设 X 最初只有 x0 版本，T1 和 T2 都是可重复读的隔离级别：

```java
T1 begin
T2 begin
R1(X) // T1读取x0
R2(X) // T2读取x0
U1(X) // T1将X更新到x1
T1 commit
U2(X) // T2将X更新到x2
T2 commit
```

这种情况实际运行起来是没问题的，但是逻辑上不太正确。T1 将 X 从 x0 更新为了 x1，这是没错的。但是 T2 则是将 X 从 x0 更新成了 x2，跳过了 x1 版本。

读提交是允许版本跳跃的，==而可重复读则是不允许版本跳跃的==。

解决版本跳跃的思路：如果 Ti 需要修改 X，而 X 已经被 Ti 不可见的事务 Tj 修改了，那么要求 Ti 回滚。

MVCC 的实现，使得 撤销或是回滚事务时：只需要将这个事务标记为 aborted 即可。根据前一章提到的可见性，每个事务都只能看到其他 committed 的事务所产生的数据，一个 aborted 事务产生的数据，就不会对其他事务产生任何影响了，也就相当于，这个事务不曾存在过

```java
if(Visibility.isVersionSkip(tm, t, entry)) {
      System.out.println("检查到版本跳跃，自动回滚");
      t.err = Error.ConcurrentUpdateException;
      internAbort(xid, true);
      t.autoAborted = true;
      throw t.err;
}
```

上一节中就总结了，Ti 不可见的 Tj，有两种情况：

XID(Tj) > XID(Ti)          修改版本在Ti之后创建
Tj in SP(Ti)                  Ti创建时修改版本已经创建但是还未提交

于是版本跳跃的检查也就很简单了，取出要修改的数据 X 的最新提交版本，并检查该最新版本的创建者对当前事务是否可见：

```java
public static boolean isVersionSkip(TransactionManager tm, Transaction t, Entry e) {
    long xmax = e.getXmax();
    if(t.level == 0) {
        return false;
    } else {
        //结合起来就是事务xmax已提交，但不是在事务xid开始前提交的就有版本跳跃问题，不可见
        return tm.isCommitted(xmax) && (xmax > t.xid || t.isInSnapshot(xmax));
  }
}
```

####  测试代码

```java
public class skip {
    public static void main(String[] args) throws Exception {
        TransactionManagerImpl tm= TransactionManager.open("cun/tm");
        DataManager dm=DataManager.open("cun/dm",1 << 20,tm);
        VersionManager vm=VersionManager.newVersionManager(tm,dm);
        byte b[]=new byte[1122];
        byte b_update[]=new byte[1122];

        long uid1=vm.insert(0,b);
        long xid1=vm.begin(1);
        System.out.println("开启事务"+xid1);

        long xid2=vm.begin(1);
        System.out.println("开启事务"+xid2);
        //xid1将x更新到x1
        vm.delete(xid1,uid1);
        long uid2=vm.insert(xid1,b_update);
        vm.commit(xid1);
        System.out.println("======");
        //xid1将x更新到x2
        vm.delete(xid2,uid1);
        long uid3=vm.insert(xid2,b_update);
        vm.commit(xid2);
    }

}
```



### 小结

和MySQL原理差不多，Read View的四个字段有三个（最小、最大、活跃列表），只是没有组合成一个Read View数据结构。