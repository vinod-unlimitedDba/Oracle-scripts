
when user complain slow in query or database 
-----

1. At os level check resource bottleneck\
   
   - Get the top consuming PID’s (Works in almost all linux related OS). And if it is Windows, please look into task manager.

          ps -eo pcpu,pid,user,args | sort -k 1 -r |head -10
          
  
 Now, you need to pick the SQL_ID mainly for proceeding further. You will be prompted for PID which you picked in above command.\
```
set linesize 2000;
select s.sid,s.serial#, s.inst_id,p.spid, s.SQL_ID, t.SQL_TEXT, s.machine from gv$process p, gv$session s, gv$sqltext t where s.paddr = p.addr and p.spid=&processid and s.SQL_HASH_VALUE = t.HASH_VALUE;
 ``` 
Now, you will be having the tables list. Check when was latest time stamp of the table analyzed.
```
SELECT OWNER, TABLE_NAME, NUM_ROWS, BLOCKS, AVG_ROW_LEN, TO_CHAR(LAST_ANALYZED, ‘MM/DD/YYYY HH24:MI:SS’) FROM DBA_TABLES WHERE TABLE_NAME =’&TABLE_NAME’;   
```

Find whether the table is partitioned table or normal table.
```
select table_name, subpartition_name, global_stats, last_analyzed, num_rows from dba_tab_subpartitions where table_name=’&Tablename’ and table_owner=’&owner’ order by 1, 2, 4 desc nulls last;
```
#2
```
  select TABLE_OWNER, TABLE_NAME, PARTITION_NAME, LAST_ANALYZED from DBA_TAB_PARTITIONS where TABLE_OWNER=’&owner’ and TABLE_NAME=’&Tablename’ order by                 LAST_ANALYZED;
```

Session who is running 
---

         set linesize 200
         set pagesize 100
         clear columns
         col inst for 99999999
         col sid for 9990
         col serial# for 999990
         col username for a12
         col osuser for a16
         col program for a10 trunc
         col Locked for a6
         col status for a1 trunc print
         col "hh:mm:ss" for a8
         col SQL_ID for a15
         col seq# for 99990
         col event heading 'Current/LastEvent' for a25 trunc
         col state head 'State (sec)' for a14
         select inst_id inst, sid , serial# , username
         , ltrim(substr(osuser, greatest(instr(osuser, '\', -1, 1)+1,length(osuser)-14))) osuser
         , substr(program,instr(program,'/',-
         1)+1,decode(instr(program,'@'),0,decode(instr(program,'.'),0,length(program),instr(program,'.')-
         1),instr(program,'@')-1)) program, decode(lockwait,NULL,' ','L') locked, status,
         to_char(to_date(mod(last_call_et,86400), 'sssss'), 'hh24:mi:ss') "hh:mm:ss"
         , SQL_ID, seq# , event,
         decode(state,'WAITING','WAITING '||lpad(to_char(mod(SECONDS_IN_WAIT,86400),'99990'),6)
         ,'WAITED SHORT TIME','ON CPU','WAITED KNOWN TIME','ON CPU',state) state
         , substr(module,1,25) module, substr(action,1,20) action
         from GV$SESSION
         where type = 'USER'
         and audsid != 0 -- to exclude internal processess
         order by inst_id, status, last_call_et desc, sid
         /


what session is running
-----

      col sql_text for a80 word_wrapped
      col inst_id for 9
      break on inst_id
      set linesize 150
      select inst_id, sql_text
      from gv$sqltext
      where sql_id = '&sqlid'
      order by inst_id,piece
      /


            col sql_id for A15
            col sql_text for A150 word_wrapped
            set linesize 170
            set pagesize 300
            SELECT /* findsql */ sql_id, executions, sql_text
            FROM gv$sql
            WHERE command_type IN (2,3,6,7,189)
            AND UPPER(sql_text) LIKE UPPER('%&sql_str%')
            AND UPPER(sql_text) NOT LIKE UPPER('%findsql%')
            /
            
sqlplan details
-----
 /* sqlplan.sql Example: @sqlplan.sql 1t8v91nxtxgjr */
     
      set define '&'
      define sqlid=&1
      col ELAPSED for 99,990.999
      col CPU for 99,990.999
      col ROWS_PROC for 999,999,990
      col LIO for 9,999,999,990
      col PIO for 99,999,990
      col EXECS for 999,990
      col sql_text for a40 trunc
      set lines 200
      set pages 300
      select inst_id,sql_id,child_number child_num ,plan_hash_value,
      round(ELAPSED_TIME/1000000/greatest(EXECUTIONS,1),3) ELAPSED,
      round(CPU_TIME/1000000/greatest(EXECUTIONS,1),3) CPU,EXECUTIONS EXECS,
      BUFFER_GETS/greatest(EXECUTIONS,1) lio,
      DISK_READS/greatest(EXECUTIONS,1) pio,
      ROWS_PROCESSED/greatest(EXECUTIONS,1) ROWS_PROC, sql_text
      from gv$sql
      where sql_id = '&sqlid'
      order by inst_id, sql_id, child_number
      /
      
      select plan_table_output from table(dbms_xplan.display_cursor('&sqlid',NULL,'ADVANCED -PROJECTION -BYTES RUNSTATS_LAST'));          


sqlplan_hist.sql
----
set linesize 200
set pagesize 200
set verify off
set define '&'
define sqlid=&1
 ``` 
  select plan_table_output from table(dbms_xplan.display_awr('&sqlid'));
```

 WHEN was the last time this report ran successfully and in optimal time?
------
##### SQL>@historical_runs sql_id


/*
historical_runs.sql Copyright dbaparadise.com
Example @historical_runs.sql sql_id */

      set linesize 200
      set pagesize 200
      set verify off
      set define '&'
      define sqlid=&1
      col execs for 999,999,999
      col avg_etime for 999,999.999
      col avg_lio for 999,999,999.9
      col begin_interval_time for a30
      col node for 9999
      break on plan_hash_value skip 1
      select sh.snap_id, sh.instance_number inst, sh.begin_interval_time, s.sql_id, s.plan_hash_value,
      nvl(s.executions_delta,0) execs,
      (s.elapsed_time_delta/decode(nvl(s.executions_delta,0),0,1,s.executions_delta))/1000000 avg_etime,
      (s.buffer_gets_delta/decode(nvl(s.buffer_gets_delta,0),0,1,s.executions_delta)) avg_lio
      from DBA_HIST_SQLSTAT S, DBA_HIST_SNAPSHOT SH
      where s.sql_id = '&sqlid'
      and sh.snap_id = s.snap_id
      and sh.instance_number = s.instance_number
      order by 1, 2, 3
      /
