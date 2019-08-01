# 《高性能MySQL》读书笔记
## 第5章 创建高性能的索引
### 5.1 索引基础
    
    1.先在索引中找到对应值，在根据匹配的索引记录找到对应的数据行。
    2.索引可以包含一个或多个列的值。如果包含多个列的值，那么列的顺序也很重要。
    
第一点就是查询的实际步骤，第二点讲的是联合索引
#### 5.1.1 索引的类型
##### B-Tree索引
默认的索引类型，实际上InnoDB是B+Tree。
B-Tree和B+Tree的区别主要是非叶子节点是否包含数据。
索引对多个值排序的依据是定义索引时列的顺序，这个回应了5.1中的第二点。
特点：
    
    1.按顺序存储
    2.适合查找范围数据
    3.适合前缀查找——最左前缀查找
    4.ORDER BY操作（按顺序查找）
    
限制：
    
    1.如果不是按照索引的最左列开始查找，则无法使用索引
    2.不能跳过索引中的列
    3.如果查询中有某个列的范围查询，则其右边所有列都无法使用索引优化查找
    
##### Hash索引
基于哈希表实现。
MySQL中只有Memory引擎显示支持哈希索引。
NDB集群引擎支持唯一哈希索引。
特点：
    
    1.精确匹配索引所有列的查询才有效
    2.结构紧凑，查询速度快
    
限制：
    
    1.索引只包含哈希值和行指针
    2.没有按照顺序存储，无法排序
    3.不支持部分索引列匹配查找
    4.只支持等值比较查询，包括=、IN()、<=>，不支持范围查询
    5.冲突越大，查询代价越大
    
创建自定义哈希索引，主要是解决大量字符串精确匹配的问题。手动维护或者通过触发器
处理Hash冲突，必须在WHERE字句中包含哈希值和对应列值
```
    CREATE TABLE pseudohash (
        id int unsigned NOT NULL auto_increment,
        url varchar(255) NOT NULL,
        url_crc int unsigned NOT NULL DEFAULT 0,
        PRIMARY_KEY(id)
    );

    mysql> SELETE id FROM url WHERE url="http://www.mysql.com"
         > AND url_crc=CRC32("http://www.mysql.com")
         
    DELIMITER //
    CREATE TRIGGER pseudohash_crc_ins BEFORE INSERT ON pseudohash FOR EACH ROW BEGIN SET NEW.url_crc=crc32(NEW.url);
    END;
    
    CREATE TRIGGER pseudohash crc_upd BEFORE UPDATE ON pseudohash FOR EACH ROW BEGIN SET NEW.url_crc=crc32(NEW.url);
    END;
    DELIMITER ;     
```
##### 空间数据索引（R-Tree）
存储地理数据
GIS解决方案做的好的是PostgreSQL的PostGIS
##### 全文索引
适用于MATCH AGAINST的操作，不适用于WHERE操作
### 5.2索引的优点

    1.大大减少了服务器需要扫描的数据量
    2.帮助服务器避免排序和临时表
    3.将随机IO变为顺序IO
    
### 5.3 高性能的索引策略
#### 5.3.1 独立的列
指索引列不能是表达式的一部分，也不能是函数的参数
下面这个查询无法使用actor_id列的索引：
```
    mysql> SELECT actor_id FROM sakila.actor WHERE actor_id + 1 = 5;
```
#### 5.3.2 前缀索引和索引选择性
索引选择性指，不重复的索引值和数据表的记录总数（#T）的比值，范围从1/#T到1之间，越高越好
BLOB、TEXT或者很长的VARCHAR，必须使用前缀索引
前缀索引缺点：
    
    1.不能做ORDER BY和GROUP BY
    2.无法使用前缀索引做覆盖扫描
    
创建前缀索引：
```
    mysql> ALTER TABLE sakila.city_demo ADD KEY (city(7));
```
#### 5.3.3 多列索引
在多个列上建立独立的单列索引大部分情况下都不能提高MySQL的查询性能
MySQL5.0以后引入了”索引合并“的策略（index merge），有三个变种：
    
    1.OR条件的联合（union）
    2.AND条件的相交（intersection）
    3.组合前两种情况的联合及相交
该策略实际上说明了表上的索引建得很糟糕
#### 5.3.4 选择合适的索引列顺序
首先按照最左列排序，其次是第二列，等等
pt-query-digst工具的报告中提取”最差“查询，再选择索引顺序
#### 5.3.5 聚簇索引
不是一种单独的索引类型，而是一种数据存储方式
InnoDB中的聚簇索引在同一个结构中保存了B+Tree索引和数据行——更精确的说法在后面
InnoDB中主键索引是聚簇索引，如果没有定义主键，会选择一个唯一的非空索引代替。如果没有这样的索引，会隐式定义一个主键来作为聚簇索引
优点：
    
    1.把相关数据保存在一起
    2.数据访问更快
    3.使用覆盖索引扫描的查询可以直接使用叶节点中的主键值
缺点：
    
    1.数据全部在内存中，没优势
    2.插入速度严重依赖于插入顺序
    3.更新代价高
    4.基于聚簇索引的表在插入新行，或者主键被更新导致需要移动行的时候，可能面临”页分裂（page split）“的问题
    5.可能导致全表扫描变慢
    6.二级索引（非聚簇索引）可能比想象的更大
    7.二级索引访问需要两次索引查找，而不是一次
##### InnoDB和MyISAM的数据分布对比
示例表：

```
    CREATE TABLE layout_test (
        col1 int NOT NULL,
        col2 int NOT NULL,
        PRIMARY KEY(col1),
        KEY(col2)
    );
```        
MyISAM按照插入顺序存储索引，主键索引和其他索引在结构上一样
InnoDB的聚簇索引的叶子节点包括了主键值、事务ID、用于事务和MVCC的回滚指针以及所有剩余的列
##### 在InnoDB表中按主键顺序插入行
无主键，定义一个代理键（surrogate key）作为主键，最简单的使用AUTO_INCREMENT自增列，避免使用UUID——使聚簇索引的插入完全随机
顺序主键造成更坏的结果：
    
    1.高并发工作负载，主键的上界会成为”热点“，因为所有的插入都发生在这里，可能导致间隙锁竞争
    2.AUTO_INCREMENT锁机制，可以更改innode_autoinc_lock_mode的配置
#### 5.3.6 覆盖索引
如果一个索引包含（或者说覆盖）所有需要查询的字段的值，就称之为”覆盖索引“
判断一个查询是否法其了覆盖索引，在EXPLAIN的Extra字段看到”Using Index“
```
    mysql> EXPLAIN SELECT stone_id, file_id FROM sakila.inventory\G
    ****************************1.row**************************
            id : 1
   select_type : SIMPLE
         table : inventory
          type : index
 possible_keys : NULL
           key : idx_store_id_file_id
       key_len : 3
           ref : NULL
          rows : 4673
         Extra : Using Index                
```
索引无法覆盖查询的情况：
    
    1.查询所有的列，`SELECT *`，但可以利用WHERE中的列做覆盖
    2.以通配符开头的LIKE操作，`title like '%APOLLO%'`。如果是最左匹配的LIKE，则可以

可以使用延迟关联（deferred join）来解决上面的两个问题
5.6以后索引条件推送/索引下沉（index condition pushdown）也能解决。简单原理如下：
>索引比较是在InnoDB存储引擎层进行的。而数据表的记录比较first_name条件是在MySQL引擎层进行的。开启ICP之后，包含在索引中的数据列条件(即上述SQL中的first_name LIKE %sal') 都会一起被传递给InnoDB存储引擎，这样最大限度的过滤掉无关的行。

#### 5.3.7 使用索引扫描来做排序
MySQL生成有序的结果：通过排序操作；或者按索引顺序扫描
使用索引扫描的条件：
    
    1.只有当索引的列顺序和ORDER BY字句的顺序完全一致，并且所有列的排序方向（倒序或正序）都一样时
    2.如果查询需要关联多张表，则只有当ORDER BY子句引用的字段全部为第一个表时

#### 5.3.8 压缩（前缀压缩）索引
#### 5.3.9 冗余和重复索引
重复索引是指在相同的列上按照相同的顺序创建的相同类型的索引
#### 5.3.10 未使用的索引
#### 5.3.11 索引和锁
索引让查询锁定更少的行
InnoDB只在访问行的时候对其加锁，在二级缩影上使用读锁，在主键索引上使用写锁
### 5.4 索引案例学习
使用索引来排序，还是先检索数据再排序。使用索引排序会严格限制索引和查询的设计。
#### 5.4.1 支持多种过滤条件
索引和查询要同时优化以达到最佳的平衡
范围查询放到最右边
使用IN()来避免多个内容交叉的索引，但不能滥用
#### 5.4.2 避免多个范围条件
#### 5.4.3 优化排序
对于选择性非常低的列，增加一些特殊的索引来做排序
使用延迟关联，通过使用覆盖索引查询返回需要的主键，再根据这些主键关联原表获得需要的行
### 5.5 维护索引和表
维护表的目的：
    
    1.找到并修复损坏的表
    2.维护准确的索引统计信息
    3.减少碎片

#### 5.5.1 找到并修复损坏的表
命令CHECK TABLE找出表和索引的错误
命令REPAIR TABLE修复损坏的表
#### 5.5.2 更新索引统计信息
命令ANALYZE TABLE重新生成统计信息
命令SHOW INDEX FROM查看索引的基数
#### 5.5.3 减少索引和数据碎片
B-Tree索引可能会碎片化
表的数据存储也可能碎片化，分为三种类型的数据碎片：
    
    1.行碎片（Row fragmentation）
    2.行间碎片（Intra-row fragmentation）
    3.剩余空间碎片（Free space fragmentation）
MyISAM表，三种碎片都可能发生。InnoDB表不会出现行碎片
通过执行OPTIMIZE TABLE或者导出再导入的方式来重新整理数据

## 第6章 查询性能优化
### 6.1 为什么查询速度会慢

