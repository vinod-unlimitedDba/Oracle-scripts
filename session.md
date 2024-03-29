To check which sessions are active in the database
------------------------------------------------------
    select username,osuser,sid,serial#, program,sql_hash_value,module from v$session where username is not null
    and status ='ACTIVE' and module is not null;

   ##### RAC
           
             select username,osuser,sid,serial#, program,sql_hash_value,module from gv$session 
             where username is not null
             and status ='ACTIVE' and module is not null;


----------------------
REM: Script to Get Os user name with terminal name
REM:*****************************************
REM: NOTE: PLEASE TEST THIS SCRIPT BEFORE USE.
REM: Author will not be responsible for any damage that may be cause by this script.
REM:*****************************************
-----------------------
        SELECT
        DBA_USERS.USERNAME USERNAME,
        DECODE(V$SESSION.USERNAME, NULL, 'NOT CONNECTED', 'CONNECTED') STATUS,
        NVL(OSUSER, '-') OSUSER,
        NVL(TERMINAL,'-') TERMINAL,
        SUM(DECODE(V$SESSION.USERNAME, NULL, 0,1)) SESSIONS
        FROM
        DBA_USERS, V$SESSION WHERE DBA_USERS.USERNAME = V$SESSION.USERNAME (+)
        GROUP BY DBA_USERS.USERNAME,
        DECODE(V$SESSION.USERNAME, NULL, 'NOT CONNECTED', 'CONNECTED'),
        OSUSER, TERMINAL ORDER BY 1 ;
        
 --------------

        PROMPT
        PROMPT
        PROMPT NUMBER OF CONNECTED SESSIONS only appliation user not sys
        ------------
        PROMPT =============================
 ---------------       
        
        
        select
               substr(a.spid,1,9) pid,
               substr(b.sid,1,5) sid,
               status,
               TO_CHAR(logon_time,'DD-Mon-YYYY HH24:MI:SS'),
               substr(b.serial#,1,5) ser#,
               substr(b.machine,1,6) box,
               substr(b.username,1,10) username,
               substr(b.osuser,1,8) os_user,
               substr(b.program,1,30) program
        from v$session b, v$process a
        where
        b.paddr = a.addr
        and type='USER'
        order by status;


Script to see particular session waitevent
-------------------------------------------------
         select sid,seq#,wait_time,event,seconds_in_wait,state from gv$session_wait where sid in (&sid);

To list all User Sessions not Background, use following scripts.This script will list you just only User type sessions and their detais.
-------------------------------------------------
    select * FROM gv$session s, gv$process p
    WHERE s.paddr = p.addr(+)
    and s.TYPE ='USER' and s.username!='SYS';

You can list how many Active and Inactive User sessions are in the Oracle database with following script.
-------------------------------------------------
    select count(*) FROM gv$session s, gv$process p
    WHERE s.paddr = p.addr(+) and s.TYPE ='USER' and s.username!='SYS';

You can list only Active User sessions without sys user with following script
-------------------------------------------------
  select count(*) FROM gv$session s, gv$process p
  WHERE s.paddr = p.addr(+) and s.TYPE ='USER' and s.username!='SYS' and status='ACTIVE';
 
You can list only Inactive User sessions without sys user with following script
-------------------------------------------------
    select count(*) FROM gv$session s, gv$process p
    WHERE s.paddr = p.addr(+) and s.TYPE ='USER' and s.username!='SYS' and status='INACTIVE';

You can list all user sessions which are ACTIVE state more than 600 Second with following script.
-------------------------------------------------
    select count(*) FROM gv$session s, gv$process p
    WHERE s.paddr = p.addr(+) and s.TYPE ='USER' and s.username!='SYS' and status='ACTIVE' and last_call_et > 600
    /

Session details associated with SID and Event waiting for
---------------------------------------------------------
      set pages 50000 lines 32767
      col EVENT for a40

      select a.sid, a.serial#, a.status, a.program, b.event,to_char(a.logon_time, 'dd-mon-yy hh24:mi') LOGON_TIME,to_char(Sysdate, 'dd-mon-yy-hh24:mi') CURRENT_TIME,   (a.last_call_et/3600) "Hrs connected" from v$session a,v$session_wait b where a.sid in(&SIDs) and a.sid=b.sid order by 8;


Detail report user session
-------------------------------------------------
            COL orauser HEA "   Oracle User   " FOR a17 TRUNC
            COL osuser HEA " O/S User " FOR a10 TRUNC
            COL ssid HEA "Sid" FOR a4
            COL sserial HEA "Serial#" FOR a7
            COL ospid HEA "O/S Pid" FOR a7
            COL slogon HEA "  Logon Time  " FOR a14
            COL sstat HEA "Status" FOR a6
            COL auth HEA "Auth" FOR a4
            COL conn HEA "Con" FOR a3

            SELECT
               ' '||NVL( s.username, '    ????    ' ) orauser, 
               ' '||s.osuser osuser, 
               LPAD( s.sid, 4 ) ssid, LPAD( s.serial#, 6 ) sserial,
               LPAD( p.spid, 6 ) ospid, 
               INITCAP( LOWER( TO_CHAR( logon_time, 'MONDD HH24:MI:SS' ) ) ) slogon,
               DECODE( s.status, 'ACTIVE', ' Busy ', 'INACTIVE', ' Idle ', 'KILLED', ' Kill ', '  ??  ' ) sstat, 
               DECODE( sc.authentication_type, 'DATABASE', ' DB ', 'OS', ' OS ', ' ?? ' ) auth,
               DECODE( s.server, 'DEDICATED', 'Dir', 'NONE', 'Mts', 'SHARED', 'Mts', '???' ) conn
            FROM
               v$session s, v$process p, 
               (
               SELECT
                  DISTINCT sid, authentication_type
               FROM
                  v$session_connect_info
               ) sc
            WHERE
               s.paddr = p.addr AND s.sid = sc.sid
            ORDER BY
               s.status,s.sid
            /

Total cursors open, by session
-------------------------------------------------
        select a.value, s.username, s.sid, s.serial# from gv$sesstat a,gv$statname b, gv$session s
        where a.statistic# = b.statistic#  and s.sid=a.sid and b.name = 'opened cursors current' order by 1;


KIlling inactive session at os level
-------------------------------------------------

        set pagesize 1200;
        set linesize 1200;
        select 'kill -9 ' || p.spid from v$session s, v$process p where s.paddr = p.addr and s.sid in (select sid from v$session where status like 'INACTIVE' and   logon_time < sysdate-0.33 and action like 'FRM:%');

--------
 ##### RAC
 ---------     
          set pagesize 1200;
          set linesize 1200;
          select 'kill -9 ' || p.spid from gv$session s, gv$process p where s.paddr = p.addr and s.sid in (select sid from gv$session where status like 'INACTIVE' and           logon_time < sysdate-0.33 and action like 'FRM:%');


Kill session for sql_id
-------------------------------------------------
        SELECT 'alter system kill session ' ||''''||SID||','||SERIAL#||' immediate ;'FROM v$session
        WHERE sql_id='&sql_id'; 

----------
 FOR RAC  
------------
        select 'alter system kill session ' ||''''||SID||','||SERIAL#||',@'||inst_id||''''||' immediate ;'  from gv$session where sql_id='&sql_id';
        
===================================================================
Notification Text: The following list of SQL statements have been running for over 60 minutes.
SQL Statement:
-------------------------------------------------
        select 'SID:'||s.sid||', Serial#:'||s.serial#||', Username:'||s.username||', Machine:'||s.machine||
               ', Program:'||s.program||', HashValue:'||s.sql_hash_value||', SQL Text:'||nvl(substr(sql.sql_text,1,40),'Unknown SQL'), last_call_et
        from v$session s
        left outer join v$sql sql on sql.hash_value=s.sql_hash_value and sql.address=s.sql_address
        where s.status='ACTIVE'
        and s.type <> 'BACKGROUND'
        and last_call_et >= 3600;




Script to how active transaction in the database
-------------------------------------------------
        col RBS format a15 trunc
        col SID format 9999
        col USER format a15 trunc
        col COMMAND format a60 trunc
        col status format a8 trunc
        select r.name "RBS", s.sid, s.serial#, s.username "USER", t.status,
        t.cr_get, t.phy_io, t.used_ublk, t.noundo, substr(s.program, 1, 78) "COMMAND" from v$session s, v$transaction t, v$rollname r where t.addr = s.taddr and t.xidusn = r.usn order by t.cr_get, t.phy_io
        /

            RAC
           ------- 
             col RBS format a15 trunc
             col SID format 9999
             col USER format a15 trunc
             col COMMAND format a60 trunc
             col status format a8 trunc
             select r.name "RBS", s.sid, s.serial#, s.username "USER", t.status, t.cr_get, t.phy_io, t.used_ublk, t.noundo, substr(s.program, 1, 78) "COMMAND" from                  gv$session s, gv$transaction t, gv$rollname r where t.addr = s.taddr and t.xidusn = r.usn order by t.cr_get, t.phy_io
             /
