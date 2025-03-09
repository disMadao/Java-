# VM模块-死锁检测与VM的实现

[toc]



## 死锁

### 死锁的造成

在mysql中，两阶段锁协议（2PL）通常包括扩张和收缩两个阶段。在扩张阶段，事务将获取锁，但不能释放任何锁。在收缩阶段，可以释放现有的锁，但不能获取新的锁，这种规定存在着死锁的风险。

例如：当T1’在扩张阶段，获取了Y的读锁，并读取了Y，此时想要去获取X的写锁，却发现T2’的读锁锁定了X，而T2’也想要获取Y的写锁。简而言之，T1’不得到X是不会释放Y的，T2’不得到Y也是不会释放X的，这便陷入了循环，便形成了死锁。

### 死锁检测

前面提到了 2PL 会阻塞事务，直至持有锁的线程释放锁。可以将这种等待关系抽象成有向边，例如 Tj 在等待 Ti，就可以表示为 Tj --> Ti。这样，无数有向边就可以形成一个图（不一定是连通图）。检测死锁也就简单了，只需要查看这个图中是否有环即可。

创建一个 LockTable 对象，在内存中维护这张图。维护结构如下：

```java
public class LockTable {
private Map<Long, List<Long>> x2u;  // 某个XID已经获得的资源的UID列表
private Map<Long, Long> u2x;        // UID被某个XID持有
private Map<Long, List<Long>> wait; // 正在等待UID的XID列表
private Map<Long, Lock> waitLock;   // 正在等待资源的XID的锁
private Map<Long, Long> waitU;      // XID正在等待的UID
private Lock lock;
 
...
}
```
在每次出现等待的情况时，就尝试向图中增加一条边，并进行死锁检测。如果检测到死锁，就撤销这条边，不允许添加，并撤销该事务。add方法是用于判断事务xid能否获得资源uid的方法。

```java
// 不需要等待则返回null，否则返回锁对象
// 会造成死锁则抛出异常
public Lock add(long xid, long uid) throws Exception {
    lock.lock();
    try {
        //xid已经获得的资源的UID列表，如果在这个列表里面，则不造成死锁，也不需要等待
        if(isInList(x2u, xid, uid)) {//x2u xid list<uid>
            return null;
        }
        //这里表示有了一个新的uid，则把uid加入到u2x和x2u里面，不死锁，不等待
        //u2x  uid没有被某个xid占有
        if(!u2x.containsKey(uid)) {//uid xid
            u2x.put(uid, xid);
            putIntoList(x2u, xid, uid);
            return null;
        }
        //以下就是需要等待的情况
        //多个事务等待一个uid的释放
        waitU.put(xid, uid);//把需要等待uid的xid添加到等待列表里面
        //System.out.println("waitU"+waitU);
        putIntoList(wait, uid, xid);//uid list<xid> 正在等待uid的xid列表
        //造成死锁
        if(hasDeadLock()) {
            //从等待列表里面删除
            waitU.remove(xid);
            removeFromList(wait, uid, xid);
            throw Error.DeadlockException;
        }
        //没有造成死锁，但是需要等待新的xid获得锁
        Lock l = new ReentrantLock();
        l.lock();
        waitLock.put(xid, l);
        return l;
    } finally {
        lock.unlock();
    }
}
```

调用 add，如果不需要等待则进行下一步，如果需要等待的话，会返回一个上了锁的 Lock 对象。调用方在获取到该对象时，需要尝试获取该对象的锁，由此实现阻塞线程的目的，如下。

```java
Lock l = lt.add(xid, uid);
if(l != null) {
    l.lock();   // 其他阻塞在这一步
    l.unlock();
}
```

### 如何判断死锁

查找图中是否有环的算法实际上就是一个深搜，只是需要注意这个图不一定是连通图。思路就是为每个节点设置一个访问戳，都初始化为 1，随后遍历所有节点，以每个非 1 的节点作为根进行深搜，并将深搜该连通图中遇到的所有节点都设置为同一个数字，不同的连通图数字不同。这样，如果在遍历某个图时，遇到了之前遍历过的节点，说明出现了环。

> 和Tarjan算法有点相似的地方。

解析下面的dfs(xid)函数

> 这是一个判断事务之间是否有环的函数。
>
> dfs遍历从xid这个事务开始的有`关联`的事务，每次遍历都是同一个时间戳，注意以下几点：
>
> - 下次dfs的时间戳一定比上次大
> - 这种dfs遍历每次遍历的子图，可能不是极大的，因为只dfs一遍
> - 如何定义`关联`：事务a需要资源u，但u被事务b持有，则有a到b的有向边，和OS里面的死锁原理里的资源图一样。这也是这个函数能判断死锁的原理。因为每类资源只有一个，所以只要有环就必定无法化简
>
> dfs遍历到一个事务，如何判断这个事务是否之前出现过，有三种情况：
>
> - 它有时间戳，并且和此次dfs的时间戳相同，则出现了环。
> - 它有时间戳，但小于此次dfs的时间戳，则遍历到了之前的子图，无环
> - 它没有时间戳，则递归dfs遍历
>
> 注意：
>
> - 虽然用的时间戳，但dfs判有向图环的问题，只要区分三个状态：未遍历，之前遍历的子图里的点，正在遍历的点。所以算法题中常常是只用一个int数组，不能只用一个bool数组，用时间戳是一个比较合适的决定。如果不用时间戳，定义静态的标识0、1、2，就要在进入递归前将节点设置成`正在遍历状态`，而在递归退出前将节点设置成`之前遍历过状态`

实现如下：

```java
private boolean hasDeadLock() {
    xidStamp = new HashMap<>();
    stamp = 1;
   // System.out.println("xid已经持有哪些uid x2u="+x2u);//xid已经持有哪些uid
    // System.out.println("uid正在被哪个xid占用 u2x="+u2x);//uid正在被哪个xid占用
    for(long xid : x2u.keySet()) {//已经拿到锁的xid
        Integer s = xidStamp.get(xid);
        if(s != null && s > 0) {
            continue;
        }
        stamp ++;
        if(dfs(xid)) {
            return true;
        }
    }
    return false;
}

private boolean dfs(long xid) {
    Integer stp = xidStamp.get(xid);
    System.out.println("xid"+xid+"的stamp是"+stp);//stamp 时间戳
    //遍历某个图时，遇到了之前遍历过的节点，说明出现了环。
    if(stp != null && stp == stamp) {
        return true;
    }
    //stp != stamp 一定是stp <stamp
    if(stp != null && stp < stamp) {//只是到这个点的第’二‘条单向更长路径而已，必定是更长
        System.out.println("遇到了前一个图，未成环");
        return false;
    }

    xidStamp.put(xid, stamp);//每个已获得资源的事务一个独特的stamp
    Long uid = waitU.get(xid);//已获得资源的事务xid正在等待的uid
    System.out.println("xid"+xid+"正在等待的uid是"+uid);
    if(uid == null){
        System.out.println("未成环，退出深搜");
        return false;//xid没有需要等待的uid,无死锁
    }

    Long x = u2x.get(uid);//xid需要等待的uid被哪个xid占用了
    assert x != null;
    return dfs(x);
}
```

### 测试代码

```java
public static void main(String[] args) throws Exception {
        LockTable lock=new LockTable();
        lock.add(1L,3L);
        lock.add(2L,4L);
        lock.add(3L,5L);
        lock.add(1L,4L);
//        lock.add(1L,4L);
        System.out.println("++++++++++++++++");
//        lock.add(3L,4L);
        lock.add(2L,5L);
        System.out.println("++++++++++++++++");
        lock.add(3L,3L);
        System.out.println(hasDeadLock());
    }
```



## VM其他实现方法

### 获取缓存

VM 的实现类还被设计为 Entry 的缓存，需要继承 前面讲的通用缓存框架。实际上，就是在获取到dataItem的基础上再获取Entry。

```java
protected Entry getForCache(long uid) throws Exception {
    Entry entry = Entry.loadEntry(this, uid);
    if(entry == null) {
        throw Error.NullEntryException;
    }
    return entry;
}
public static Entry loadEntry(VersionManager vm, long uid) throws Exception {
    DataItem di = ((VersionManagerImpl)vm).dm.read(uid);
    return newEntry(vm, di, uid);
}
```

释放缓存也是如此，不再赘诉

### 从图中删除事务 (LockTable)

在一个事务 commit 或者 abort 时，就可以释放所有它持有的锁，并将自身从等待图中删除。并从等待队列中选择一个xid来占用uid
解锁时，将该 Lock 对象 unlock 即可，这样其他业务线程就获取到了锁，就可以继续执行了。

```java
public void remove(long xid) {
    lock.lock();
    try {
        List<Long> l = x2u.get(xid);
        if(l != null) {
            while(l.size() > 0) {
                Long uid = l.remove(0);
                //从等待列表中选择一个事务占用这个资源
                selectNewXID(uid);
            }
        }
        waitU.remove(xid);
        x2u.remove(xid);
        waitLock.remove(xid);
    } finally {
        lock.unlock();
    }
}
```



```java
private void selectNewXID(long uid) {
    u2x.remove(uid);//uid被某个xid持有
    List<Long> l = wait.get(uid);//正在等待uid的xid的列表
    if(l == null) return;
    assert l.size() > 0;
    while(l.size() > 0) {
        //l中的事务都需要这个资源，直接删除，有的不行是因为有的事务从waitLock中删除了，但没从wait中删除（回看上面的remove方法）
        // 所以l中有些`无用的空事务`
        long xid = l.remove(0);
        if(!waitLock.containsKey(xid)) {//这是个 空事务
            continue;
        } else {//这个事务有锁，证明是在阻塞态。事务被阻塞，只会因为一个资源，不会因为多个。
            u2x.put(uid, xid);
            Lock lo = waitLock.remove(xid);
            waitU.remove(xid);
            lo.unlock();
            break;
        }
    }

    if(l.size() == 0) wait.remove(uid);
}
```

### 开启事务

begin() 开启一个事务，并初始化事务的结构，将其存放在 activeTransaction 中，用于检查和快照使用：

```java
//begin() 每开启一个事务，并计算当前活跃的事务的结构，将其存放在 activeTransaction 中，
// 用于检查和快照使用：
@Override
public long begin(int level) {
    lock.lock();
    try {
        long xid = tm.begin();
        //activeTransaction 当前事务创建时活跃的事务,，如果level!=0,放入t的快照中
        Transaction t = Transaction.newTransaction(xid, level, activeTransaction);
        activeTransaction.put(xid, t);
        return xid;
    } finally {
        lock.unlock();
    }
}
```

### 提交事务

commit() 方法提交一个事务，主要就是 free 掉相关的结构，并且释放持有的锁，并修改 TM 状态：

```java
//commit() 方法提交一个事务，主要就是 free 掉相关的结构，并且释放持有的锁，并修改 TM 状态：
@Override
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

### 回滚事务

abort 事务的方法则有两种，手动和自动。手动指的是调用 abort() 方法，而自动，则是在事务被检测出出现死锁时，会自动撤销回滚事务；或者出现版本跳跃时，也会自动回滚：

```java
private void internAbort(long xid, boolean autoAborted) {
    lock.lock();
    Transaction t = activeTransaction.get(xid);
    //手动回滚
    if(!autoAborted) {
        activeTransaction.remove(xid);
    }
    lock.unlock();
    //自动回滚
    if(t.autoAborted) return;
    lt.remove(xid);
    tm.abort(xid);
}
```

### 读取数据

read() 方法读取一个 entry，注意判断下可见性即可：

```java
//read() 方法读取一个 entry，注意判断下可见性即可
//xid当前事务   uid要读取的记录
@Override
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
        //通过uid找要读取的记录Entry
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

### 插入数据

```java
//insert() 则是将数据包裹成 Entry，无脑交给 DM 插入即可：
@Override
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

### 删除版本

delete() 方法看起来略为复杂，实际上主要是前置的三件事：一是可见性判断，二是获取资源的锁，三是版本跳跃判断。删除的操作只有一个设置 XMAX。

**表里面的更新操作实际是先删除再插入的，所以只要在这里获取数据的锁即可。**

```java
public boolean delete(long xid, long uid) throws Exception {
    lock.lock();
    Transaction t = activeTransaction.get(xid);
    lock.unlock();
    if(t.err != null) {
        throw t.err;
    }
    Entry entry = null;
    try {
        entry = super.get(uid);
    } catch(Exception e) {
        if(e == Error.NullEntryException) {
            return false;
        } else {
            throw e;
        }
    }
    try {
        if(!Visibility.isVisible(tm, t, entry)) {
            return false;
        }
        Lock l = null;
        try {
            l = lt.add(xid, uid);
        } catch(Exception e) {
            //检测到死锁
            t.err = Error.ConcurrentUpdateException;
            internAbort(xid, true);
            t.autoAborted = true;
            throw t.err;
        }
        //没有死锁，但是需要等待当前事务拿到锁
        if(l != null) {
            l.lock();
            l.unlock();
        }
        if(entry.getXmax() == xid) {
            return false;
        }
        //System.out.println(entry.getXmax());
        //检查到版本跳跃，回滚事务
        if(Visibility.isVersionSkip(tm, t, entry)) {
           // System.out.println("检查到版本跳跃，自动回滚");
            t.err = Error.ConcurrentUpdateException;
            internAbort(xid, true);
            t.autoAborted = true;
            throw t.err;
        }
        //删除事务，把当前xid设置为Xmax
        entry.setXmax(xid);
        return true;

    } finally {
        entry.release();
    }
}
```



