---
tags:
  - mysql
  - lock
---
### 1. mySql 数据库有哪几种锁
在mysql中, 锁(lock)机制主要分为一下几类, 他们在不同层面(全局, 表, 行, 元数据, 用户自定义)发挥作用.
#### 1)  存储引擎层面的锁
* 表级锁(table lock)
1) MyISAM 默认使用表级锁, 分为共享锁(S 锁)和排他锁(X 锁)
2) 通过 `lock table... READ/WRITE` 对表显示加锁
* 行级锁(row locks)
1) 仅 InnoDB, NDB cluster 等支持
2) 具体包括:
	1) 记录所(record lock):  只锁定满足条件的行
	2) 间隙锁(Gap lock): 锁定索引间的'空白' 区间, 用于防止幻读
	3) Next-Key lock: 记录锁 + 间隙锁的组合, 用于更严格的幻读防护
	4) 自增锁(Auto Inc Lock):  保护自增列分别的连续性
#### 2) 意向锁(Intention Lock)
* 由InnoDB用于实现表级锁与行级锁的协同
* 意向共享锁(IS): 表示事务将在该表上申请共享锁
* 意向排他锁(IX): 表示事务将在该表上申请排他锁

#### 3) 元数据锁(Metadata Locks, MDL)
* 控制对表结构的并发访问
* 任何DDL(如 ALTER TABLE), DML(select/insert) 都要先获得相应的MDL锁, 确保操作的隔离和安全

#### 4) 全局锁(Global Locks)
* 例如`Flush tables with READ LOCK`: 对整个服务器的所有表加锁,  常用于备份场景


#### 5) 用户层面的应用锁(User-level locks/Advisory Locks)
通过函数`GET_LOCK(str, timeout)`, `RELEASE_LOCK(str)` 获取和释放, 属于应用级别的轻量级互斥锁, 不影响底层存储引擎

#### 6) 分区锁(Partition Locks)
* 当表使用分区时(Partition), 可对当个分区加锁,  主要出现在执行与分区相关的DDL 操作.


| 锁类别    | 作用范围       | 典型用途                      |     |
| :----- | :--------- | :------------------------ | :-- |
| 表级锁    | 整张表        | MyISAM 并发控制, 显示LOCK table |     |
| 行级锁    | 索引记录, 间隙   | InnoDB并发控制, 防止幻读          |     |
| 意向锁    | 表(作为行锁的辅助) | 协调表锁与行锁, 提升并发效率           |     |
| 元数据锁   | 表结构        | 保证DDL 与DML 之间的安全隔离        |     |
| 全局锁    | 整个实例       | 热备份, 锁住所有表                |     |
| 用户自定义锁 | 应用层面       | 程序内部的互斥需求, 例如分布式任务调度      |     |
| 分区锁    | 单个分区       | 对分区表进行细粒度的DDL 操作          |     |



### 2. 如何利用mysql实现乐观锁
在MySql中, 乐观锁(Optimistic Lock)通常不依赖底层的锁机制, 而是通过在记录维护一个版本号或时间戳, 并在更新时检查版本是否被修改, 从而检测并发冲突.

#### 1) 基于版本号(version number)
在表中添加一个整型字段(version)
```shell
ALTER TABLE orders ADD COLUMN version INT NOT NULL DEFAULT 0;

# 读取数据时, 一并读取version
select id, mount, status, version FROM orders WHERE id=123;

# 应用业务逻辑后, 带着原始的version发起更新, 条件中强制version=原版本号, 并在成功更新后将版本号+1
update orders set mount=999.99, status='PAID', version=version+1 where id=123 and version=5; # 之前读取到的version号  

# 检查受影响的行数(affected_rows)
## 如果是1, 说明更新成功, 且未发生并发冲突
## 如果是0, 说明在此期间已被其他事务修改, 需要重新读取, 重试或报错告知用户

```


#### 2) 基于时间戳(Timestamp)
```shell
# 在表中添加一个 last_update 时间戳字段
ALTER TABLE products ADD COLUMN last_update TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON update CURRENT_TIMESTAMP;

# 读取数据时, 取出last_update
select id,name,price,last_update from products where id=1;

# 更新时, 条件中检查时间戳未变
update products set price=199 where id=1 AND last_update='2025-05-20 14:33:00';

# 检查受影响行数来判断是否发生冲突

```

#### 3) 应用层`重试`机制

```shell
# 无论使用版本号还是时间戳, 通常在应用层封装一个'读-改-写'+检查 的循环
do {
row = select ... FROM table where id=?;
# 在内存中修改row, 并记下原 version/timestamp
result = update ... where id=? AND version=row.version;

if result.affected_rows ==1:
	# 成功
else:
	# 失败, 可能由并发操作, 重试或向 用户报告冲突
	retry_count++;
} while retry_coount<MAX_RETRY

```

#### 4) 混合事务隔离: 读 已提交 + 自检测
* 将InnoDB的事务隔离级别设置未 READ COMMITTED, 减少曾都和不可重复读.
* 事务内不对行加锁, 仅用上述 '版本检查' 方式保证写时无冲突.
```shell
set session transaction isolation level read committed;

start transaction;
# read data
selct * from inventory where sku='abc';
# update as business
update inventory
	set stock=stock-1, version=version+1
	where sku='abc' and version=7;

# check affect_rows
commit;
```


| 实现方式    | 优点                   | 适用场景               |
| ------- | -------------------- | ------------------ |
| 版本号     | 简单直观, 任何字段更新都集中到一个字段 | 并发更新冲突少,  能容忍少了重试  |
| 时间戳     | 无需手动维护版本号, 自动更新时间戳   | 对`最后修改时间`本身也有业务意义时 |
| 应用层重试   | 整体流程可控, 可加入指数退避等策略   | 冲突率可预估, 可接受少量重试延迟  |
| 低隔离+自检测 | 降低锁冲突, 提升并发性能        | 高并发场景下对读性能由较高要求    |

采用乐观锁方案, 一般要权衡`冲突重试开销`与`锁等待开销`之间的关系: 当冲突概率低, 读多写少, 乐观锁往往能带来更好的吞吐;  如果冲突频繁, 则可能需要考虑使用`行级悲观锁(select...for update)` 或 分库分表来分摊写压力.



### 3. 利用mysql 实现分布式锁



### 4. 什么是幻读,脏读及如何解决













