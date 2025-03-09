# DM模块-日志文件与恢复文件

[toc]

承接上一章节的最后一部分，DM 层在每次对底层数据操作时，都会记录一条日志到磁盘上。在数据库奔溃之后（即第一页校验不对时），再次启动时，可以根据日志的内容，恢复数据文件，保证其一致性。

## 日志格式

```java
[XChecksum][Log1][Log2][Log3]...[LogN][BadTail]
4字节
```

其中 XChecksum 是一个四字节的整数，是对后面所有日志计算的校验和。Log1 ~ LogN 是常规的日志数据，BadTail 是在数据库崩溃时，没有来得及写完的日志数据，这个 BadTail 不一定存在。

每条日志 [LogN] 的格式如下：

```java
[Size][Checksum][Data]
4字节   4字节
```

其中，Size 是一个四字节整数，标识了 Data 段的字节数。Checksum 则是该条日志的校验和。

### 校验日志

单条日志的校验和，其实就是通过一个指定的种子实现的：

```java
private int calChecksum(int xCheck, byte[] log) {
    for (byte b : log) {
        xCheck = xCheck * SEED + b;
    }
    return xCheck;
}
```

对所有日志求出校验和，用一特殊方法 ’ 求和 ’ 就能得到日志文件的校验和了。

### 读取日志

#### next

Logger 被实现成迭代器模式，通过 next() 方法，不断地按照格式从文件中读取下一条日志，并将其中的 Data 解析出来并返回。next() 方法的实现主要依靠 internNext()，大致如下，其中 position 是当前日志文件读到的位置偏移：

```java
//通过 next() 方法，不断地从文件中读取下一条日志，并将日志文件的 Data 解析出来并返回。
//其中 position 是当前日志文件读到的位置偏移：
private byte[] internNext() {
    //[XChecksum] [Log1] [Log2]
    //[Size] [Checksum] [Data]
    if(position + OF_DATA >= fileSize) {//size>fileSize
        return null;
    }
    // 读取单条日志的size（data段的字节数）
    ByteBuffer tmp = ByteBuffer.allocate(4);
    try {
        fc.position(position);
        fc.read(tmp);
    } catch(IOException e) {
        Panic.panic(e);
    }
    int size = Parser.parseInt(tmp.array());
    if(position + size + OF_DATA > fileSize) {
        return null;
    }
    // 读取checksum+data
    ByteBuffer buf = ByteBuffer.allocate(OF_DATA + size);
    try {
        fc.position(position);
        fc.read(buf);
    } catch(IOException e) {
        Panic.panic(e);
    }
    //单条日志的checksum+data
    byte[] log = buf.array();
    // 一条日志根据存放的数据算出来的校验和
    int checkSum1 = calChecksum(0, Arrays.copyOfRange(log, OF_DATA, log.length));
    //一条日志的Checksum部分存的东西
    int checkSum2 = Parser.parseInt(Arrays.copyOfRange(log, OF_CHECKSUM, OF_DATA));
    if(checkSum1 != checkSum2) {//毁坏的日志文件
        return null;
    }
    position += log.length;
    return log;
}
```

#### 效验

在打开一个日志文件时，需要首先校验日志文件的 XChecksum，并移除文件尾部可能存在的 BadTail，由于 BadTail 该条日志尚未写入完成，文件的校验和也就不会包含该日志的校验和，去掉 BadTail 即可保证日志文件的一致性。

```java
private void checkAndRemoveTail() {
    rewind();//position=4
    int xCheck = 0;
    while(true) {
        //遍历日志
        byte[] log = internNext();
        if(log == null) break;
        xCheck = calChecksum(xCheck, log);
    }
    //xCheck 多条日志总的校验和
    if(xCheck != xChecksum) {
        Panic.panic(Error.BadLogFileException);
    }
    try {
        truncate(position);
    } catch (Exception e) {
        Panic.panic(e);
    }
    try {
        file.seek(position);
    } catch (IOException e) {
        Panic.panic(e);
    }
    rewind();
}
```

### 写入日志

向日志文件写入日志时，也是首先将数据包裹成日志格式，写入文件后，再更新文件的校验和，更新校验和时，会刷新缓冲区，保证内容写入磁盘。

```java
public void log(byte[] data) {
    byte[] log = wrapLog(data);
    ByteBuffer buf = ByteBuffer.wrap(log);
    lock.lock();
    try {
        fc.position(fc.size());
        fc.write(buf);
    } catch(IOException e) {
        Panic.panic(e);
    } finally {
        lock.unlock();
    }
    updateXChecksum(log);
}
```
更新文件的校验和，会将插入日志的校验值添加到原日志文件的校验值上。

```java
private void updateXChecksum(byte[] log) {
    this.xChecksum = calChecksum(this.xChecksum, log);
    try {
        fc.position(0);
        fc.write(ByteBuffer.wrap(Parser.int2Byte(xChecksum)));
        fc.force(false);
    } catch(IOException e) {
        Panic.panic(e);
    }
}
```

### 测试代码

```java
public class test {
    public static void main(String[] args) {
        Logger logger=Logger.create("log");
        byte b[]=new byte[128];
        logger.log(b);
        byte[] log=logger.next();
        logger.close();
    }
}
```

## 恢复策略

DM 为上层模块，提供了两种操作，分别是插入新数据（I）和更新现有数据（U）。至于为啥没有删除数据，这个会在 VM 一节叙述。

DM 的日志策略：
在进行 I 和 U 操作之前，必须先进行对应的日志操作，在保证日志写入磁盘后，才进行数据操作。

### 单线程条件下的恢复策略-日志redo、undo

==重做所有崩溃时已完成（committed 或 aborted）的事务==：redo

> aborted的事务应该不用重做，但代码里重做了，不懂这个设计。

==撤销所有崩溃时未完成（active）的事务==：undo

（看代码就知道通过isActive判断，不是由redoTranscations方法处理，是的由undoTranscations方法处理）
**在第一步，如何对事务 T 进行 redo？**

==正序==扫描事务 T 的所有日志
如果日志是插入操作 (Ti, I, A, x)，就将 x 重新插入 A 位置
如果日志是更新操作 (Ti, U, A, oldx, newx)，就将 A 位置的值设置为 newx

> 为什么要做redo日志？
>
> 这是MySQL的设计策略，日志先行，先写日志再说，然后异步持久化到硬盘（这项目没有实现异步持久化）。崩溃时可以恢复。
>

**在第二步，如何对事务 T 进行undo？**

==倒序==扫描事务 T 的所有日志
如果日志是插入操作 (Ti, I, A, x)，就将 A 位置的数据删除
如果日志是更新操作 (Ti, U, A, oldx, newx)，就将 A 位置的值设置为 oldx

> 为什么要做undo日志？
>
> 这也是MySQL的设计，里面需要的MVCC机制需要用起来回滚和多个行版本并发控制（MVCC）。
>

### 多线程下恢复策略

上述单线程恢复策略下会出现两种问题

情景一：事务T2读取到了事务T1未提交的数据，然后，事务T1提交

T1 begin
T2 begin
T2 U(x)
T1 R(x)
...
T1 commit
MYDB break down
**此时，数据库崩溃**，T2 仍然是活跃状态，那么当数据库重新启动，执行恢复例程时，会撤销 T2，它对数据库的影响会被消除。但是由于 T1 读取了 T2 更新的值，既然 T2 被撤销，那么 T1 也应当被撤销。这种情况，就是**级联回滚**。但是，T1 已经 commit 了，所有 commit 的事务的影响，应当被持久化。这里就造成了**逻辑上的矛盾**。

所以需要增加条件

**规定1：正在进行的事务，不会读取其他任何未提交的事务产生的数据。**（脏读）
情景二：事务T2修改了事务T1修改后但是并未提交的数据，然后，事务T2提交

x的初始值为0
T1 begin
T2 begin
T1 set x = x+1 // 产生的日志为(T1, U, A, 0, 1)
T2 set x = x+1 // 产生的日志为(T1, U, A, 1, 2)
T2 commit
MYDB break down
在系统崩溃时，T1 仍然是活跃状态。那么当数据库重新启动，执行恢复例程时，会对 T1 进行撤销，对 T2 进行重做，但是，

如果先撤销T1，再重做T2，x变为2

如果先重做T2，再撤销T1，x变为0

都不是1，都是错误的。

所以需要增加条件

**规定2：正在进行的事务，不会修改其他任何未提交的事务修改或产生的数据。**（丢失修改）
由于 VM层 的存在，传递到 DM 层，真正执行的操作序列，都可以保证规定 1 和规定 2，**所以在DM层无需另外的代码来保证这两个规定。**



### 恢复策略的代码实现

定义两种日志格式
 关于[LoggerN]的Data部分内容，定义两种日志格式

**插入型日志：**表示事务XID 在[Pgno] [Offset]位置插入了一条数据 Raw

insertLog:
[LogType] [XID] [Pgno] [Offset] [Raw]
**更新型日志：**表示事务XID 在[UID]位置从 OldRaw 更新成 NewRaw，这里的UID也是由Pgno和Offset通过移位计算出来的

updateLog:
[LogType] [XID] [UID] [OldRaw] [NewRaw]
LogType表示两种日志格式的日志类型

```java
private static final byte LOG_TYPE_INSERT = 0;
private static final byte LOG_TYPE_UPDATE = 1;
```

生成这两种日志格式的代码如下：

```java
public static byte[] updateLog(long xid, DataItem di) {
    byte[] logType = {LOG_TYPE_UPDATE};
    byte[] xidRaw = Parser.long2Byte(xid);
    byte[] uidRaw = Parser.long2Byte(di.getUid());
    byte[] oldRaw = di.getOldRaw();
    SubArray raw = di.getRaw();
    byte[] newRaw = Arrays.copyOfRange(raw.raw, raw.start, raw.end);
    return Bytes.concat(logType, xidRaw, uidRaw, oldRaw, newRaw);
}
public static byte[] insertLog(long xid, Page pg, byte[] raw) {
    byte[] logTypeRaw = {LOG_TYPE_INSERT};//标识日志为插入型
    byte[] xidRaw = Parser.long2Byte(xid);//事务id
    byte[] pgnoRaw = Parser.int2Byte(pg.getPageNumber());//数据库中对应的页号
    byte[] offsetRaw = Parser.short2Byte(PageX.getFSO(pg));//页的偏移量
    return Bytes.concat(logTypeRaw, xidRaw, pgnoRaw, offsetRaw, raw);
}
```



### redo处理的实现

读取logger的data部分，如果是插入型日志，且记录该日志的事务已完成，执行插入数据的重做处理
如果是更新型日志，且记录该日志的事务已完成，执行更新数据的重做处理。

```java
private static void redoTranscations(TransactionManager tm, Logger lg, PageCache pc) {
    lg.rewind();
    while(true) {
        byte[] log = lg.next();//返回logger的数据部分
        if(log == null) break;
        if(isInsertLog(log)) {
            InsertLogInfo li = parseInsertLog(log);
            long xid = li.xid;
            if(!tm.isActive(xid)) {//事务不是处于正在被处理的阶段
                doInsertLog(pc, log, REDO);
            }
        } else {
            UpdateLogInfo xi = parseUpdateLog(log);
            long xid = xi.xid;
            if(!tm.isActive(xid)) {
                doUpdateLog(pc, log, REDO);
            }
        }
    }
}
```



### undo处理的实现

读取logger的data部分，如果是插入型日志，且记录该日志的事务未完成，执行插入数据的撤销处理，如果是更新型日志，且记录该日志的事务未完成，执行更新数据的撤销处理。

与重做日志不同的是，应该按照日志xid的大小进行倒序撤销。

倒序记录active事务是为了MVCC。

```java
private static void undoTranscations(TransactionManager tm, Logger lg, PageCache pc) {
    //事务xid号 list里面装日志
    Map<Long, List<byte[]>> logCache = new HashMap<>();
    lg.rewind();
    while(true) {
        byte[] log = lg.next();//log的data部分
        if(log == null) break;
        if(isInsertLog(log)) {
            InsertLogInfo li = parseInsertLog(log);
            long xid = li.xid;
            if(tm.isActive(xid)) {//未完成的事务，事务对应的日志
                if(!logCache.containsKey(xid)) {
                    logCache.put(xid, new ArrayList<>());
                }
                logCache.get(xid).add(log);
            }
        } else {
            UpdateLogInfo xi = parseUpdateLog(log);
            long xid = xi.xid;
            if(tm.isActive(xid)) {
                if(!logCache.containsKey(xid)) {
                    logCache.put(xid, new ArrayList<>());
                }
                logCache.get(xid).add(log);
            }
        }
    }
    // 对所有active log按照事务xid进行倒序日志undo——撤销
    for(Entry<Long, List<byte[]>> entry : logCache.entrySet()) {
        List<byte[]> logs = entry.getValue();
        for (int i = logs.size()-1; i >= 0; i --) {
            byte[] log = logs.get(i);
            if(isInsertLog(log)) {
                doInsertLog(pc, log, UNDO);
            } else {
                doUpdateLog(pc, log, UNDO);
            }
        }
        tm.abort(entry.getKey());
    }
}
```



根据updateLog 和 insertLog 对数据的重做和撤销处理，如下

```java
private static void doUpdateLog(PageCache pc, byte[] log, int flag) {
    int pgno;
    short offset;
    byte[] raw;
    if(flag == REDO) {
        UpdateLogInfo xi = parseUpdateLog(log);
        pgno = xi.pgno;
        offset = xi.offset;
        raw = xi.newRaw;
    } else {
        UpdateLogInfo xi = parseUpdateLog(log);
        pgno = xi.pgno;
        offset = xi.offset;
        raw = xi.oldRaw;
    }
    Page pg = null;
    //从页号在数据库中得到页的对象
    try {
        pg = pc.getPage(pgno);
    } catch (Exception e) {
        Panic.panic(e);
    }
    try {
        PageX.recoverUpdate(pg, raw, offset);
    } finally {
        pg.release();
    }
}

private static void doInsertLog(PageCache pc, byte[] log, int flag) {
    InsertLogInfo li = parseInsertLog(log);
    Page pg = null;
    try {
        pg = pc.getPage(li.pgno);
    } catch(Exception e) {
        Panic.panic(e);
    }
    //doInsertLog() 方法中的删除，使用的是 DataItem.setDataItemRawInvalid(li.raw);，
    // dataItem 将在下一节中说明，大致的作用，就是将该条 DataItem 的有效位设置为无效，来进行逻辑删除
    try {
        if(flag == UNDO) {
            DataItem.setDataItemRawInvalid(li.raw);
        }
        PageX.recoverInsert(pg, li.raw, li.offset);
    } finally {
        pg.release();
    }
}
```



doInsertLog() 方法中的删除，使用的是 DataItem.setDataItemRawInvalid(li.raw);，dataItem 将在下一节中说明，大致的作用，就是将该条 DataItem 的有效位设置为无效，来进行逻辑删除。

### 测试代码

```java
public class test {
    public static void main(String[] args) throws Exception {
        TransactionManagerImpl tm=TransactionManager.open("cun/tm");
        PageCacheImpl pc=PageCache.open("cun/page",1 << 17);
        Logger lg = Logger.open("cun/logger");
        //先截断，然后执行redoTranscations、undoTranscations
        Recover.recover(tm, lg, pc);
    }
}
```



