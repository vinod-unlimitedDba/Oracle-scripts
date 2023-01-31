```
Query to find blocking session and blocked session
========================================================

select a.SID "Blocking Session", b.SID "Blocked Session"  
from v$lock a, v$lock b 
where a.SID != b.SID and a.ID1 = b.ID1  and a.ID2 = b.ID2 and 
b.request > 0 and a.block = 1;
```
   ```  
   RAC
   ====
      select a.SID "Blocking Session", b.SID "Blocked Session"  
      from gv$lock a, gv$lock b  where a.SID != b.SID and a.ID1 = b.ID1  and a.ID2 = b.ID2 and  b.request > 0 and a.block = 1;
```

```
Another query that can help you with finding the blocking and blocked sessions
===============================================================================

col blocking_status for a120;
select s1.username || '@' || s1.machine || ' ( SID=' || s1.sid || ' ) is blocking '
 || s2.username || '@' || s2.machine  || ' ( SID=' || s2.sid || ' ) ' AS blocking_status
 from v$lock l1, v$session s1, v$lock l2, v$session s2
 where s1.sid=l1.sid and s2.sid=l2.sid
 and l1.BLOCK=1 and l2.request > 0 and l1.id1 = l2.id1 and l2.id2 = l2.id2 ;
```
   ```  RAC
     ========
     col blocking_status for a120;
    select s1.username || '@' || s1.machine || ' ( SID=' || s1.sid || ' ) is blocking ' || s2.username || '@' || s2.machine  || ' ( SID=' || s2.sid || ' ) ' AS blocking_status
    from gv$lock l1, gv$session s1, gv$lock l2, gv$session s2
    where s1.sid=l1.sid and s2.sid=l2.sid  and l1.BLOCK=1 and l2.request > 0 and l1.id1 = l2.id1 and l2.id2 = l2.id2 ;
```

```To find how long the blocked session is waiting (in minutes)
===============================================================

SELECT 
  blocking_session "BLOCKING_SESSION",  sid "BLOCKED_SESSION",
  serial# "BLOCKED_SERIAL#",seconds_in_wait/60 "WAIT_TIME(MINUTES)"
FROM v$session WHERE blocking_session is not NULLORDER BY blocking_session;
```
   ```     RAC
        ====
        SELECT   blocking_session "BLOCKING_SESSION",sid "BLOCKED_SESSION",serial# "BLOCKED_SERIAL#",  seconds_in_wait/60 "WAIT_TIME(MINUTES)" FROM gv$session 
        WHERE blocking_session is not NULL ORDER BY blocking_session;
```
```To check what SQL is being run by the BLOCKED SESSION inside the database OR which SQL command is waiting
=========================================================================================
SELECT SES.SID, SES.SERIAL# SER#, SES.PROCESS OS_ID, SES.STATUS, SQL.SQL_FULLTEXT
FROM V$SESSION SES, V$SQL SQL, V$PROCESS PRC WHERE
   SES.SQL_ID=SQL.SQL_ID AND  SES.SQL_HASH_VALUE=SQL.HASH_VALUE AND 
   SES.PADDR=PRC.ADDR AND SES.SID=&Enter_blocked_session_SID;
```
   ```   RAC
      ======
      SELECT SES.SID, SES.SERIAL# SER#, SES.PROCESS OS_ID, SES.STATUS, SQL.SQL_FULLTEXT
      FROM gV$SESSION SES, gV$SQL SQL, gV$PROCESS PRC WHERE SES.SQL_ID=SQL.SQL_ID AND
         SES.SQL_HASH_VALUE=SQL.HASH_VALUE AND SES.PADDR=PRC.ADDR AND SES.SID=&Enter_blocked_session_SID;
```
```Run below query to find the table locked, table owner, lock type and other details
===============================================================================
col session_id head 'Sid' form 9999
col object_name head "Table|Locked" form a30
col oracle_username head "Oracle|Username" form a10 truncate 
col os_user_name head "OS|Username" form a10 truncate 
col process head "Client|Process|ID" form 99999999
col owner head "Table|Owner" form a10
col mode_held form a15
select lo.session_id,lo.oracle_username,lo.os_user_name,
lo.process,do.object_name,do.owner,
decode(lo.locked_mode,0, 'None',1, 'Null',2, 'Row Share (SS)',
3, 'Row Excl (SX)',4, 'Share',5, 'Share Row Excl (SSX)',6, 'Exclusive',
to_char(lo.locked_mode)) mode_held
from gv$locked_object lo, dba_objects do
where lo.object_id = do.object_id
order by 5
/
```



