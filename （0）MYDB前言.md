# MYDB前言

[toc]

此项目来源于github一开源项目：[CN-GuoZiyang/MYDB: 一个简单的数据库实现](https://github.com/CN-GuoZiyang/MYDB)

## 项目结构

MYDB 分为后端和前端，前后端通过 socket 进行交互。前端（客户端）的职责很单一，读取用户输入，并发送到后端执行，输出返回结果，并等待下一次输入。MYDB 后端则需要解析 SQL，如果是合法的 SQL，就尝试执行并返回结果。不包括解析器，MYDB 的后端划分为五个模块，每个模块都又一定的职责，通过接口向其依赖的模块提供方法。五个模块如下：

- Transaction Manager（TM）
- Data Manager（DM）
- Version Manager（VM）
- Index Manager（IM）
- Table Manager（TBM） 

五个模块的依赖关系如下：
  ![MYDB 模块依赖](.\assets\b59faf9c5e9647dfd9bc2278d687b48f.jpeg)

后面的文档也是按照这个依赖顺序推进的。

每个模块的职责如下：

TM 通过维护 XID 文件来维护事务的状态，并提供接口供其他模块来查询某个事务的状态。
DM 直接管理数据库 DB 文件和日志文件。DM 的主要职责有：1) 分页管理 DB 文件，并进行缓存；2) 管理日志文件，保证在发生错误时可以根据日志进行恢复；3) 抽象 DB 文件为 DataItem 供上层模块使用，并提供缓存。
VM 基于两段锁协议实现了调度序列的可串行化，并实现了 MVCC 以消除读写阻塞。同时实现了两种隔离级别。
IM 实现了基于 B+ 树的索引，BTW，目前 where 只支持已索引字段。（后面自己补了全表扫描和两表连接查询）
TBM 实现了对字段和表的管理。同时，解析 SQL 语句，并根据语句操作表。

## 运行实例

项目运行截图：

![请添加图片描述](.\assets\fc872227f12b7fe05d0796b0002ef710.png)