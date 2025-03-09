# 一、TM模块

[toc]



## TM模块作用

维护 XID 文件,并提供接口供其他模块来查询某个事务的状态。

### 提供的方法

```java
begin(T)//开启事务
commit(T)//提交事务
abort(T)//撤销事务

//判断事务状态
isActive(T)
isCommitted(T)
isAborted(T)
```

### XID文件存储格式

事务的个数(8个字节) | 事务 xid1 的状态(1个字节)  | 事务 xid2 的状态(1个字节)|  ...

```java
[Xid_header][xid（隐含递增）][status（1B）][xid（隐含递增）][status（1B）].......
```



### 备注

1 每一个事务都有一个 XID，这个 ID 唯一标识了这个事务。==事务的 XID 从 1 开始标号，并自增，不可重复。==

2 定义xid0为超级事务（即可以没有申请事务的情况下进行，永远处于提交状态），不需要记录

3 RandomAccessFile用来记录事务ID的文件，好处是有个指针可以指示当前文件位置，文件读写都采用了 NIO 方式的 FileChannel

4 每个事务都有下面的三种状态：

active，正在进行，尚未结束
committed，已提交
aborted，已撤销（回滚）

## 主要源码

### 创建一个 xid 文件并创建 TM对象

```java
public static TransactionManagerImpl create(String path) {
    File f = new File(path+TransactionManagerImpl.XID_SUFFIX);
    try {
        if(!f.createNewFile()) {
            Panic.panic(Error.FileExistsException);
        }
    } catch (Exception e) {
        Panic.panic(e);
    }
    if(!f.canRead() || !f.canWrite()) {
        Panic.panic(Error.FileCannotRWException);
    }
    FileChannel fc = null;
    RandomAccessFile raf = null;
    try {
        raf = new RandomAccessFile(f, "rw");
        fc = raf.getChannel();
    } catch (FileNotFoundException e) {
        Panic.panic(e);
    }

    // 写空XID文件头
    ByteBuffer buf = ByteBuffer.wrap(new byte[TransactionManagerImpl.LEN_XID_HEADER_LENGTH]);
    try {
        //从零创建 XID 文件时需要写一个空的 XID 文件头，即设置 xidCounter 为 0，
        // 否则后续在校验时会不合法：
        fc.position(0);
        fc.write(buf);
    } catch (IOException e) {
        Panic.panic(e);
    }
    return new TransactionManagerImpl(raf, fc);
}
```

 ```java
//从一个已有的 xid 文件来创建 TM
public static TransactionManagerImpl open(String path) {
    File f = new File(path+TransactionManagerImpl.XID_SUFFIX);
    if(!f.exists()) {
        Panic.panic(Error.FileNotExistsException);
    }
    if(!f.canRead() || !f.canWrite()) {
        Panic.panic(Error.FileCannotRWException);
    }
    FileChannel fc = null;
    RandomAccessFile raf = null;
    try {
        raf = new RandomAccessFile(f, "rw");//用来访问那些保存数据记录的文件
        fc = raf.getChannel();//返回与这个文件有关的唯一FileChannel对象
    } catch (FileNotFoundException e) {
        Panic.panic(e);
    }
    return new TransactionManagerImpl(raf, fc);
}
 ```



### 检查XID文件是否合法

​        通过xid文件开头的 8 个字节（记录了事务的个数）反推文件的理论长度与文件的实际长度做对比。如果不等则认为 XID 文件不合法。每次创建xid对象都会检查一次。

文件的理论长度：开头的8个字节+一个事务状态所占用的字节*事务的个数

```java
public void checkXIDCounter() {
    long fileLen = 0;
    try {
        fileLen = file.length();
    } catch (IOException e1) {
        Panic.panic(Error.BadXIDFileException);
    }
    if(fileLen < LEN_XID_HEADER_LENGTH) {//8
        //对于校验没有通过的，会直接通过 panic 方法，强制停机。
        // 在一些基础模块中出现错误都会如此处理，
        // 无法恢复的错误只能直接停机。
        Panic.panic(Error.BadXIDFileException);
    }
    // java NIO中的Buffer的array()方法在能够读和写之前，必须有一个缓冲区，
    // 用静态方法 allocate() 来分配缓冲区
    ByteBuffer buf = ByteBuffer.allocate(LEN_XID_HEADER_LENGTH);//8
    try {
        fc.position(0);
        fc.read(buf);
    } catch (IOException e) {
        Panic.panic(e);
    }
    //从文件开头8个字节得到事务的个数
    this.xidCounter = Parser.parseLong(buf.array());
    // 根据事务xid取得其在xid文件中对应的位置
    long end = getXidPosition(this.xidCounter + 1);
    if(end != fileLen) {
        //对于校验没有通过的，会直接通过 panic 方法，强制停机
        Panic.panic(Error.BadXIDFileException);
    }
}

// 根据事务xid取得其在xid文件中对应的位置
private long getXidPosition(long xid) {
    return LEN_XID_HEADER_LENGTH + (xid-1)*XID_FIELD_SIZE;
}
```



### 改变事务状态

​        事务xid的偏移量=开头的8个字节+一个事务状态所占用的字节*事务的xid

**其大致思路为：通过事务xid计算出记录该事务状态的偏移量，再更改状态。**

```java
private void updateXID(long xid, byte status) {
    long offset = getXidPosition(xid);
    byte[] tmp = new byte[XID_FIELD_SIZE];//1每个事务占用长度
    tmp[0] = status;
    ByteBuffer buf = ByteBuffer.wrap(tmp);
    try {
        fc.position(offset);
        fc.write(buf);
    } catch (IOException e) {
        Panic.panic(e);
    }
    try {
        //对于校验没有通过的，会直接通过 panic 方法，强制停机
        fc.force(false);
    } catch (IOException e) {
        Panic.panic(e);
    }
}
```
同上，下面abort()和commit()则是调用这个方法

```java
// 提交XID事务,事务id改为已提交
public void commit(long xid) {
    updateXID(xid, FIELD_TRAN_COMMITTED);
}
// 回滚XID事务
public void abort(long xid) {
    updateXID(xid, FIELD_TRAN_ABORTED);
}
```



### 更新XID Header

每次创建一个事务时，都需要将文件头记录的事务个数+1

```java
private void incrXIDCounter() {
    xidCounter ++;
    ByteBuffer buf = ByteBuffer.wrap(Parser.long2Byte(xidCounter));
    //System.out.println(buf);
    //游标pos， 限制为lim， 容量为cap
    try {
        fc.position(0);
        fc.write(buf);
    } catch (IOException e) {
        Panic.panic(e);
    }
    try {
        fc.force(false);
    } catch (IOException e) {
        Panic.panic(e);
    }
}
```



### 开启事务

其大致流程如下

1 首先设置 xidCounter+1 事务的状态为 actived
2 随后 xidCounter 自增，并更新文件头（流程4）。

```java
	public long begin() {
        counterLock.lock();
        try {
            long xid = xidCounter + 1;
            updateXID(xid, FIELD_TRAN_ACTIVE);//正在进行xid
            incrXIDCounter();
            return xid;
        } finally {
            counterLock.unlock();
        }
    }
```

### 提交事务

设置事务xid的状态为 committed

```java
	// 提交XID事务,事务id改为已提交
    public void commit(long xid) {
        updateXID(xid, FIELD_TRAN_COMMITTED);
    }
```

### 回滚事务

设置事务xid的状态为 aborted

```java
public void abort(long xid) {
    updateXID(xid, FIELD_TRAN_ABORTED);
}
```

### 检查事务状态

根据xid-》获取记录事务xid的状态的offset-》读取事务xid的状态-》是否等于status

```java
private boolean checkXID(long xid, byte status) {
    long offset = getXidPosition(xid);
    //ByteBuffer俗称缓冲器， 是将数据移进移出通道的唯一方式
    ByteBuffer buf = ByteBuffer.wrap(new byte[XID_FIELD_SIZE]);
    try {
        fc.position(offset);
        fc.read(buf);
    } catch (IOException e) {
        Panic.panic(e);
    }
    return buf.array()[0] == status;
}
```

## 测试代码

```java
public class test {
    public static void main(String[] args) throws FileNotFoundException {
        TransactionManagerImpl tm=TransactionManager.create("cun/a");
        //开启一个事务，返回事务的xid号
        long xid=tm.begin();
        //提交事务
        tm.commit(xid);
        //tm.abort(xid);//回滚事务
        System.out.println(tm.isAborted(xid));
    }
}
```

