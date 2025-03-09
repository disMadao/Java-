# 实现SelectPlus

需要实现的功能

1. 无索引字段的where
2. 连接两个表的查询，三种join，先实现full join



select student.id,class.name from student full join class where student.classId = class.id



整理下思路，现在insert语句里面应该是不允许null，所以三种join一样，这个我之后再改。

现在我先实现inner join：直接利用两个表查出所有的uid列表，然后一个二重循环构成一个StringBuffer，然后再从这个StringBuffer里面用where查找。



Parser里添加了两个解析select语句的方法。

```
parseSelectPlus  parseWherePlus

```





public void Cartesian(long xid) throws Exception{

这个方法里面的两个list还未充数据，uids1,uids2







这个项目有问题！！！！！b+数索引有问题，`select * form person`，但where条件为空时查询无效，查不到数据！！

leftKey不能设置为0，因为里面的索引ik可能为负数！！！！！！！！！！！





# SelectPlus

[toc]

## 大致思路

selectPlus实现两个表的连接查询操作。

为了尽量不更改原项目代码，直接增加一种语法：

```java
selectplus [字段] from [表名1] [join方式] [表名2] on [连接条件] where [条件1] [逻辑] [条件2]
```

直接仿照原项目select的实现。但是原项目中，select是只能在where条件中使用索引，这里做个更改。

selectplus里不用索引，索引只在笛卡儿积找两表全部数据的时候使用，后面直接将得到的数据做成二维byte数组，然后进行on的过滤，和where的过滤。

## 自顶向下分析sql

我们先来自顶向下的看看一条sql语句的执行的实现过程：

看过第十节就直到，sql语句都是由Executor类的execute(byte[] sql)方法来执行的。

查看这个方法可以知道，sql语句首先被 Parser.Parse(sql);解析成一个Object对象，然后判断这个Object对象到底是哪个语句的。然后进入抛给Eexcutor内部的一个TableManager对象来执行对应方法的对应操作。

然后这个TableManagerImpl实现对象，以select为例，会调用里面的read方法，然后这个read方法又会调用Table对象的read方法。返回最终byte数组结果。

在这个table.read(long xid, Select read)方法里面。会像通过传入的Select对象，解析出符合条件的uids，然后抛给VersionManager对象查询出可见的数据转化成String类型。

> 这里有点啰嗦，建议自己从Executor类按照select语句往下查看就是，不算复杂。



## 模仿select

仿照select语句，创建SelectPlus类和WherePlus类（因为两个表相连有别名，需要个新的类）

SelectPlus类如下：

```java
public class SelectPlus {
    public String tableName1;
//    public String[] fields1;
    public String[] fields;//查询的字段
    public String tableName2;

    public WherePlus wherePlus;

    public String connectionType;
    public String on1;//on条件
    public String on2;
}
```

WherePlus类如下：

> 其实内容和Where类一样，但为了区分还是新增一个，也方便后面添加新功能。

```java
public class WherePlus {
    public SingleExpression singleExp1;
    public String logicOp;
    public SingleExpression singleExp2;

}
```

修改Parser类，解析SelectPlus，得到SelectPlus对象和WherePlus对象，主要实现两个方法：

```java
//selectPlus功能
private static SelectPlus parseSelectPlus(Tokenizer tokenizer) throws Exception {
    SelectPlus read = new SelectPlus();
    List<String> fields = new ArrayList<>();//字段，两表链接的话，应该类似student.id

    //        System.out.println("进入paraSelectPlus");
    String asterisk = tokenizer.peek();
    System.out.println(asterisk);
    if("*".equals(asterisk)) {//查询两个表的所有字段
        fields.add(asterisk);
        tokenizer.pop();
    } else {
        while (true) {
            String field = tokenizer.peek();
            if (!isName(field)) {
                throw Error.InvalidCommandException;
            }
            fields.add(field);
            tokenizer.pop();
            if (",".equals(tokenizer.peek())) {
                tokenizer.pop();
            } else {
                break;
            }
        }
    }
    //        System.out.println("查询字段解析完成");
    read.fields = fields.toArray(new String[fields.size()]);
    if(!"from".equals(tokenizer.peek())) {
        throw Error.InvalidCommandException;
    }
    //        System.out.println('a');
    tokenizer.pop();
    String tableName1 = tokenizer.peek();//第一个表
    if(!isName(tableName1)) {
        throw Error.InvalidCommandException;
    }
    tokenizer.pop();
    //        System.out.println('b');
    String connectionType = tokenizer.peek();//取出连接方式
    tokenizer.pop();
    if(!"join".equals(tokenizer.peek())) {
        throw Error.InvalidCommandException;
    }
    tokenizer.pop();
    //        System.out.println('c');
    String tableName2 = tokenizer.peek();//第二个表
    if(!isName(tableName2)) {
        throw Error.InvalidCommandException;
    }
    tokenizer.pop();
    read.tableName1 = tableName1;
    read.tableName2 = tableName2;
    read.connectionType = connectionType;
    if(!"on".equals(tokenizer.peek())) {//on字段，后面跟连接的字段
        throw Error.InvalidCommandException;
    }
    //        System.out.println("解析到了on后面");
    tokenizer.pop();
    String on = tokenizer.peek();//on就暂时不判断了。后面再改
    tokenizer.pop();
    read.on1 = on;
    //        System.out.println("得到了on1");
    if(!"=".equals(tokenizer.peek())) {
        throw Error.InvalidCommandException;
    }
    tokenizer.pop();
    read.on2 = tokenizer.peek();
    tokenizer.pop();

    //        System.out.println("得到的on1："+read.on1+" on2："+read.on2);
    String tmp = tokenizer.peek();
    if("".equals(tmp)) {
        //            System.out.println("解析出当前where条件为空");
        read.wherePlus = null;
        return read;
    }
    //        System.out.println("开始解析where条件");

    read.wherePlus = parseWherePlus(tokenizer);
    //        System.out.println("得到了wherePlus");
    return read;
}
private static WherePlus parseWherePlus(Tokenizer tokenizer) throws Exception {
    WherePlus wherePlus = new WherePlus();

    if(!"where".equals(tokenizer.peek())) {
        throw Error.InvalidCommandException;
    }
    tokenizer.pop();

    SingleExpression exp1 = parseSingleExp(tokenizer);
    wherePlus.singleExp1 = exp1;//注意，这里面是包含别名的

    String logicOp = tokenizer.peek();
    if("".equals(logicOp)) {
        wherePlus.logicOp = logicOp;
        return wherePlus;
    }
    if(!isLogicOp(logicOp)) {
        throw Error.InvalidCommandException;
    }
    wherePlus.logicOp = logicOp;
    tokenizer.pop();

    SingleExpression exp2 = parseSingleExp(tokenizer);
    wherePlus.singleExp2 = exp2;

    if(!"".equals(tokenizer.peek())) {
        throw Error.InvalidCommandException;
    }
    return wherePlus;
}
```



## 笛卡尔积

直接把两个表的所有行的数据项查找出来。

然后一个双重for连接起来，形成一个二维byte数组。

同时注意下，遍历过程中用两个list记录下笛卡尔积表的第一行的属性名和属性类型。后面要用的。

```java
public void Cartesian(long xid) throws Exception{
    //先笛卡尔积
    List<Long> uids1 = table1.findAllForSelectPlus();
    List<Long> uids2 = table2.findAllForSelectPlus();

    cartersian = new byte[uids1.size() * uids2.size()][];//初始化成一个不等长的二维数组
    Field fd1 = null,fd2 = null;
    for(Field field:table1.fields) {
        if(field.isIndexed()) {
            fd1 = field;
            break;
        }
    }
    for(Field field:table2.fields) {
        if(field.isIndexed()) {
            fd2 = field;
            break;
        }
    }
    if(fd1 == null || fd2 == null) {
        throw Error.FieldNotFoundException;
    }

    for (int i = 0; i < table1.fields.size(); i++) {
        Field tmp = table1.fields.get(i);
        field_types.add(tmp.fieldType);
        field_name.add(table1.name+"."+tmp.fieldName);
    }
    for (int i = 0; i < table2.fields.size(); i++) {
        Field tmp = table2.fields.get(i);
        field_types.add(tmp.fieldType);
        field_name.add(table2.name+"."+tmp.fieldName);
    }
    int counts = 0;
    for (int i = 0; i < uids1.size(); i++) {
        for (int j = 0; j < uids2.size(); j++) {
            byte[] raw1 = ((TableManagerImpl)tbm).vm.read(xid, uids1.get(i));
            byte[] raw2 = ((TableManagerImpl)tbm).vm.read(xid, uids2.get(j));
            if(raw1 == null || raw2 == null) continue;
            cartersian[counts] = new byte[raw1.length+raw2.length];
            System.arraycopy(raw1,0,cartersian[counts],0,raw1.length);
            System.arraycopy(raw2,0,cartersian[counts], raw1.length,raw2.length);//笛卡尔连接成byte数组形式
            ++counts;
        }
    }
    all_counts = counts;
}
```



## on过滤器

朴实无华的过滤，只需要写个从刚才的笛卡尔积的每行byte数组里面找到需要的属性名的方法即可。

判断过程中把对的元组保留，不对的直接覆盖掉，记录此时的总元组数。

```java
public void onFilter() throws Exception {
        int counts = 0;
//        System.out.println("on过滤之前的all_counts = "+ all_counts);
        if(selectPlus.connectionType.equals("inner")) {
            for (int i = 0; i < all_counts; i++) {
                String filed1 = selectPlus.on1;
                String filed2 = selectPlus.on2;
//                System.out.println("这一行数据的两个部分的字段名分别是："+filed1+" "+filed2);
                byte[] tg1,tg2;
                tg1 = getTarget(filed1,cartersian[i]);
                tg2 = getTarget(filed2,cartersian[i]);
               // System.out.println("这一行数据的两个字段得到的字节数组数据是 ： "+Arrays.toString(tg1)+" "+Arrays.toString(tg2)+" 判断相等："+Arrays.equals(tg1,tg2));
                if(Arrays.equals(tg1,tg2)) {
                    cartersian[counts] = cartersian[i];
                    counts++;
                }
            }
        }
        all_counts = counts;
//        System.out.println("on过滤之后的all_counts = "+all_counts);
    }
```

## where过滤器

实现思路和上面的on过滤器基本一样。也是扫描整个笛卡尔积。

```java
ublic void whereFilter() throws Exception {
//        System.out.println("进入where过滤器");
        int counts = 0;//如果从0开始，跳出后记得是all_counts = counts -1
        WherePlus wherePlus = selectPlus.wherePlus;
        if(wherePlus == null) return;
//        System.out.println("这是之前的 all_counts = "+all_counts);
        for (int i = 0; i < all_counts; i++) {
            SingleExpression singleExp1 = wherePlus.singleExp1;
            SingleExpression singleExp2 = wherePlus.singleExp2;

            byte[] tg1,tg2;
            if(singleExp1 != null) {
                tg1 = getTarget(singleExp1.field,cartersian[i]);
                //System.out.println("查到这一行数据中，第一个比较字段的值 = "+Arrays.toString(tg1));
                if(!wherePlusFlag(tg1,singleExp1)) {
//                    System.out.println("终于有个不匹配的");
                    continue;
                }
            }
           if(singleExp2 != null) {
               tg2 = getTarget(singleExp2.field,cartersian[i]);
               if(!wherePlusFlag(tg2,singleExp2)) {
//                   System.out.println("终于有个不匹配的");
                   continue;
               }
           }

            cartersian[counts] = cartersian[i];
            counts++;
        }
        all_counts = counts;
//        System.out.println("这是之后的 all_counts = "+all_counts);
    }
```

最后返回结果的时候，可以看到原来的项目是用一个map做的，但两表连接，如果是自连接的话，就会有同名的列，后面的值会覆盖前面的值，所以这里就不用它给的方法，简单写个byte数组转String的方法就行了。





## select的全表扫描

实现思路就是仿照上面的，where字段没有索引就把整个表的内容拉下来，然后一个一个判断wher条件。





