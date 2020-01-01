---
title: MySQL
categories:
  - 数据库
tags:
  - MySQL
  - 索引
  - 锁
  - B+树
date: 2019-12-26 23:07:27
---

# MySQL

三种常见索引：

1. 哈希表：适合等值查询，不适合区间查询；
2. 有序数组：使用二分法查询等值和区间值，复杂度为O(logn)，但只适合静态数据，不再修改的数据；
3. 树：Innodb的索引采用B+树，是一个N叉树，这里N大约为1200.

<!-- more --> 



B树与B+树的区别：

- B+树非叶子节点不存数据，只存索引。那么就可以存储更多的索引，减小树的高度；
- B+树叶子节点之间用链表关联，方便区间查询。



聚簇索引：索引与数据存储在一起；

二级索引（普通索引）：数据不与索引在一起，需要回表查询（即还需要到聚簇索引中再查询一遍）；

联合索引：用多个字段联合建立的索引，可防止回表查询





建议使用自增主键，不要使用业务字段作为主键：

1. 性能方面：自增主键有序，不会造成叶子节点的分裂（业务字段往往无序）；
2. 空间方面：自增主键往往占用空间小于业务字段，可减小索引占用空间。





Buffer Pool

Innodb会在内存中开辟一块区域用以存放索引和数据。这块区域就是Buffer Pool。这块区域越大，Mysql的性能就越接近内存型数据库（如Redis等）。

Innodb的最小存储单位是page，在旧版本中，一个page的大小是16KB。在新版本中，page的大小可设置。Innodb以页为单位从磁盘读入数据放入Buffer Pool中。Buffer Pool使用改进的LRU算法进行缓存淘汰。主要有两点改进：

- 对缓存进行分区域管理，参数`innodb_old_blocks_pct`可调整两区域的比例，可对预读进行优化；
- 对热数据提至头部设置时间阈值，参数` innodb_old_blocks_time `可调整时间阈值，可对全表扫描进行优化。

Buffer Pool被划分为两个区域：new sublist和old sublist。新加入的页会被放到old sublist的头部。若某一页被访问到了，就会被提至new sublist的头部，说明该页是热数据。old sublist的尾部是最冷的数据，随时会被淘汰。



`innodb_old_blocks_pct`：默认为37。该值表示old sublist占整个Buffer Pool的比例。值越小，越接近传统的LRU算法。对于预读，Innodb会将访问数据之后的页也读入Buffer Pool，如果预读的页在后续操作中没使用到，就会挤出宝贵的热点数据。Buffer Pool划分了区域后，无用的预读页对热点数据的影响会降到最低。

` innodb_old_blocks_time `：默认为0。该值表示当新页插入old sublist中，需要停留的时间。过了这个停留时间后的访问才能将该页移动到new sublist头部。设置该参数可防止全表扫描污染热点数据的问题。对于全表扫描，会读入许多页，而这些页在后续的操作中大概率是用不到的，所以这些页不应该被移动到new sublist中。



悲观锁

```mysql
begin;
select status from goods where id=#{id} for update;
update goods set status=#{status} where id=#{id};
commit;
```

排他锁

`select ... for update`

共享锁

`select ... lock in share mode`

乐观锁

通过版本号实现

```mysql
begin;
select (status,version) from goods where id=#{id};

update goods 
set statstatus=#{status}, version=version+1 
where id=#{id} and version=#{version};
commit;
```



Buffer Pool：https://dev.mysql.com/doc/refman/5.5/en/innodb-buffer-pool.html 