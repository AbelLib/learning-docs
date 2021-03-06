# MySQL存储引擎
mysql是插件式存储引擎，将数据库查询处理及其他系统的任务与数据的存储、处理相分离。

## MySQL的体系结构
1. 客户端
数据库的最上层，和数据库本身并没有太多联系

2. 连接管理器(也叫MySQL服务层)
这一层实现了所有存储引擎无关的特性，对于上一层来说屏蔽了底层不同存储引擎的区别
包括连接管理器，查询缓存，查询解析以及查询优化器。

3. 存储引擎层
MySQL 存储引擎是用来存储和查询数据的。
MySQL提供了一个存储引擎接口，只要实现了接口就可以作为MySQL存储引擎的具体实现。
注意：存储引擎是针对表的，而不是针对库的，一个库中不同表可以使用不同的存储引擎

### MySQL 服务层
Server 层主要包括连接器，查询缓存，分析器，优化器，执行器。
1. 连接器
主要负责 MySQL 客户端和服务端建立连接，连接成功后会获得当前用户的权限，此权限在整个连接期间
都是有效的，直到下次断开重连。
2. 查询缓存
执行查询语句前会查询缓存，如果命中则从缓存中取出数据，本次查询结束；如果没有命中，则继续执行后面
的步骤，并将查询结果存到缓存中来。在 MySQL 8 之后这个部分被移除，因为缓存命中率低而且经常会被清空，
所以实际上效果并不理想。
3. 分析器
对 SQL 语句进行解析，主要是机型语法和语义分析，检查单词是否拼写错误，检查查询的表或者是否存在。
如果分析器检测出错误就会返回 "error in sql" 类似的错误信息，并结束查询。
4. 优化器
对于一条 SQL 语句，MySQL 内部可能存在多种执行方案，如索引的选择，表关联，连接顺序，MySQL 优化器会尝试在执行前找出一个最优的方案。
5. 执行器
执行语句前首先会进行权限判断，如果没有权限会直接返回没有权限的错误。权限通过后会打开表，调用
存储引擎的接口进行查询并返回结果集。

## 常用存储引擎
### MyISAM存储引擎
MySQL5.5之前版本默认的存储引擎，也是系统表和临时表的存储引擎。
MyISAM存储引擎是由MYD和MYI组成
1. 特性
* MyISAM不支持事务
* 并发性和锁级别
    * 读写都要加锁，读写互斥，对读写混合操作性能并不好，写需要加一个表级锁，读也需要加共享锁
* 表损坏修复
    * 可以支持任意因意外关闭而损坏的MyISAM表进行检查和修复，可能造成数据丢失
    * check table TABLE_NAME 进行表的检查
    * repair table TABLE_NAME 修复表
* MyISAM表支持的索引类型
    * 支持全文索引
* MyISAM表支持数据压缩
    * myisampack -b -f myIsam.MYI 对MyISAM表进行压缩
    * 对于压缩的表只支持读操作，不支持写操作
2. 存储文件  
\*.frm文件存放的是结构信息，\*.MYD文件存放的是数据信息，\*.MYI文件存放的是索引信息

3. 限制
* MySQL<5.0时单表默认最大为4G，如果需要存储大表要修改MAX_Rows和AVG_ROW_LENGTH
* 5.0后默认支持256TB

4. 适用场景
* 非事务场景
* 只读类应用
* 空间类应用

### Innodb存储引擎
MySQL5.5.8及以后版本默认存储引擎。
#### Innodb的存储结构
1. 表空间
从InnoDB存储引擎的逻辑存储结构看，所有数据都被逻辑地存放在一个空间中，称为表空间。InnoDB使用
表空间进行数据存储，具体存储在哪个表空间是靠：  
innodb_file_per_table 来控制的，   
ON：独立表空间：tablename.ibd, 数据会存在放在以db文件夹下数据库表名+.bd的文件中   
OFF：系统表空间：ibdataX, X从1到4，默认配置下会有一个ibdata1，初始大小为10MB的系统表

2. 系统表空间和独立表空间要如何选择
* 系统表空间无法简单的收缩文件大小
* 独立表空间可以通过optimize table命令收缩系统文件
* 系统表空间会产生IO瓶颈(由于只有一个文件在对多个文件进行刷新时会产生IO瓶颈)
* 独立表空间可以同时向多个文件刷新数据
* 推荐对Innodb使用独立表空间

3. 系统表空间中的表转移到独立表空间中的方法
* 使用mysqldump导出所有数据库表数据
* 停止MySQL服务，修改(innodb_file_per_table)参数，并删除Innodb相关文件
* 重启MySQL服务，重建Innodb系统表空间
* 重新导入数据

4.从系统表空间导入到独立表空间后系统表空间还剩下什么内容
* Innodb 数据字典信息
* 回滚信息
* 系统事务信息
* 插入缓冲索引页等

#### 什么是锁
1.
* 锁主要作用是管理共享资源的并发访问
* 锁用于实现事务的隔离性

2. 锁的类型
* 共享锁(读锁)
* 独占锁(写锁)

3. 锁的粒度
* 表级锁(开销小，并发低)
* 行级锁(开销大，并发高)

4. 阻塞和死锁
* 什么是阻塞
一个事务中的锁需要等待另一个事务中的锁释放才能继续执行，阻塞是为了事务可以并发并且合理运行
* 什么是死锁
两个或多个事务互相占用对方的资源，导致所有的事务都无法提交

#### Innodb存储引擎的特性
1. Innodb是一种事务性存储引擎
    * 完全支持事务的ACID特性
    * Redo Log(重做日志，用于实现事务的持久性，存放的是已提交事务) 
    和Undo Log(回滚日志，存放的是未提交事务)
2. Innodb支持行级锁
    * 行级锁可以最大程度的支持并发
    * 行级锁是由存储引擎层实现的

#### Innodb存储引擎的适用场景
 Innodb适合于大多数OLTP(联机在线事务)应用
 
### MySQL其他存储引擎
1. CSV存储引擎
存储文件就是CSV文件，数据以文本方式存储在文件中。
2. Archive存储引擎
节约存储空间，数据存储在ARZ后缀的文件，只支持insert和select操作
3. Memory存储引擎
所有数据保存在内存中
4. Federated存储引擎
* 提供了访问远程MySQL服务器上表的方法
* 本地不存储数据，数据全部放到远程服务器上
* 本地需要保存表结构和远程服务器的连接信息

### 如何选择存储引擎
参考条件：
    * 事务
    * 备份
    * 崩溃恢复
    * 存储引擎的特有特性

### 其他常用配置参数
* sync_binlog 控制MySQL如何向磁盘刷新binlog
* tmp_table_size 和 max_heap_table_size控制内临时表大小
* max_connections控制允许最大的连接数
