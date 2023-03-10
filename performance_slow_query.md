
when user complain slow in query or database 


1. At os level check resource bottleneck\
   
   - Get the top consuming PID’s (Works in almost all linux related OS). And if it is Windows, please look into task manager.

          ps -eo pcpu,pid,user,args | sort -k 1 -r |head -10
          
  
 Now, you need to pick the SQL_ID mainly for proceeding further. You will be prompted for PID which you picked in above command.

set linesize 2000;
select s.sid,s.serial#, s.inst_id,p.spid, s.SQL_ID, t.SQL_TEXT, s.machine from gv$process p, gv$session s, gv$sqltext t where s.paddr = p.addr and p.spid=&processid and s.SQL_HASH_VALUE = t.HASH_VALUE;
  
Now, you will be having the tables list. Check when was latest time stamp of the table analyzed.

SELECT OWNER, TABLE_NAME, NUM_ROWS, BLOCKS, AVG_ROW_LEN, TO_CHAR(LAST_ANALYZED, ‘MM/DD/YYYY HH24:MI:SS’) FROM DBA_TABLES WHERE TABLE_NAME =’&TABLE_NAME’;   


Find whether the table is partitioned table or normal table.

select table_name, subpartition_name, global_stats, last_analyzed, num_rows from dba_tab_subpartitions where table_name=’&Tablename’ and table_owner=’&owner’ order by 1, 2, 4 desc nulls last;


select TABLE_OWNER, TABLE_NAME, PARTITION_NAME, LAST_ANALYZED from DBA_TAB_PARTITIONS where TABLE_OWNER=’&owner’ and TABLE_NAME=’&Tablename’ order by LAST_ANALYZED;









