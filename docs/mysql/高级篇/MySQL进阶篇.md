# 						MySQL进阶篇

## 一、存储引擎

### 1.MySQL体系结构

![txjg](./images/txjg.jpg)

> 连接层 -》服务层 -》引擎层 （索引在引擎层）-》存储层

![txjg2](./images/txjg2.jpg)

### 2.存储引擎简介

![ccyq](./images/ccyq.jpg)

战斗机有战斗机的引擎、直升机有直升机的引擎、火箭有火箭的引擎。不同的引擎适用于不同的场景，像战斗机的引擎适用于战斗机这种高空作战的情况、火箭的引擎适用于火箭在外太空飞行，因此存储引擎不分好与坏，看适用的场景而已。

![ccyq2](./images/ccyq2.jpg)



MySQL5.5版本及之后默认支持的存储引擎是Innodb，因此在创建表的时候不指定存储引擎则默认是Innodb存储引擎。

### 3.存储引擎特点

#### 1）Innodb存储引擎

![innodb](./images/innodb.jpg)

- 事务、行级锁、外键。
- 8.0版本之前表结构存储在以frm为后缀的文件中。
- 参数innodb_file_per_table决定是否每张表各用自己的表空间还是公用一个表空间。8.0版本默认是开启的，代表的是各自用各自的表空间。可以通过show variables like 'innodb_file_per_table'命令查看是否开启。

![innodb2](./images/innodb2.jpg)

表空间 -》段 -》区（1M） -》页(16K）-》行

#### 2）MyISAM存储引擎

![myisam](./images/myisam.jpg)

#### 3）Memory存储引擎

![memory](./images/memory.jpg)

因为表数据信息存储在内存中，因此只有sdi文件存储在磁盘中用于存储表结构信息。

#### 4）三种存储引擎的总体特点

![bj](./images/bj.jpg)

> 面试题，Innodb和MyIsam存储引擎的不同？
>
> ​	Innodb支持事务、行级锁、外键；MyIsan不支持事务、表锁、不支持外键。

## 4.存储引擎选择

![ccyqxz](./images/ccyqxz.jpg)



MyIAM被MogodB取代；Memory被Redis取代。

## 二、索引

### 1.索引的概述

![sygs](./images/sygs.jpg)

![sygsz](./images/sygsz.jpg)

![sygs3](./images/sygs3.jpg)

### 2.索引结构

![syjg](./images/syjg.jpg)

![syzc](./images/syzc.jpg)

Full-text即全文索引

#### 1.二叉树

![ecs](./images/ecs.jpg)



#### 2.B-Tree

![B-tree](./images/B-tree.jpg)

```
https://www.cs.usfca.edu/~galles/visualization/Algorithms.html
```

#### 3.B+Tree

![B+Tree](./images/B+Tree.jpg)

mysql对B+tree进行了优化：

![MySQL-B+Tree](./images/MySQL-B+Tree.jpg)

#### 4.Hash索引

![Hash](./images/Hash.jpg)



![hash2](./images/hash2.jpg)

![mst](./images/mst.jpg)

### 3.索引分类

![syfl](./images/syfl.jpg)

![syfl2](./images/syfl2.jpg)

二级索引也叫非聚集索引

![ys](./images/ys.jpg)

聚集索引叶子节点挂的是一行数据，而非聚集索引叶子节点挂的是id

![fjjsy](./images/fjjsy.jpg)

走非聚集索引会回表查询，所以走了两次查询（不一定走非聚集索引都要回表查询，上图select * 查询，由于非聚集索引叶子节点存储的id值，并没有存储其他字段，所以需要再走一次回表查询，如果是select id，则直接返回数据，无需回表查询了）

### 4.索引语法

![syyf](./images/syyf.jpg)

### 5.SQL性能分析

##### 1）SQL执行频率

![xnfx](./images/xnfx.jpg)

show global status like 'Com_______';

如果查询的次数占的比例比较大，则存在优化的空间，否则没有。

##### 2）慢日志查询

![mcxrz](./images/mcxrz.jpg)

通过show variables like 'slow_query_log' 来判断慢查询日志是否开启

![mcxrz1](./images/mcxrz1.jpg)

##### 3）profile详情

![profile](./images/profile.jpg)

![profile2](./images/profile2.jpg)

##### 4）explain执行计划

![zxjh](./images/zxjh.jpg)

![zxjh2](./images/zxjh2.jpg)

![zxjh3](./images/zxjh3.jpg)

### 6.索引使用（explain)

##### 1）最左前缀法则

![sysy](./images/sysy.jpg)

##### 2）范围查询

![sysy2](./images/sysy2.jpg)

profession、age、status为联合索引 create index index_pro_age_status on tb_user(profession,age,status),注意创建联合索引各个字段的顺序，这个顺序是很重要的。第一条sql由于age>30，则会导致(profession,age,status)中的status索引失效，因此为了让其生效，可以使用>=、或者<=来让status索引生效。

##### 3)索引列运算

![sysy3](./images/sysy3.jpg)

##### 4）字符串不加引号

![sysy4](./images/sysy4.jpg)

##### 5）模糊查询

![sysy5](./images/sysy5.jpg)

##### 6）or连接的条件

![sysy6](./images/sysy6.jpg)

##### 7)数据分布影响

![sysy7](./images/sysy7.jpg)

如果表中的大部分数据都满足使用索引的情况，则走全表扫面，而不走索引。比如如果插入的数据的age是递增的，加入第一条数据的age为1，而表中有10000条数据，最后一条数据的age为10000，当使用age > 1 时，则表中的数据大部门都满足这个条件，则不走索引,直接全表扫描。

##### 8）SQL提示，指定【建议】或者【忽略】或者【强制】使用索引。

如果表中的某一列既存在联合索引，又存在单列索引。那么使用该列时，数据库以某种算法决定使用联合索引或者单列索引。假如这种算法决定使用的是联合索引，但发现走的联合索引没有单列索引走的快，这是我们想让其使用单列索引，则需要用到下面的关键字，指定sql要走哪种索引。

![sysy8](./images/sysy8.jpg)

##### 9）覆盖索引（避免回表查询）

![sysy9](./images/sysy9.jpg)

##### 10)覆盖索引(避免回表查询)

![sysy10](./images/sysy10.jpg)

第一条sql，用的id及聚集索引，聚集索引叶子节点挂的是整行数据的记录，select * 要查询的字段都在这整行记录中，因此不需要回表查询。

第二条sql，用的是辅助索引即非聚集索引或者二级索引，叶子节点挂的数据是行的id，select id,name要查询的数据在叶子节点的key(name),value(id)中，因此不需要回表查询。

第三条sql，用的是辅助索引即非聚集索引或者二级索引，叶子节点挂的数据是行的id，select id,name,gender要查询的数据在叶子节点的key(name),value(id)存在，但不存在gender，因此依旧需要回表查询，利用得到的id再去聚集索引中查询gender数据。

![sysy10.1](./images/sysy10.1.jpg)



##### 11)前缀索引（解决大文本字段作为索引占大量磁盘空间的问题）![sysy11](./images/sysy11.jpg)

![sysy11.1](./images/sysy11.1.jpg)

首先取出email='lvbu666@163.com'的前五个元素lvbu6，然后通过二级索引（辅助索引或者非聚集索引）找到数据对应的id，回表查询一级索引（聚集索引），取出整行数据拿到实际的email值，然后在用lvbu666@163.com匹配这行数据的email，如果值相同，则匹配成功返回数据，如果匹配不成功，则通过链表继续寻找。

##### 12）单列索引和联合索引

![sysy12](./images/sysy12.jpg)

phone和name都存在索引，发现实际结果只走了phone的索引，name的索引没有被使用到。Extra为NULL证明回表查询了。

![sysy12.1](./images/sysy12.1.jpg)

创建联合索引，并设置推荐使用联合索引，结果是实际确实用到了联合索引。

![sysy12.2](./images/sysy12.2.jpg)

像上述图中phone和name的联合索引，首先按照phone进行索引排序，如果phone相同则再比较name。

### 7.索引设计原则

![sysjyz](./images/sysjyz.jpg)

## 三、SQL优化

### 1.插入数据



![yh1](./images/yh1.jpg)

![yh1.1](./images/yh1.1.jpg)

mysql --local-infile -h 8.155.53.241 -P 3306 -uroot -p

load data local infile 'C://Users//wuzhaosheng//Desktop//demo.log' into table `demo` fields terminated by ',' lines terminated by '/n';

![yh1.2](./images/yh1.2.jpg)

### 2.主键优化（主键顺序插入，避免出现页分裂）

具体见https://www.bilibili.com/video/BV1Kr4y1i7ru?spm_id_from=333.788.player.switch&vd_source=d368586e770cb2fcb2a5035e37683bff&p=90的33集

### 3.order by 优化

具体见https://www.bilibili.com/video/BV1Kr4y1i7ru?spm_id_from=333.788.player.switch&vd_source=d368586e770cb2fcb2a5035e37683bff&p=90的34集

![yh2](./images/yh2.jpg)

![yh2.1](./images/yh2.1.jpg)

### 4.group by优化

排序的字段建立索引即可。

### 5.limit 优化（利用覆盖索引，避免回表查询）

![yh3](./images/yh3.jpg)

### 6.count优化

![yh4](./images/yh4.jpg)

![yh5](./images/yh5.jpg)

![yh6](./images/yh6.jpg)

面试题：

https://blog.csdn.net/m0_69057918/article/details/131047753?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-4-131047753-blog-101013526.235^v43^pc_blog_bottom_relevance_base2&spm=1001.2101.3001.4242.3&utm_relevant_index=7

### 7.update优化

具体见https://www.bilibili.com/video/BV1Kr4y1i7ru?spm_id_from=333.788.player.switch&vd_source=d368586e770cb2fcb2a5035e37683bff&p=90的38集

![yh7](./images/yh7.jpg)

## 四、视图/存储过程/触发器

### 1.视图

![stdy](./images/stdy.jpg)

![stcz](./images/stcz.jpg)

详见：https://www.bilibili.com/video/BV1Kr4y1i7ru?spm_id_from=333.788.player.switch&vd_source=d368586e770cb2fcb2a5035e37683bff&p=97

### 2.存储过程

#### 1）存储过程介绍

![ccgc](./images/ccgc.jpg)

#### 2）存储过程的特点

![ccgctd](./images/ccgctd.jpg)



#### 3）存储过程使用

![ccgccz](./images/ccgccz.jpg)

![ccgccz2](./images/ccgccz2.jpg)

![bl](./images/bl.jpg)

![bl2](./images/bl2.jpg)

![bl3](./images/bl3.jpg)

![if](./images/if.jpg)

![cs](./images/cs.jpg)



![case](./images/case.jpg)

![where](./images/where.jpg)



![repeat](./images/repeat.jpg)

![loop](./images/loop.jpg)

![cursor](./images/cursor.jpg)

具体见：https://www.bilibili.com/video/BV1Kr4y1i7ru?spm_id_from=333.788.player.switch&vd_source=d368586e770cb2fcb2a5035e37683bff&p=113

![tjclcx](./images/tjclcx.jpg)

### 3.存储函数

![cchs](./images/cchs.jpg)

### 4.触发器（多看看）

https://www.bilibili.com/video/BV1Kr4y1i7ru?spm_id_from=333.788.player.switch&vd_source=d368586e770cb2fcb2a5035e37683bff&p=116

## 五、锁（多看视频）

### 1.锁的概念

![](./images/lock.jpg)

### 2.锁的分类

![](./images/sfl.jpg)

#### 1)全局锁

![](./images/qjs.jpg)

![](./images/qjscz.jpg)

注意mysqldump  -uroot -p123456 redis_test > C:/Users/wuzhaosheng/Desktop/redis_test.sql在windows命令行中执行，且结尾不要带分号。

![](./images/qjstd.jpg)

#### 2)表级锁

![](./images/bs.jpg)

##### 1）表锁

![](./images/bszs.jpg)

##### 2)元数据锁

![](./images/ysjs.jpg)

元数据即表的结构。

##### 3）意向锁

详细见：https://www.bilibili.com/video/BV1Kr4y1i7ru?spm_id_from=333.788.player.switch&vd_source=d368586e770cb2fcb2a5035e37683bff&p=126

![](./images/yxsjs.jpg)

#### 3)行级锁

https://www.bilibili.com/video/BV1Kr4y1i7ru?spm_id_from=333.788.player.switch&vd_source=d368586e770cb2fcb2a5035e37683bff&p=129

![](./images/hs.jpg)

## 六、InnoDB引擎


## 七、MySQL管理
