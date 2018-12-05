---
layout: post
title: "这竟是索引的'锅'"
author: "flyer0126"
---

最近在开发异地交易可视化项目中，为了做不同城市系统的兼容，需要获取当前所属的城市编码来区分不同appId，利用框架查询表apps结果，与偶然自己手写sql查询表数据对比，发现两次查询结果竟然不一致。  

第一条SQL（框架查询）：  

```
select * from apps limit 1;
```

id  | city_code | company_code | company_name 
--- | --------- | ------------ | ------------
1   | 410100    | ZZLJ8888     | zzlj

第二条SQL（自己查询）：  

```
select id from apps limit 1;
```

id  | 
--- |
2   |

> 注：后文分别简称以上查询为：SQL1、SQL2

为了一探究竟，查阅一下表数据及相关索引如下：

id  | city_code | company_code | company_name
--- | --------- | ------------ | ------------
1   | 410100    | ZZLJ8888     | zzlj
2   | 410100    | HNFD0001     | hnfd

索引如下：
![image](https://flyer0126.github.io/assets/imgs/20180309mysql_index/indexs.png)

> 表中存在两条数据记录，索引分别为`id`列的主键索引和`company_code`列的唯一索引。  

首先，那来看看MySQL本身是如何解释的？

```
explain select * from apps limit 1;
```
![image](https://flyer0126.github.io/assets/imgs/20180309mysql_index/explain_1.png)

```
explain select id from apps limit 1;
```
![image](https://flyer0126.github.io/assets/imgs/20180309mysql_index/explain_2.png)

由此可见，第一条没有用到索引，按主键默认排序，SQL1 取到了第一条；第二条用到了索引`uniq_company_code`，按照索引排序，SQL2 取到了第二条。

总结一下：根据`select`的字段不同，MySQL选取的查询策略不同，导致了不同的展示结果。

看起来这个总结可以解释上面的问题了，但是存在几个疑惑点：  

1. 为什么`SQL2`中并没有出现`company_code`字段，却会使用索引`uniq_company_code`?  
2. 为什么`SQL1`中不会使用索引`uniq_company_code`？

在回答以上问题之前，我们先了解一下MySQL常用表引擎的实现方式。 

MySQL服务器逻辑架构图 如下所示：
![image](https://flyer0126.github.io/assets/imgs/20180309mysql_index/mysql.png)  
最下层的存储引擎负责MySQL中数据的存储和提取。

示例表如下所示：

id   | city_code | company_code 
---- | --------- | ------------
10   | 410100    | ZZLJ8888
21   | 410100    | HNFD0001
32   | 420100    | WH9999
43   | 430100    | CS9999

不同表引擎索引的实现：
![image](https://flyer0126.github.io/assets/imgs/20180309mysql_index/index_store.png)

由此，我们应该可以得到如下结论: 

1. 因为索引`uniq_company_code`中包含`id`字段，SQL2 可以从索引`uniq_company_code`中直接取得数据，所以优化器选择走索引`uniq_company_code`；  
2. 而 SQL1 中`select * ` 选取了在索引`uniq_company_code`中不包含的列，所以无法使用索引`uniq_company_code`。 

为了验证上面的结论，进一步进行试验：
> 假如查询字段包含了`company_name`（索引`uniq_compamy_code`中不包含的字段），则应该无法使用此索引。

```
explain select id, company_name from apps limit 1;
```
![image](https://flyer0126.github.io/assets/imgs/20180309mysql_index/explain_3.png)

至此，验证了索引覆盖的问题（`compamy_name`不在索引`uniq_compamy_code`索引覆盖范围内，导致无法使用其索引）。

那么，MySQL为什么要使用索引覆盖呢？MySQL是下面这么解释的：

```
It is possible that key will name an index that is not present in the possible_keys value. This can happen if none of the possible_keys indexes are suitable for looking up rows, but all the columns selected by the query are columns of some other index. That is, the named index covers the selected 
columns, so although it is not used to determine which rows to retrieve, an index scan is more efficient than a data row scan .
```
主要原因：假如索引覆盖包含了所选取的字段，会优先使用索引覆盖，因为效率更快。

> 不是所有类型的索引都可以成为覆盖索引，覆盖索引必须要存储索引列的值，而哈希索引、空间索引和全文索引等都不存储索引列的值，所以MySQL只能使用B-Tree索引做覆盖索引。另外，不同的存储引擎实现覆盖索引的方式也不同，而且不是所有的引擎都支持索引覆盖。

既然主键索引列包含所有的数据列，那么主键索引一样可以做到索引覆盖，那优化器为什么不选择使用主键索引呢？

在MySQL v5.1.46的优化器在对index选择上做了一点改动，具体描述如下：

```
Performance: While looking for the shortest index for a covering index scan, the optimizer did not consider the full row length for a clustered primary key, as in InnoDB. Secondary covering indexes will now be preferred, making full table scans less likely。
```

该版本中增加了`find_shortest_key()`，该函数的作用是可以认为选择最小的`key_length`的索引来满足我们的查询。
`find_shortest_key()` 函数注释如下：

```
As far as clustered primary key entry data set is a set of all record fields (key fields and not key fields) and secondary index entry data is a union of its key fields and primary key fields (at least InnoDB and its derivatives don’t duplicate primary key fields there, even if the primary and the secondary keys have a common subset of key fields), then secondary index entry data is always a subset of primary key entry. Unfortunately, key_info[nr].key_length doesn’t show the length of key/pointer pair but a sum of key field lengths only, thus we can’t estimate index IO volume comparing only this key_length value of secondary keys and clustered PK. So, try secondary keys first, and choose PK only if there are no usable secondary covering keys or found best secondary key include all table fields (i.e. same as PK)
```
主要原因：辅助索引总是主键的子集，从节约IO的角度，优先选择辅助索引。

为了验证一下主键索引同样可以做索引覆盖，我们将索引`uniq_company_code`删除，然后看看SQL2如何解释：

```
explain select id from apps limit 1;
```

![image](https://flyer0126.github.io/assets/imgs/20180309mysql_index/explain_4.png)


最后，整体做下总结：  

1、没有查询条件时全表扫描，如果可以用到索引，则索引顺序决定结果顺序
> SQL2 用到了索引`uniq_company_code`，结果按字段`company_code`顺序返回结果；SQL1 没有用到索引，按默认主键`id`顺序返回结果。

2、辅助索引包含主键索引字段
> 索引`uniq_company_code`为辅助索引，叶子节点包含`company_code`和`id`字段，所以SQL2 可以使用这个索引，但SQL1 不能。

3、索引覆盖
> 如果索引的叶子节点中已经包含要查询的数据，那么查询只需要在索引文件上进行，不需要再进行回表查询.

4、主键索引和辅助索引都可以覆盖查询列，为何优先使用辅助索引（辅助索引占用空间少，优先使用辅助索引可以节约IO）
> 主键索引`id`和辅助索引`uniq_company_code`两个索引都可以覆盖查询列，优先使用辅助索引`uniq_company_code`.


凡事讲究个实锤，这竟是索引的“锅”，哈哈。




<br>

_The end_