---
layout: post
title: "mysql事务提交"
author: "flyer0126"
---

MySQL 作为一种关系型数据库，已被广泛应用到互联网诸多项目中。
由于MySQL插件式存储架构，导致开启binlog后，事务提交实质是二阶段提交，来保证存储引擎和binlog的一致。
![MySQL体系结构image](https://flyer0126.github.io/assets/imgs/20171205mysql_submit/mysql_system.jpg "Placeholder image")

## 1、mysql_execute_command
一般常用DML、DCL、DDL语句的执行，均是通过MySQL提供的公共接口mysql_execute_command执行相应SQL来完成。
> DML：Data Manipulation Language（insert、update、delete）
> DCL：Data Control Language（创建用户、删除用户、授权、取消授权）
> DDL：Data Definition Language（对数据库内部的对象进行创建、删除、修改的操作语句）

分析mysql_execute_command接口的执行流程：

```
mysql_execute_command

{
   switch (command)
   {
       case SQLCOM_INSERT:
                mysql_insert();
                break;
       case SQLCOM_UPDATE:
                mysql_update();
                break;
       case SQLCOM_DELETE:
                mysql_delete();
                break;
       ......
   }

   if thd->is_error()  //语句执行错误
     trans_rollback_stmt(thd);
  else
    trans_commit_stmt(thd);
}
```
上述流程中，可以明确执行任何语句，最后都会执行trans_rollback_stmt或transcommit_stmt，这两个分别是回滚和提交操作。
SQL语句提交，对于非自动模式下，主要两个作用：
{% highlight markdown %}
1. 释放autonic锁（主要用来处理多个事务互斥的获取自增序列），无论最后执行语句的提交还是回滚，资源都是立马释放掉的。
2. 标识语句在事务中的位置，方便语句级回滚。执行commit后，可以进入commit流程。
{% endhighlight %}

## 2、事务提交
具体事务提交流程：

```
mysql_execute_command
trans_commit_stmt
ha_commit_trans(thd, FALSE);
{
    TC_LOG_DUMMY:ha_commit_low
        ha_commit_low()   
            innobase_commit
            {

                //获取innodb层对应的事务结构
                trx = check_trx_exists(thd);
                if(单个语句，且非自动提交)
                {
                     //释放自增列占用的autoinc锁资源
                     lock_unlock_table_autoinc(trx);
                     //标识sql语句在事务中的位置，方便语句级回滚
                     trx_mark_sql_stat_end(trx);
                }
                else 事务提交
                {
                     innobase_commit_low()
                     {  
                        trx_commit_for_mysql();
                            <span style="color: #ff0000;">trx_commit</span>(trx); 
                     }

						//确定事务对应的redo日志是否落盘【根据flush_log_at_trx_commit参数，确定redo日志落盘方式】
                    trx_commit_complete_for_mysql(trx);
						trx_flush_log_if_needed_low(trx->commit_lsn);
						log_write_up_to(lsn);
                }
            }
}
```

```
trx_commit

	trx_commit_low
	{
            trx_write_serialisation_history
            {
                trx_undo_update_cleanup //供purge线程处理，清理回滚页
            }

            trx_commit_in_memory
            {
                lock_trx_release_locks //释放锁资源
                trx_flush_log_if_needed(lsn) //刷日志
                trx_roll_savepoints_free //释放savepoints
            }
	}
```

MySQL是通过WAL方式，来保证数据库事务的一致性和持久性，即ACID特性中的C(constistent)和D（durability）。
WAL(Write-Ahead Logging)是一种实现事务日志的标准方法，具体如下：
{% highlight markdown %}
1. 修改记录前，一定要先写日志；
2. 事务提交过程中，一定要保证日志先落盘，才能算事务提交完成。
{% endhighlight %}
通过WAL方式，在保证事务特性的情况下，可以提高数据库的性能。

针对事务提交流程，主要分4个过程：

> 1. `清理undo段信息`，对于innodb存储引擎的更新操作来说，undo段需要purge，这里主要职能是：真正删除物理记录。在执行delete或update操作时，实际就记录没有真正删除，只是在记录上打了一个标记，在事务提交后，purge线程真正删除，释放物理空间。因此，提交过程中会将undo信息假如purge列表，供purge线程处理；
> 
> 2. `释放锁资源`，mysql通过锁互斥机制保证不同事务不同时操作一条记录，事务执行后才会真正释放所有锁资源，并唤醒等待其锁资源的其他事务；
> 
> 3. `刷redo日志`，mysql实现事务一致性和持久性的机制，是通过redo日志落盘操作，保证了即使修改的数据页没有更新到磁盘，只要日志完成了，就能保证数据库的完整性和一致性；
> 
> 4. `清理保存点列表`，每个语句实际都会有一个savepoint（保存点），为了可以回滚到事务的任何一个语句执行前的状态，由于事务都已经提交了，所以保存点可以被清理了。

## 3、存储引擎实现事务
![MySQL体系结构image](https://flyer0126.github.io/assets/imgs/20171205mysql_submit/mysql_binlog.jpg "Placeholder image")
MySQL本身不提供事务支持，而是开放了存储引擎接口，由具体的存储引擎来实现，以InnoDB为例。
存储引擎实现事务的通用方式是基于redo log 和 undo log。redo log记录事务修改后的数据，undo log记录事务修改前的原始数据。

当一个事务执行时，实际发生过程可以简要描述如下：
{% highlight markdown %}
1. 先记录 undo/redo log，确保日志刷到磁盘上持久存储；
2. 更新数据记录，缓存操作并异步刷盘；
3. 提交事务，在redo log中写入commit记录。
{% endhighlight %}

在MySQL执行事务过程中如果因故障中断，可以通过redo log来重做事务或通过undo log来回滚，确保数据的一致性。
这些都是事务性存储引擎来完成的，但binlog不在事务存储范围内，而是由MySQL Server来记录的。

那么就必须保证binlog数据和redo log之间的一致性，所以开启了binlog后实际的事务执行多了一步，即在上述步骤2、3之间补充一条：
{% highlight markdown %}
3. 将事务日志持久化到binlog
{% endhighlight %}

这样的话，只要binlog没写成功，整个事务是需要回滚的，而binlog写成功后即使MySQL Crash了都可以恢复事务并完成提交。
要做到这点，就需要把binlog和事务关联起来，而只有保证了binlog和事务数据的一致性，才能保证主从数据的一致性。
所以binlog的写入过程不得不嵌入到纯粹的事务存储引擎执行过程中，并已内部分布式事务的方式完成两阶段提交。

<br>

_The end_
