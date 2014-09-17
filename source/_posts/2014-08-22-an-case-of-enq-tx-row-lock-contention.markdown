---
layout: post
title: "一次业务设计失误，因唯一性索引导致的enq: TX – row lock contention总结"
date: 2014-08-22 01:14:21 +0800
comments: true
categories: 
---
年前，一个系统发布，刚上线就导致了大量的enq。之前一直没写下来，今天静下心记录一下。  

业务上对某张表T进行insert操作，并且伴随着并发。T表中的XXX字段因为业务需求有唯一性约束，所以这个字段上有唯一性约束并且有唯一索引。而XXX的值需要通过一次额WS调用才能取得，为了提高系统吞吐量，减少响应时间，这里采取异步化的方式。然而业务设计上insert时先给HID字段初始0值，然后再update。由于有唯一性索引的关系，加上并发，导致大量session等待这个行级锁。  
  
我们知道Oracle的索引会根据索引列的值进行排序，而对于唯一性索引来说，每个值在索引上只有一个位置可以写，所以大量的session等到0这个值的索引块的锁释放。  

下面来重现一下这个场景：

1、建一张表T，字段HID建唯一性索引，并insert值为0的记录  
``` sql 1、建一张表T，字段HID建唯一性索引，并insert值为0的记录
yuqing>set sqlp 'SESSION A>'
SESSION A>create table t(xxx number(16));
 
Table created.
 
SESSION A>create unique index t_hid_ind on t(xxx) tablespace xxxxxxxx;
 
Index created.
 
SESSION A>insert into t values(0);
 
1 row created.  
```  
<!-- more -->  
  
2、然后在session B，也执行同样的insert  
``` sql 2、然后在session B，也执行同样的insert  
yuqing>set sqlp 'SESSION B>'
SESSION B>insert into t values(0);
```  
这时候发现，session B hang住了，它在等待session A的事务提交或者回滚  
``` sql
sys>select USERNAME,EVENT,SQL_ID from v$session where username='YUQING';
 
USERNAME   EVENT                                    SQL_ID
---------- ---------------------------------------- -------------
YUQING     SQL*Net message from client
YUQING     enq: TX - row lock contention            70nctf7574d1y

sys>select sql_id,sql_text from v$sql where sql_id='70nctf7574d1y';
 
SQL_ID        SQL_TEXT
------------- ----------------------------
70nctf7574d1y insert into t values(0)
```  

从上面的等待事件可以看出，这里的等待。

解决的方法：

1、业务改造，改为同步方式，接收到正确值以后再insert（牺牲吞吐量和响应时间）

2、最简单的方式，将0改为null  
``` sql
SESSION A>insert into t values(null);
 
1 row created.
 
SESSION A>
 
再另一个session中执行相同的insert
 
SESSION B>insert into t values(null);
 
1 row created.
 
SESSION B>
```  

从上面的实验可以看出，这里将0改为null后并没有产生等待，原因是，在Oracle中，null被定义成为一个不等于任何值的值，也即是说null不等于null，并且null并不会记录索引，唯一键并不需要非空，所以在不确定值的情况采用异步，初始一个null值能达到效果。