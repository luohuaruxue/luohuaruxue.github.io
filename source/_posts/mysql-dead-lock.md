title: mysql 死锁分析
date: 2016-02-27 21:51:02

---

线上服务器告警，使用VPN远程登录数据库，查看当前有哪些线程在运行

```sql
show processlist;
```
一看，果然是发生了死锁
死锁语句如下

```sql
update table set example_row1="0" where id="123" and createTime="2016-02-01 00:00:00"
update table set example_row2="0" where id="123"
```
* 表结构为 id 和 createTime 为是复合主键，而createTime作为分区主键，整个表使用分区存储

这不禁让人费解，为什么更新同一条记录的不同属性(且属性并非为主键)，会导致出现死锁。并且查询条件中，应该均用到索引，且先获取到id这个字段的语句先执行，应该不满足出现死锁的四个必要条件。

依次 explain  一下上面两个SQL语句：


| selectType | table  | type | key | keyLen | ref | rows | Extra |
| --------   | -----:  | :----:  |
| SIMPLE | table | index_merge| createTime,id|6182|NULL|1|Using intersect createTime_index,id_index ,Using where,Using temporary|

| selectType | table  | type | possibleKeys | key | keyLen | ref | rows | Extra |
| --------   | -----:  | :----:  |
| SIMPLE | table | range| PRIMARY,id | PRIMARY |182|const|1|Using where,Using temporary|

令人费解的是，两个update语句，怎么会造成死锁呢。本着我遇到的坑肯定有人已然跳进过的心态，于是查找资料，果不其然：

![Alt text](http://pic.yupoo.com/hedengcheng/DnJ6S9J0/medish.jpg)

>第二个用例，虽然每个Session都只有一条语句，仍旧会产生死锁。要分析这个死锁，首先必须用到本文前面提到的MySQL加锁的规则。针对Session 1，从name索引出发，读到的[hdc, 1]，[hdc, 6]均满足条件，不仅会加name索引上的记录X锁，而且会加聚簇索引上的记录X锁，加锁顺序为先[1,hdc,100]，后[6,hdc,10]。而Session 2，从pubtime索引出发，[10,6],[100,1]均满足过滤条件，同样也会加聚簇索引上的记录X锁，加锁顺序为[6,hdc,10]，后[1,hdc,100]。发现没有，跟Session 1的加锁顺序正好相反，如果两个Session恰好都持有了第一把锁，请求加第二把锁，死锁就发生了。
 
> -- [何登成的技术博客-MySQL 加锁处理分析](http://hedengcheng.com/?p=771)

根据上述分析，我们可以知道上面所说的两个SQL语句，语句1，先获得了索引createTime的锁，再去获取索引id的锁的时候，已然被语句2持有了。于是更新同一条记录的两个SQL语句发生了死锁。

```sql
update table set example_row1="0" where id="123"  and createTime="2016-02-01 00:00:00"
update table set example_row2="0" where id="123"
```

然而当在测试环境的数据库中explain执行造成死锁的两个语句的时候，输出均为:

| selectType | table  | type | possibleKeys | key | keyLen | ref | rows | Extra |
| --------   | -----:  | :----:  |
|  SIMPLE | table | range| PRIMARY,id | PRIMARY |182|const|1|Using where,Using temporary|

这不禁让人觉得诧异，在其他因素均相同的情况下，为什么又不使用到 createTime 的字段索引了呢。
想了想，统计线上和测试环境的同一张表的记录数：

```sql
select count(1) from table;
```

发现线上的数据库表将近千万，而测试环境的数据库只有几十万，数量上差了两个数量级。那么为什么数量的差别会导致没有用到 createTime 这个主键索引呢。
在文章开头的时候有说过：

>表结构为 id 和 createTime 为是复合主键，而createTime作为分区主键，整个表使用分区存储。

因此可以得出推论，Mysql 在更新分区存储表的记录时，如果 where 同时含有 id 和 createTime 的查询条件，会根据分区主键 createTime 先找到记录所在的硬盘分区，再通过 id 找到唯一的那条记录进行更新，使用到聚簇索引。而如果表记录数量少、只存储在同一个分区里，则只使用到主键 id 。

