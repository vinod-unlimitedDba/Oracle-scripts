
space for temp
=============
     select tablespace_name
     ,tablespace_size/1024/1024 mb_size
     ,allocated_space/1024/1024 mb_alloc
     ,free_space/1024/1024 mb_free
     from dba_temp_free_space;

Block wise Check
===========================================================

   select TABLESPACE_NAME, TOTAL_BLOCKS, USED_BLOCKS, MAX_USED_BLOCKS, MAX_SORT_BLOCKS, FREE_BLOCKS from V$SORT_SEGMENT;

    select sum(free_blocks) from gv$sort_segment where tablespace_name = 'TEMP';


To check instance-wise total allocated, total used TEMP for both rac and non-rac
===========================================================

             set lines 152
             col FreeSpaceGB format 999.999
             col UsedSpaceGB format 999.999
             col TotalSpaceGB format 999.999
             col host_name format a30
             col tablespace_name format a30
             select tablespace_name,
             (free_blocks*8)/1024/1024 FreeSpaceGB,
             (used_blocks*8)/1024/1024 UsedSpaceGB,
             (total_blocks*8)/1024/1024 TotalSpaceGB,
             i.instance_name,i.host_name
             from gv$sort_segment ss,gv$instance i where ss.tablespace_name in (select tablespace_name from dba_tablespaces where contents='TEMPORARY') and
             i.inst_id=ss.inst_id;



Viewing SQL That Is Consuming Temporary Space
===========================================
         SELECT s.sid, s.serial#, s.username
         ,p.spid, s.module, p.program
         ,SUM(su.blocks) * tbsp.block_size/1024/1024 mb_used
         ,su.tablespace
         FROM v$sort_usage su
         ,v$session s
         ,dba_tablespaces tbsp
         ,v$process p
         WHERE su.session_addr = s.saddr
         AND su.tablespace = tbsp.tablespace_name
         AND s.paddr = p.addr
         GROUP BY
         s.sid, s.serial#, s.username, s.osuser, p.spid, s.module,
         p.program, tbsp.block_size, su.tablespace
         ORDER BY s.sid; 



Track Who is Currently using the Temp and how many statments
======================================================
      SELECT b.tablespace, ROUND(((b.blocks*p.value)/1024/1024),2)||'M' "SIZE",
      a.sid||','||a.serial# SID_SERIAL, a.username, a.program
      FROM sys.v_$session a, sys.v_$sort_usage b, sys.v_$parameter p
      WHERE p.name  = 'db_block_size' AND a.saddr = b.session_addr
      ORDER BY b.tablespace, b.blocks;

find out which SQL statement is using up space in a sort segment.
==========================================
        select s.sid || ',' || s.serial# sid_serial, s.username,
        o.blocks * t.block_size / 1024 / 1024 mb_used, o.tablespace,
        o.sqladdr address, h.hash_value, h.sql_text
        from v$sort_usage o, v$session s, v$sqlarea h, dba_tablespaces t
        where o.session_addr = s.saddr
        and o.sqladdr = h.address (+)
        and o.tablespace = t.tablespace_name
        order by s.sid;

— Temp segment usage per session along with os user
===========================================================

        col SID_SERIAL for a15
        col MB_USED format 999999
         col SPID for a15
        col PROGRAM for a30
        col MODULE for a20
        col OSUSER for a15
        Set lines 999 pages 1000
        SELECT S.sid || ',' || S.serial# sid_serial, S.username, S.osuser, P.spid, S.module,
        P.program, SUM (T.blocks) * TBS.block_size / 1024 / 1024 /1024 mb_used, T.tablespace,
        COUNT(*) statements
        FROM v$sort_usage T, v$session S, dba_tablespaces TBS, v$process P
        WHERE T.session_addr = S.saddr
        AND S.paddr = P.addr
        AND T.tablespace = TBS.tablespace_name
        GROUP BY S.sid, S.serial#, S.username, S.osuser, P.spid, S.module,
        P.program, TBS.block_size, T.tablespace
        ORDER BY sid_serial;

— Check the sessions that use temp tablespace along with segment type
===========================================================

col hash_value for a40
col tablespace for a10
col username for a15
set linesize 132 pagesize 1000

     SELECT s.sid, s.username, u.tablespace, s.sql_hash_value||'/'||u.sqlhash hash_value, u.segtype, u.contents, u.blocks
     FROM v$session s, v$tempseg_usage u
     WHERE s.saddr=u.session_addr
     order by u.blocks;

--BTW, v$sort_usage is same as v$tempseg_usage.
--However, the tempspace can be used by any open cursor in that session. The current SQL is not necessary the culprit. In that case, we can check it from v$sql:
===========================================================

       col hash_value for 999999999999
       select hash_value, sorts, rows_processed/executions
        from v$sql
        where hash_value in (select hash_value from v$open_cursor where sid=31)
        and sorts > 0
        and PARSING_SCHEMA_NAME='MERCURY'
        order by rows_processed/executions;




Killing session using TEMP 
================================
        SELECT  b.TABLESPACE, a.username , a.osuser , a.program , a.status ,
               'ALTER SYSTEM KILL SESSION '''||a.SID||','||a.SERIAL#||',@'||a.inst_ID||''' IMMEDIATE;'
            FROM gv$session a
               , gv$sort_usage b
               , gv$process c
               , gv$parameter p
           WHERE p.NAME = 'db_block_size'
             AND a.saddr = b.session_addr
             AND a.paddr = c.addr
             -- AND b.TABLESPACE='TEMP'
        ORDER BY a.inst_ID , b.TABLESPACE
               , b.segfile#
               , b.segblk#
               , b.blocks;

