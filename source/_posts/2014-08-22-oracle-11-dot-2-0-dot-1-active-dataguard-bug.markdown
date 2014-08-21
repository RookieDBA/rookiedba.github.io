---
layout: post
title: "Oracle 11.2.0.1 Active DataGuard的一个Bug"
date: 2014-08-22 00:13:49 +0800
comments: true
categories: 
---
昨天在线下环境遇到一个Oracle的bug。现象为MRP进程中断，导致无法恢复恢复归档。  
重新恢复应用归档，导致600错误，MRP进程中断  
``` sql
recover managed standby database using current logfile disconnect;
```
日志如下：  
```
Completed: ALTER DATABASE RECOVER managed standby database using current logfile disconnect
Media Recovery Log /data/arch/mysid/mysid_1_27059_794419612.arc
Sat Dec 22 19:05:04 2012
Errors in file /opt/oracle/mysid/diag/rdbms/mysid/mysid/trace/mysid_pr0d_8142.trc (incident=626085):
ORA-00600: internal error code, arguments: [kcbr_apply_change_11], [], [], [], [], [], [], [], [], [], [], []
Incident details in: /opt/oracle/mysid/diag/rdbms/mysid/mysid/incident/incdir_626085/mysid_pr0d_8142_i626085.trc
Slave exiting with ORA-600 exception
Errors in file /opt/oracle/mysid/diag/rdbms/mysid/mysid/trace/mysid_pr0d_8142.trc:
ORA-00600: internal error code, arguments: [kcbr_apply_change_11], [], [], [], [], [], [], [], [], [], [], []
Errors in file /opt/oracle/mysid/diag/rdbms/mysid/mysid/trace/mysid_mrp0_8084.trc (incident=625909):
ORA-00600: internal error code, arguments: [kcbr_apply_change_11], [], [], [], [], [], [], [], [], [], [], []
Incident details in: /opt/oracle/mysid/diag/rdbms/mysid/mysid/incident/incdir_625909/mysid_mrp0_8084_i625909.trc
Sat Dec 22 19:05:05 2012
Trace dumping is performing id=[cdmp_20121222190505]
Recovery Slave PR0D previously exited with exception 600
MRP0: Background Media Recovery terminated with error 448
Errors in file /opt/oracle/mysid/diag/rdbms/mysid/mysid/trace/mysid_pr00_8089.trc:
ORA-00448: normal completion of background process
Managed Standby Recovery not using Real Time Apply
Recovery interrupted!
Sat Dec 22 19:05:07 2012
Sweep [inc][626085]: completed
Sweep [inc][625909]: completed
Sweep [inc2][626085]: completed
Sweep [inc2][625909]: completed
Recovered data files to a consistent state at change 9887019691275
Errors in file /opt/oracle/mysid/diag/rdbms/mysid/mysid/trace/mysid_pr00_8089.trc:
ORA-00448: normal completion of background process
Errors in file /opt/oracle/mysid/diag/rdbms/mysid/mysid/trace/mysid_mrp0_8084.trc:
ORA-00600: internal error code, arguments: [kcbr_apply_change_11], [], [], [], [], [], [], [], [], [], [], []
MRP0: Background Media Recovery process shutdown (mysid)
```  
<!--more-->

根据关键字```kcbr_apply_change_11```查找```metalink```，发现这是一个Oracle的bug。  

``` text Bug 10419984 – Active data guard standby gets ORA-600 [kcbr_apply_change_11] [ID 10419984.8]
Bug 10419984 – Active data guard standby gets ORA-600 [kcbr_apply_change_11] [ID 10419984.8]

Rediscovery Notes:
Look for all the following:
- ADG media recovery fails with ORA-00600:[kcbr_apply_change_11]
- primary/ADGstandby configuration
- there has been at least one primary<->standby switchover(the evidence is an earlier “SWITCHOVER TO PHYSICAL
STANDBY” message in the standby alert log)
- in the incident dump tracefile:
- the failing redo change vector header is dumped, and the opcode is 18.1 (OP:18.1)
- in 11.2 look for “Block SCN:” and “Redo SCN:” in 11.1 look for “Redo scn:”
the scns involved will have the following ordering:
cv.SCN < Block_SCN < Redo_SCN
- the block header is dumped and it shows the standby’s version of the block has an ITL entry with:
Flag=U, Lck!=0, Scn/Fsc=<0×0000.Block_SCN.base#>
- if you take the Block_SCN, and if you can map it to a date+time, (eg by using time/scn
data from v$archived_log,or from media recovery messsages in the alert log) then
you should see the standby was running as the primary database at that time/Block_SCN.
- alert logs have the message “Lost write protection disabled” written at database mount time
- after the failure, the ADG media recovery fails the same way even if it is restarted,
but regular media recovery (ie without the database open read-only) is able to apply the failing redo

Workaround

After the problem has happened:
shutdown the ADG standby, then mount it, and do media recovery until it has recovered
past all the redo generated during the hot backup taken on the primary. Then stop media
recovery, and open the database read only, and restart media recovery again.
``` 
 
根据```metalink```上的描述，将ADG的物理备库shutdown，然后startup到mount状态，恢复之前的归档之后，open database，再恢复就OK了。