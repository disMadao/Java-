# TBM模块-字段解析与表管理

[toc]

BM也是最后一个模块，实现了对字段结构和表结构的管理，其与其他模块的依赖关系如下

<img src="D:\stu\my_blog_note\blog_note\项目\MYDB\assets\image-20250111145354172.png" alt="image-20250111145354172" style="zoom:50%;" />

 

## SQL 解析器

实现的 SQL 语句语法如下：

```html
<begin statement>
    begin [isolation level (read committed|repeatable read)]
        begin isolation level read committed
 
<commit statement>
    commit
 
<abort statement>
    abort
 
<create statement>
    create table <table name>
    <field name> <field type>
    <field name> <field type>
    ...
    <field name> <field type>
    [(index <field name list>)]
        create table students
        id int32,
        name string,
        age int32,
        (index id name)
 
<drop statement>
    drop table <table name>
        drop table students
 
<select statement>
    select (*|<field name list>) from <table name> [<where statement>]
        select * from student where id = 1
        select name from student where id > 1 and id < 4
        select name, age, id from student where id = 12
 
<insert statement>
    insert into <table name> values <value list>
        insert into student values 5 "Zhang Yuanjia" 22
 
<delete statement>
    delete from <table name> <where statement>
        delete from student where name = "Zhang Yuanjia"
 
<update statement>
    update <table name> set <field name>=<value> [<where statement>]
        update student set name = "ZYJ" where id = 5
 
<where statement>
    where <field name> (>|<|=) <value> [(and|or) <field name> (>|<|=) <value>]
        where age > 10 or age < 3
 
<field name> <table name>
    [a-zA-Z][a-zA-Z0-9_]*
 
<field type>
    int32 int64 string
```



### 语句切割

parser 包的 Tokenizer 类，对语句进行逐字节解析，根据空白符或者上述词法规则，将语句切割成多个 token。对外提供了 peek()、pop() 方法方便取出 Token 进行解析。

> 其解析规则如下
>
> 1 取出完整的字符串
>
> 2 取出引号包起来的内容
>
> 3 取出符号，如> < =

其具体实现代码为

```java
private String nextMetaState() throws Exception {
    while(true) {
        //遍历字节数组，直至遍历完，取出
        Byte b = peekByte();
        if(b == null) {
            return "";
        }
        //遇到空格或回车，忽略，继续遍历
        if(!isBlank(b)) {
            break;
        }
        popByte();//pos++;
    }
    byte b = peekByte();
    //如果b是<>=*,这些符号，取出符号
    if(isSymbol(b)) {
        popByte();
        return new String(new byte[]{b});
        //如果b是单引号或者双引号，找到单引号双引号包起来的内容
    } else if(b == '"' || b == '\'') {
        return nextQuoteState();
        //如果b是数字或字母，找到完整的数字或字符串
    } else if(isAlphaBeta(b) || isDigit(b)) {
        return nextTokenState();
    } else {
        err = Error.InvalidCommandException;
        throw err;
    }
}
```

测试代码

```java
public class test {
    public static void main(String[] args) throws Exception {
        String ss=" insert into student values 5 \"xiaohong\" 22";
        byte[]b=ss.getBytes(StandardCharsets.UTF_8);
        Tokenizer tokenizer = new Tokenizer(b);
        for(int i=0;i<b.length;i++){
            String token = tokenizer.peek();
            System.out.println(token);
            tokenizer.pop();
        }
    }
}
```

解析出的字段为  

insert
into
student
values
5
xiaohong
22
语句解析
解析过程是根据解析出的第一个 Token 来区分语句类型，并分别做处理

```java
Tokenizer tokenizer = new Tokenizer(statement);
String token = tokenizer.peek();
tokenizer.pop();
Object stat = null;
Exception statErr = null;
try {
    switch(token) {
        case "begin":
            stat = parseBegin(tokenizer);
            break;
        case "commit":
            stat = parseCommit(tokenizer);
            break;
        case "abort":
            stat = parseAbort(tokenizer);
            break;
        case "create":
            stat = parseCreate(tokenizer);
            break;
        case "drop":
            stat = parseDrop(tokenizer);
            break;
        case "select":
            stat = parseSelect(tokenizer);
            break;
        case "insert":
            stat = parseInsert(tokenizer);
            break;
        case "delete":
            stat = parseDelete(tokenizer);
            break;
        case "update":
            stat = parseUpdate(tokenizer);
            break;
        case "show":
            stat = parseShow(tokenizer);
            break;
        default:
            throw Error.InvalidCommandException;
    }
} catch(Exception e) {
    statErr = e;
}
```



以解析插入的SQL 语句为例，需要解析出要插入的表名，和要插入的字段。如果解析出的字段不满足规则，报错。

以insert为例解析出的字段必须满足

insert into 表名（自定 格式 以字母开头） values value（自定）

```java
private static Insert parseInsert(Tokenizer tokenizer) throws Exception {
    Insert insert = new Insert();
    //比如解析insert的下一句一定得是into
    if(!"into".equals(tokenizer.peek())) {
        throw Error.InvalidCommandException;
    }
    tokenizer.pop();
    //解析出表名，不满足规则报错
    String tableName = tokenizer.peek();
    if(!isName(tableName)) {
        throw Error.InvalidCommandException;
    }
    insert.tableName = tableName;
    tokenizer.pop();
    //下一个字段必须是values
    if(!"values".equals(tokenizer.peek())) {
        throw Error.InvalidCommandException;
    }
    //要插入的值
    List<String> values = new ArrayList<>();
    while(true) {
        tokenizer.pop();
        String value = tokenizer.peek();
        if("".equals(value)) {
            break;
        } else {
            values.add(value);
        }
    }
        insert.values = values.toArray(new String[values.size()]);
	return insert;
}
```

## 字段与表管理

### 字段管理Field

#### 存储结构

由于 TBM 基于 VM的基础之上的，单个字段信息和表信息都是直接保存在 Entry 中，即保存在VM结构的【data】部分。Entry的存储结构为

```java
[FieldName][TypeName][IndexUid]
字段名      字段类型    字段根节点uid
```

 这里 FieldName 和 TypeName，以及后面要介绍的TableName，存储的都是字节形式的字符串。这里规定一个字符串的存储方式，以明确其存储边界。

```java
[StringLength][StringData]
```

TypeName 为字段的类型，限定为 int32、int64 和 string 类型。如果这个字段有索引，那个 IndexUID 指向了索引二叉树的根，否则该字段为 0。

即，**如果该字段建立了索引，则以该字段会建立一个B+树。**

#### 梳理前面存储结果

为了不容易混淆，我们总结一下前面介绍过的存储结果。

DataItem的存储结构为

```java
[ValidFlag][Size][Data]
```

VM的存储结构为一个Entry，存储在DataItem的【Data】中，Entry的存储结构为

```java
[XMin][XMax][Data]
```

IM存储结构为一个Node，存储在DataItem的【Data】中，Node的存储结构为

```java
[LeafFlag][KeyNumber][SiblingUid]
占1个字节          2         8
[Son0][Key0][Son1][Key1]...[SonN][KeyN]
8      8     8
```


重新回到TBM模块上来

#### 读取字段

根据这个结构，通过一个 UID 从表中读取字段并解析如下：

```java
public static Field loadField(Table tb, long uid) {
    byte[] raw = null;
    try {
        //读取一个entry.data()
        raw = ((TableManagerImpl)tb.tbm).vm.read(TransactionManagerImpl.SUPER_XID, uid);
    } catch (Exception e) {
        Panic.panic(e);
    }
    assert raw != null;
    return new Field(uid, tb).parseSelf(raw);
}

private Field parseSelf(byte[] raw) {
    int position = 0;
    ParseStringRes res = Parser.parseString(raw);
    //解析出字段名
    fieldName = res.str;
    position += res.next;
    res = Parser.parseString(Arrays.copyOfRange(raw, position, raw.length));
    //解析出字段类型
    fieldType = res.str;
    position += res.next;
    //该字段根节点的索引
    this.index = Parser.parseLong(Arrays.copyOfRange(raw, position, position+8));
    if(index != 0) {
        try {
            bt = BPlusTree.load(index, ((TableManagerImpl)tb.tbm).dm);
        } catch(Exception e) {
            Panic.panic(e);
        }
    }
    return this;
}
```



#### 创建字段

创建一个字段的方法类似，将相关的信息通过 VM 持久化即可：

```java
//创建字段
public static Field createField(Table tb, long xid, String fieldName, String fieldType, boolean indexed) throws Exception {
    typeCheck(fieldType);
    Field f = new Field(tb, fieldName, fieldType, 0);
    //有索引
    if(indexed) {
        //生成一个空的根节点   ，返回存放根节点的uid
        long index = BPlusTree.create(((TableManagerImpl)tb.tbm).dm);
        //以该uid为bootUid的节点为根节点建立一个B+树
        BPlusTree bt = BPlusTree.load(index, ((TableManagerImpl)tb.tbm).dm);
        f.index = index;
        f.bt = bt;
    }
    f.persistSelf(xid);
    return f;
}
//创建一个字段的方法类似，将相关的信息通过 VM 持久化即可
private void persistSelf(long xid) throws Exception {
    byte[] nameRaw = Parser.string2Byte(fieldName);
    byte[] typeRaw = Parser.string2Byte(fieldType);
    byte[] indexRaw = Parser.long2Byte(index);
    this.uid = ((TableManagerImpl)tb.tbm).vm.insert(xid, Bytes.concat(nameRaw, typeRaw, indexRaw));
}
```



### 表管理Table

#### 存储结构

一个数据库中存在多张表，TBM 使用链表的形式将其组织起来，每一张表都保存一个指向下一张表的 UID。表的二进制结构如下：

```java
[TableName][NextTable]
表名        下一张表的uid
[Field1Uid][Field2Uid]...[FieldNUid]
```

#### 读取表

根据 UID 从 Entry 中读取表数据的过程和读取字段的过程类似。

```java
//根据uid从已有文件中加载表
public static Table loadTable(TableManager tbm, long uid) {
    byte[] raw = null;
    try {
        raw = ((TableManagerImpl)tbm).vm.read(TransactionManagerImpl.SUPER_XID, uid);
    } catch (Exception e) {
        Panic.panic(e);
    }
    assert raw != null;
    Table tb = new Table(tbm, uid);
    return tb.parseSelf(raw);
}
```

#### 解析表

表的解析过程如下，由于uid的字节数是固定的（8个），所以于是无需保存字段的个数。

1 解析表名

2 解析下一个表的uid

3 解析表中的所有字段，且创建字段，把所有的字段存储在一个fields中

```java
private Table parseSelf(byte[] raw) {
    int position = 0;
    ParseStringRes res = Parser.parseString(raw);
    //表名
    name = res.str;
    position += res.next;
    //下一个表的uid
    nextUid = Parser.parseLong(Arrays.copyOfRange(raw, position, position+8));
    position += 8;
    //字段的uid
    while(position < raw.length) {
        long uid = Parser.parseLong(Arrays.copyOfRange(raw, position, position+8));
        position += 8;
        fields.add(Field.loadField(this, uid));
    }
    return this;
}
```



#### 创建表

常常用于插入新表时，用Create存储一张表的信息。

```java
public class Create {
    public String tableName;//表名
    public String[] fieldName;//存储的字段名
    public String[] fieldType;//存储的字段类型
    public String[] index;//存储的索引
}
```

创建表时，需要把表中所有的字段信息存储到dm中

```java
//创建表，用于插入新表时使用
//nextUid原来第一张表的uid
//xid插入新表的xid
//要插入新表的所有信息
public static Table createTable(TableManager tbm, long nextUid, long xid, Create create) throws Exception {
    Table tb = new Table(tbm, create.tableName, nextUid);
    for(int i = 0; i < create.fieldName.length; i ++) {
        String fieldName = create.fieldName[i];
        String fieldType = create.fieldType[i];
        boolean indexed = false;
        for(int j = 0; j < create.index.length; j ++) {
            if(fieldName.equals(create.index[j])) {
                indexed = true;
                break;
            }
        }
        //为新表的字段存储到dm中
        tb.fields.add(Field.createField(tb, xid, fieldName, fieldType, indexed));
    }
    return tb.persistSelf(xid);
}

private Table persistSelf(long xid) throws Exception {
    byte[] nameRaw = Parser.string2Byte(name);
    byte[] nextRaw = Parser.long2Byte(nextUid);
    byte[] fieldRaw = new byte[0];
    for(Field field : fields) {
        fieldRaw = Bytes.concat(fieldRaw, Parser.long2Byte(field.uid));
    }
    uid = ((TableManagerImpl)tbm).vm.insert(xid, Bytes.concat(nameRaw, nextRaw, fieldRaw));
    return this;
}
```



#### Update操作

其步骤为

1 确认要更新的是哪张表

2 找出命中字段中满足where条件的uid，解析出存储的一条数据（表中可能有多个字段，以键值对存储）

3 找到要更新的字段fd

4 删除原有的字段，实则是设置xmax

5 更新一条数据，往DM插入更新后的值得到uid

6 在所有indexed字段建立的B+树中插入新的的key|uid

```java
public int update(long xid, Update update) throws Exception {
    //找出命中字段中满足where条件的uid，如a=2,a=3
    List<Long> uids = parseWhere(update.where);
    Field fd = null;
    //找到要更新的字段fd
    for (Field f : fields) {
        if(f.fieldName.equals(update.fieldName)) {
            fd = f;
            break;
        }
    }
    if(fd == null) {
        throw Error.FieldNotFoundException;
    }
    //把需要更新的值变成相应的类型
    Object value = fd.string2Value(update.value);
    int count = 0;
    for (Long uid : uids) {
        //根据条件找到的uid
        byte[] raw = ((TableManagerImpl)tbm).vm.read(xid, uid);
        if(raw == null) continue;
        //更新即先删除再插入，删除时要判断死锁和可视化,实际上是设置xmax
        ((TableManagerImpl)tbm).vm.delete(xid, uid);
        //返回未更新之前的字段entry={name=xiaohong, age=18}
        Map<String, Object> entry = parseEntry(raw);
        //替换新值
        entry.put(fd.fieldName, value);//name=ZYJ,age=18
        //把新的值concat到一起
        raw = entry2Raw(entry);
        long uuid = ((TableManagerImpl)tbm).vm.insert(xid, raw);
        count ++;
        //更新B+树
        for (Field field : fields) {
            if(field.isIndexed()) {
                //字段的值，存储字段的值的uid
                field.insert(entry.get(field.fieldName), uuid);
            }
        }
    }
    return count;
}
```

> 从上面的代码中可以看出，删除数据时，并没有删除b+树中的索引，而是直接把新值再插入b+树索引中。mysql也是这么设计的，它可以通过新建一个表复制原表压缩索引文件的空间。

#### insert操作

把插入的值合并到一个raw里面作为索引进行插入，然后为所有indexed字段建立的B+树更新树

```java
public void insert(long xid, Insert insert) throws Exception {
    Map<String, Object> entry = string2Entry(insert.values);
    System.out.println("要插入的数据是"+entry);
    //把entry里面所有的值合并在一个raw里面
    byte[] raw = entry2Raw(entry);
    long uid = ((TableManagerImpl)tbm).vm.insert(xid, raw);
    for (Field field : fields) {
        if(field.isIndexed()) {
            field.insert(entry.get(field.fieldName), uid);
        }
    }
}
```

#### read操作

同样，IM模块找出所有符合条件的uid，再利用VM模块进行可视化判断。

```java
public String read(long xid, Select read) throws Exception {
    List<Long> uids = parseWhere(read.where);
    StringBuilder sb = new StringBuilder();
    for (Long uid : uids) {
        byte[] raw = ((TableManagerImpl)tbm).vm.read(xid, uid);
        if(raw == null) continue;
        Map<String, Object> entry = parseEntry(raw);
        sb.append(printEntry(entry)).append("\n");
    }
    return sb.toString();
}
```

计算 Where 条件的范围
对表和字段的操作，有一个很重要的步骤，就是计算 Where 条件的范围， 比如Delete和Select都需要计算 Where，最终就需要获取到条件范围内所有的 UID，**这里只支持了带有索引的两个条件的查询**

```java
//解析where语句，返回uid
private List<Long> parseWhere(Where where) throws Exception {
    long l0=0, r0=0, l1=0, r1=0;
    boolean single = false;//or字段连起来是false，其他是true
    Field fd = null;
    if(where == null) {//没有指定where范围
        for (Field field : fields) {
            if(field.isIndexed()) {
                fd = field;
                break;
            }
        }
        l0 = 0;
        r0 = Long.MAX_VALUE;
        single = true;
    } else {//指定了where范围
        for (Field field : fields) {
            //指定了是查找哪个字段
            if(field.fieldName.equals(where.singleExp1.field)) {
                if(!field.isIndexed()) {
                    throw Error.FieldNotIndexedException;
                }
                fd = field;
                break;
            }
        }
        if(fd == null) {
            throw Error.FieldNotFoundException;
        }
        CalWhereRes res = calWhere(fd, where);
        l0 = res.l0; r0 = res.r0;//第一个条件的低水位和高水位
        l1 = res.l1; r1 = res.r1;//第二个条件的低水位和高水位
        single = res.single;
    }
    //在b+树种找出所有满足范围的uid
    List<Long> uids = fd.search(l0, r0);
    if(!single) {//or字段，增加后一个条件的uid
        List<Long> tmp = fd.search(l1, r1);
        uids.addAll(tmp);
    }
    return uids;
}
```



在字段中搜寻满足where条件的高低水位如下，如果是and字段，取两个条件的高低水位的并集。

```java
private CalWhereRes calWhere(Field fd, Where where) throws Exception {
    CalWhereRes res = new CalWhereRes();
    switch(where.logicOp) {
        case "":
            res.single = true;
            FieldCalRes r = fd.calExp(where.singleExp1);
            res.l0 = r.left; res.r0 = r.right;
            break;
        case "or":
            res.single = false;
            r = fd.calExp(where.singleExp1);
            res.l0 = r.left; res.r0 = r.right;
            r = fd.calExp(where.singleExp2);
            res.l1 = r.left; res.r1 = r.right;
            break;
        case "and"://两个条件的高低水位取交集
            res.single = true;
            r = fd.calExp(where.singleExp1);
            res.l0 = r.left; res.r0 = r.right;
            r = fd.calExp(where.singleExp2);
            res.l1 = r.left; res.r1 = r.right;
            if(res.l1 > res.l0) res.l0 = res.l1;//取left的最大值
            if(res.r1 < res.r0) res.r0 = res.r1;//取right的最小值
            break;
        default:
            throw Error.InvalidLogOpException;
    }
    return res;
}
```



从一个where条件中解析出高低水位如下

```java
//a>5  left=6 right=MAX_VALUE
//a<5  left=0 right=4;
//a=5  left=right=5
public FieldCalRes calExp(SingleExpression exp) throws Exception {
    Object v = null;
    FieldCalRes res = new FieldCalRes();
    switch(exp.compareOp) {
        case "<":
            res.left = 0;
            v = string2Value(exp.value);
            res.right = value2Uid(v);
            if(res.right > 0) {
                res.right --;
            }
            break;
        case "=":
            v = string2Value(exp.value);
            res.left = value2Uid(v);
            res.right = res.left;
            break;
        case ">":
            res.right = Long.MAX_VALUE;
            v = string2Value(exp.value);
            res.left = value2Uid(v) + 1;
            break;
    }
    return res;
}
```



### 表的启动管理 

由于 TBM 的表管理，使用的是链表串起的 Table 结构，所以就必须保存一个链表的头节点，即第一个表的 UID，这样在数据库 启动时，才能快速找到表信息。

 

数据库使用 Booter 类和 bt 文件，来管理 数据库 的启动信息，虽然现在所需的启动信息，只有一个：头表的 UID。Booter 类对外提供了两个方法：load 和 update，并保证了其原子性。update 在修改 bt 文件内容时，没有直接对 bt 文件进行修改，而是首先将内容写入一个 bt_tmp 文件中，随后将这个文件重命名为 bt 文件。以期通过操作系统重命名文件的原子性，来保证操作的原子性。

```java
public void update(byte[] data) {
    //创建一个bt_tmp文件
    File tmp = new File(path + BOOTER_TMP_SUFFIX);
    try {
        tmp.createNewFile();
    } catch (Exception e) {
        Panic.panic(e);
    }
    if(!tmp.canRead() || !tmp.canWrite()) {
        Panic.panic(Error.FileCannotRWException);
    }
    //将启动信息写入一个 bt_tmp 文件中
    try(FileOutputStream out = new FileOutputStream(tmp)) {
        out.write(data);
        out.flush();
    } catch(IOException e) {
        Panic.panic(e);
    }
    try {
        //随后将这个文件重命名为 bt 文件,并覆盖之前的同名文件
        Files.move(tmp.toPath(), new File(path+BOOTER_SUFFIX).toPath(), StandardCopyOption.REPLACE_EXISTING);
    } catch(IOException e) {
        Panic.panic(e);
    }
    file = new File(path+BOOTER_SUFFIX);
    if(!file.canRead() || !file.canWrite()) {
        Panic.panic(Error.FileCannotRWException);
    }
}
```



当创建tbm对象时，需要把数据库中所有的表加入到缓存中。

```java
TableManagerImpl(VersionManager vm, DataManager dm, Booter booter) {
    this.vm = vm;
    this.dm = dm;
    this.booter = booter;
    this.tableCache = new HashMap<>();
    this.xidTableCache = new HashMap<>();
    lock = new ReentrantLock();
    loadTables();
}
//得到tableCache
private void loadTables() {
    //里面调用booter.load();从文件里面读第一个表的uid
    long uid = firstTableUid();
    while(uid != 0) {
        Table tb = Table.loadTable(this, uid);
        uid = tb.nextUid;
        tableCache.put(tb.name, tb);
    }
}
```



### TableManager的实现

TBM 层对外提供服务的是 TableManager 接口，如下：

```java
public interface TableManager {
    BeginRes begin(Begin begin);
    byte[] commit(long xid) throws Exception;
    byte[] abort(long xid);
    byte[] show(long xid);
    byte[] create(long xid, Create create) throws Exception;

    byte[] insert(long xid, Insert insert) throws Exception;
    byte[] read(long xid, Select select) throws Exception;
    byte[] update(long xid, Update update) throws Exception;
    byte[] delete(long xid, Delete delete) throws Exception;
}
```

由于 TableManager 已经是直接被最外层 Server 调用），这些方法直接返回执行的结果，例如错误信息或者结果信息的字节数组）。

各个方法的具体实现很简单，不再赘述，无非是调用 VM 的相关方法。唯一值得注意的一个小点是，在创建新表时，采用的是**头插法**，所以每次创建表都需要更新 Booter 文件。

测试代码

```java
public class test {
    public static void main(String[] args) throws Exception {
        TransactionManagerImpl tm= TransactionManager.create("cun/tm");
        DataManager dm=DataManager.create("cun/dm",1 << 20,tm);
        VersionManager vm=VersionManager.newVersionManager(tm,dm);
        TableManager tbm=TableManager.create("cun/",vm,dm);
        //开启事务
        BeginRes br=tbm.begin(new Begin());
        long xid=br.xid;
        //建立一张新表
        String ss="create table students " +
                "name string,age int32 " +
                "(index name age)";
        byte b[]=ss.getBytes(StandardCharsets.UTF_8);
        Object stat = Parser.Parse(b);
        tbm.create(xid,(Create) stat);
        System.out.println("===测试插入操作===");
        ss="insert into students values xiaohong 18";
        b=ss.getBytes(StandardCharsets.UTF_8);
        stat = Parser.Parse(b);
        tbm.insert(xid,(Insert) stat);
        System.out.println("===测试更新操作===");
        ss="update students set name = \"ZYJ\" where name = xiaohong";
        b=ss.getBytes(StandardCharsets.UTF_8);
        stat = Parser.Parse(b);
        tbm.update(xid,(Update) stat);
        System.out.println("===测试查询操作===");
        ss="select name,age from students where age=18";
        b=ss.getBytes(StandardCharsets.UTF_8);
        stat = Parser.Parse(b);
        byte[] output=tbm.read(xid,(Select) stat);
        System.out.println(new String(output));
        System.out.println("===测试删除操作===");
        ss=" delete from students where age =18";
        b=ss.getBytes(StandardCharsets.UTF_8);
        stat = Parser.Parse(b);
        tbm.delete(xid,(Delete) stat);
    }
}
```



 

