---
layout: post
title: "expdp/impdp"
date: 2014-08-21 22:29:20 +0800
comments: true
categories: 
---
今天用expdp和impdp做数据迁移，针对整个schema的数据导入导出，也很有幸体验到了exp和expdp在速度上的差别，因为exp要过一层buffer，所以速度上慢特别多，一个67G数据的schema，用expdp大概花了半个小时，而exp半个小时大概生成3G左右的数据，速度上还是相差很大的。

#EXPDP
用expdp之前要在oracle中创建一个directory。    
``` sql directory
conn / as sysdba
create or replace directory dir_name as ‘/path’;
```
然后到os中，用expdp命令导出数据
expdp system/password schemas=schema_name directory=dump dumpfile=out_file_name.dmp logfile=log_file_name.log;  
则会在dump目录下生成dmp文件和log文件，这里的directory就是之前在Oracle中创建的directory。
#IMPDP
       用impdp来导入数据和前面一样，dmp文件就是之前expdp出来的dmp文件。
       impdp system/password schemas=schema_name directory=dump dumpfile=imp_file.dmp logfile=log_imp.log;
       impdp似乎可以直接和expdp连接使用，就可以边导出边导入了，有待研究。
#关于NETWORK_LINK参数
      如果有指定NETWORK_LINK参数，则可以远程传输至目标端，network_link的值为dblink的名字。
      首先，在目标端库建立一个dblink至源端。
      expdp指定network_link参数将可以直接在目标端生成dump文件。
      impdp指定network_link参数，可以直接绕过生成dump文件，直接从源端导出并导入目标端。  
相应的EXP和IMP就简单一点（buffer设大一点快，exp不小于64000，imp不小于100000）  

       exp schema_name/password  BUFFER=640000 FILE=/data/oradata/dump/out_file_name.dmp OWNER=schema_name log=logfile.log  
       imp schema_name/password  BUFFER=640000 FILE=/data/oradata/dump/imp_file_name.dmp  FROMUSER=from_schema TOUSER=schema_name log=logfile.log