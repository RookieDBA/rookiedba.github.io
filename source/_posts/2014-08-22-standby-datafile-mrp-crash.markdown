---
layout: post
title: "备库创建datafile失败，MRP进程crash的处理"
date: 2014-08-22 00:23:08 +0800
comments: true
categories: 
---
今天遇到一个问题，主库加数据文件后备库不久MRP进程crash。

看```alter```日志，发现如下错误：  
``` text
File #85 added to control file as 'UNNAMED00085' because
the parameter STANDBY_FILE_MANAGEMENT is set to MANUAL
The file should be manually created to continue.
Errors with log /data/arch/xxxx/xxxx_1_50043_794444625.arc
MRP0: Background Media Recovery terminated with error 1274
Errors in file /opt/oracle/xxxx/diag/rdbms/xxxx/xxxx/trace/xxxx_pr00_16671.trc:
ORA-01274: cannot add datafile '/data/oradata/xxxx/XXXX_DATA_040.dbf' - file could not be created
Recovery interrupted!
从alter日志上看，是备库的STANDBY_FILE_MANAGEMENT参数设置成了manual，所以主库创建数据文件备库不能自动添加。
```  
从DB上看  
``` text
SQL>select name,status from v$datafile order by 1;

NAME                                                         STATUS
----------------------------------------------------------- -------
/data/oradata/xxxx/xxxx_A1M_000.dbf                         ONLINE
/data/oradata/xxxx/xxxx_A1M_001.dbf                         ONLINE
/data/oradata/xxxx/xxxx_1M_000.dbf                          ONLINE
/data/oradata/xxxx/xxxx_1M_001.dbf                          ONLINE
/data/oradata/xxxx/xxxx_DATA_000.dbf                        ONLINE
/data/oradata/xxxx/xxxx_DATA_001.dbf                        ONLINE
/data/oradata/xxxx/xxxx_DATA_002.dbf                        ONLINE
/data/oradata/xxxx/xxxx_DATA_003.dbf                        ONLINE
/data/oradata/xxxx/xxxx_DATA_004.dbf                        ONLINE
/data/oradata/xxxx/xxxx_DATA_005.dbf                        ONLINE
/data/oradata/xxxx/xxxx_DATA_006.dbf                        ONLINE
/data/oradata/xxxx/xxxx_DATA_007.dbf                        ONLINE
/data/oradata/xxxx/xxxx_DATA_008.dbf                        ONLINE
/data/oradata/xxxx/xxxx_DATA_009.dbf                        ONLINE
/data/oradata/xxxx/xxxx_DATA_010.dbf                        ONLINE
...
/opt/oracle/xxxx/products/11.2.0/dbs/UNNAMED00085        RECOVER 

85 rows selected.
```  
<!--more-->

处理的方式为：  
1、备库重新创建数据文件

``` sql 1、备库重新创建数据文件
SQL>alter database create datafile '/opt/oracle/xxxx/products/11.2.0/dbs/UNNAMED00085' as '/data/oradata/xxxx/xxxx_DATA_040.dbf';
```   
2、设置STANDBY_FILE_MANAGEMENT参数为auto  
```sql 2、设置STANDBY_FILE_MANAGEMENT参数为auto
SQL>alter system set STANDBY_FILE_MANAGEMENT=auto;
```  
3、将备库恢复打开  
``` sql 3、将备库恢复打开
SQL>alter database recover managed standby database disconnect from session;
```
这么处理以后，后续的文件都陆续创建上来，不过唯独这个文件的status是recover状态，recover datafile 85;无效，后续再看看是什么原因，跟进一下。

–跟进
最后那个问题，是因为database的状态为```open```，只需将状态打到```mount```，再恢复就OK了