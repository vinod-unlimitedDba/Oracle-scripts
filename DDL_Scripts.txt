
set pagesize 0
set linesize 30000
set long 500000
set longchunksize 500000
set trimspool on
set feed off


Tablespace DDL:-
====================
select 'select dbms_metadata.get_ddl(''TABLESPACE'','''|| tablespace_name || ''') from dual;' from dba_tablespaces;
select dbms_metadata.get_ddl('TABLESPACE','&TABLESPACE_NAME') from dual;
select 'create tablespace ' || df.tablespace_name || chr(10)
|| ' datafile ''' || df.file_name || ''' size ' || df.bytes
|| decode(autoextensible,'N',null, chr(10) || ' autoextend on maxsize '
|| maxbytes)
|| chr(10)
|| 'default storage ( initial ' || initial_extent
|| decode (next_extent, null, null, ' next ' || next_extent )
|| ' minextents ' || min_extents
|| ' maxextents ' || decode(max_extents,'2147483645','unlimited',max_extents)
|| ') ;'
from dba_data_files df, dba_tablespaces t
where df.tablespace_name=t.tablespace_name
/



Generate DDL of DB_LINK:-
==================================
SET LONG 1000
SELECT DBMS_METADATA.GET_DDL('DB_LINK',db.db_link,db.owner) from dba_db_links db;
SELECT 'create '||DECODE(U.NAME,'PUBLIC','public ')||'database link '||CHR(10)
||DECODE(U.NAME,'PUBLIC',Null, U.NAME||'.')|| L.NAME||chr(10)
||'connect to ' || L.USERID || ' identified by '
||L.PASSWORD||' using ''' || L.host || ''''
||chr(10)||';' TEXT
FROM sys.link$ L, sys.user$ U
WHERE L.OWNER# = U.USER#;


Generate Sequence DDL:-
================================
select 'CREATE SEQUENCE '||SEQUENCE_NAME||chr(10)||
' INCREMENT BY '||INCREMENT_BY||chr(10)||
' START WITH '||LAST_NUMBER||chr(10)||
' MINVALUE '||MIN_VALUE||chr(10)||
' MAXVALUE '||MAX_VALUE||chr(10)||
decode(CYCLE_FLAG,'N',' NOCYCLE','CICLE')||chr(10)||
decode(ORDER_FLAG,'N',' NOORDER','ORDER')||chr(10)||
' CACHE '||CACHE_SIZE|| ';'
from DBA_SEQUENCES where SEQUENCE_OWNER='&OWNER_NAME';



Users DDL:-
=================
set long 2000
select (case
when ((select count(*)
from dba_users
where username = 'ODB') > 0)
then dbms_metadata.get_ddl ('USER', 'ODB')
else to_clob (' -- Note: User not found!')
end ) Extracted_DDL from dual
UNION ALL
select (case
when ((select count(*)
from dba_ts_quotas
where username = 'ODB') > 0)
then dbms_metadata.get_granted_ddl( 'TABLESPACE_QUOTA', 'ODB')
else to_clob (' -- Note: No TS Quotas found!')
end ) from dual
UNION ALL
select (case
when ((select count(*)
from dba_role_privs
where grantee = 'ODB') > 0)
then dbms_metadata.get_granted_ddl ('ROLE_GRANT', 'ODB')
else to_clob (' -- Note: No granted Roles found!')
end ) from dual
UNION ALL
select (case
when ((select count(*)
from dba_sys_privs
where grantee = 'ODB') > 0)
then dbms_metadata.get_granted_ddl ('SYSTEM_GRANT', 'ODB')
else to_clob (' -- Note: No System Privileges found!')
end ) from dual
UNION ALL
select (case
when ((select count(*)
from dba_tab_privs
where grantee = 'ODB') > 0)
then dbms_metadata.get_granted_ddl ('OBJECT_GRANT', 'ODB')
else to_clob (' -- Note: No Object Privileges found!')
end ) from dual
/ spool Riskusercreate.sql
set pagesize 0
set escape on
select 'create user ' || U.username || ' identified ' ||
DECODE(password,
NULL, 'EXTERNALLY',
' by values ' || '''' || password || ''''
)
|| chr(10) ||
'default tablespace ' || default_tablespace || chr(10) ||
'temporary tablespace ' || temporary_Tablespace || chr(10) ||
' profile ' || profile || chr(10) ||
'quota ' ||
decode ( Q.max_bytes, -1, 'UNLIMITED', NULL, 'UNLIMITED', Q.max_bytes) ||
' on ' || default_tablespace ||
decode (account_status,'LOCKED', ' account lock',
'EXPIRED', ' password expire',
'EXPIRED and LOCKED', ' account lock password expire',
null)
||
';'
from dba_users U, dba_ts_quotas Q
-- Comment this clause out to include system and default users
where U.username not in ('SYS','SYSTEM',
'SCOTT','DBSNMP','OUTLN','WKPROXY','WMSYS','ORDSYS','ORDPLUGINS','MDSYS',
'CTXSYS','XDB','ANONYMOUS','OWNER','WKSYS','ODM_MTR','ODM','OLAPSYS',
'PERFSTAT')
and U.username=Q.username(+) and U.default_tablespace=Q.tablespace_name(+)
;
SELECT 'ALTER USER '||username||' QUOTA '||
DECODE(MAX_BYTES, -1, 'UNLIMITED', TO_CHAR(ROUND(MAX_BYTES/1024))||'K')
||' ON '||tablespace_name||';' lne
FROM DBA_TS_QUOTAS
ORDER BY USERNAME;
select 'GRANT '||granted_role||' to '||grantee||
DECODE(ADMIN_OPTION, 'Y', ' WITH ADMIN OPTION;', ';')
from dba_role_privs
where grantee like upper('%&&uname%')
UNION
select 'GRANT '||privilege||' to '||grantee||
DECODE(ADMIN_OPTION, 'Y', ' WITH ADMIN OPTION;', ';')
from dba_sys_privs
where grantee like upper('%&&uname%')
UNION
select 'grant ' || PRIVILEGE || ' on ' || OWNER || '.' ||TABLE_NAME || ' to ' || GRANTEE || ';'
from dba_tab_privs
where grantee like upper('%&&uname%');
set pagesize 100
set escape off
spool off set long 20000 longchunksize 20000 pagesize 0 linesize 1000 feedback off verify off trimspool on
column ddl format a1000
begin
dbms_metadata.set_transform_param (dbms_metadata.session_transform, 'SQLTERMINATOR', true);
dbms_metadata.set_transform_param (dbms_metadata.session_transform, 'PRETTY', true);
end;
/ variable v_username VARCHAR2(30);
exec:v_username := upper('&1');
select dbms_metadata.get_ddl('USER', u.username) AS ddl
from dba_users u
where u.username = :v_username
union all
select dbms_metadata.get_granted_ddl('TABLESPACE_QUOTA', tq.username) AS ddl
from dba_ts_quotas tq
where tq.username = :v_username
and rownum = 1
union all
select dbms_metadata.get_granted_ddl('ROLE_GRANT', rp.grantee) AS ddl
from dba_role_privs rp
where rp.grantee = :v_username
and rownum = 1
union all
select dbms_metadata.get_granted_ddl('SYSTEM_GRANT', sp.grantee) AS ddl
from dba_sys_privs sp
where sp.grantee = :v_username
and rownum = 1
union all
select dbms_metadata.get_granted_ddl('OBJECT_GRANT', tp.grantee) AS ddl
from dba_tab_privs tp
where tp.grantee = :v_username
and rownum = 1
union all
select dbms_metadata.get_granted_ddl('DEFAULT_ROLE', rp.grantee) AS ddl
from dba_role_privs rp
where rp.grantee = :v_username
and rp.default_role = 'YES'
and rownum = 1
union all
select to_clob('/* Start profile creation script in case they are missing') AS ddl
from dba_users u
where u.username = :v_username
and u.profile <> 'DEFAULT'
and rownum = 1
union all
select dbms_metadata.get_ddl('PROFILE', u.profile) AS ddl
from dba_users u
where u.username = :v_username
and u.profile <> 'DEFAULT'
union all
select to_clob('End profile creation script */') AS ddl
from dba_users u
where u.username = :v_username
and u.profile <> 'DEFAULT'
and rownum = 1
/ 

Other scripts for user DDL and roles,priviege
======================================================
set linesize 80 pagesize 14 feedback on trimspool on verify on
select dbms_metadata.get_ddl('USER', 'KPHU000') || '/' usercreate from dual;
SELECT DBMS_METADATA.GET_GRANTED_DDL('ROLE_GRANT','KPHU000') || '/' FROM DUAL;
SELECT DBMS_METADATA.GET_GRANTED_DDL('SYSTEM_GRANT','KPHU000') || '/' FROM DUAL;
SELECT DBMS_METADATA.GET_GRANTED_DDL('OBJECT_GRANT','KPHU000') || '/' FROM DUAL;
select DBMS_METADATA.GET_GRANTED_DDL('TABLESPACE_QUOTA', 'KPHU000') '/' from dual;


Roles,system privs,other privs granted to user:-
==================================================
select dbms_metadata.get_ddl('USER', '&USER') || '/' usercreate from dual;
SELECT DBMS_METADATA.GET_GRANTED_DDL('ROLE_GRANT','&USER') || '/' FROM DUAL;
SELECT DBMS_METADATA.GET_GRANTED_DDL('SYSTEM_GRANT','&USER') || '/' FROM DUAL;
SELECT DBMS_METADATA.GET_GRANTED_DDL('OBJECT_GRANT','&USER') || '/' FROM DUAL;
select DBMS_METADATA.GET_GRANTED_DDL('TABLESPACE_QUOTA', '&USER') '/' from dual;

Single Objects Grants any type of Object :-
==================================================
select 'grant ' || PRIVILEGE || ' on ' || OWNER || '.' ||TABLE_NAME || ' to ' || GRANTEE || ';' from dba_tab_privs where table_name='&Any_type_of_object_name' ;

Password DDL:-
===================
select 'alter user ' || NAME ||' identified by values ''' || password || ''';' from user$;

Cannot reuse the password:-
==============================
declare
userNm varchar2(100);
userpswd varchar2(100);
begin
userNm := upper('&TypeUserNameHere');
select password into userpswd from sys.user$ where name = userNm;
execute immediate ('ALTER PROFILE "FUNCTIONAL_USER" LIMIT
PASSWORD_VERIFY_FUNCTION null
PASSWORD_LIFE_TIME UNLIMITED
PASSWORD_REUSE_TIME UNLIMITED
PASSWORD_REUSE_MAX UNLIMITED');
execute immediate ('alter user '||userNm||' identified by oct152014oct');
execute immediate ('alter user '||userNm||' identified by values '''||userpswd||'''');
execute immediate ('ALTER PROFILE "FUNCTIONAL_USER" LIMIT
PASSWORD_VERIFY_FUNCTION PASSWDCOMPLEXVERIFICATION');
end;
/ ORA-28003: password verification for the specified password failed ORA-20009: Error: You cannot change password SQL> ALTER PROFILE FUNCTIONAL_USER LIMIT PASSWORD_VERIFY_FUNCTION NULL;
Profile altered.
SQL> alter user trial identified by test;
User altered.
SQL> conn trial/test;
ALTER PROFILE "FUNCTIONAL_USER" LIMIT PASSWORD_VERIFY_FUNCTION PASSWDCOMPLEXVERIFICATION;

Profile DDL:-
=============
select ' alter user '||username||' profile '||PROFILE||';' from dba_users;
select 'grant ' || GRANTED_ROLE || ' to ' || ROLE || ';' from role_role_privs where role='&ROLE'
union
select 'grant ' || PRIVILEGE || ' to ' || ROLE || ';' from role_sys_privs where role='&&ROLE'
union
select 'grant ' || PRIVILEGE || ' on ' || OWNER || '.' ||TABLE_NAME || ' to ' || GRANTEE || ';' from dba_tab_privs where GRANTEE='&&ROLE';

Roles DDL:-
===============
set long 20000 longchunksize 20000 pagesize 0 linesize 1000 feedback off verify off trimspool on
column ddl format a1000
begin
dbms_metadata.set_transform_param (dbms_metadata.session_transform, 'SQLTERMINATOR', true);
dbms_metadata.set_transform_param (dbms_metadata.session_transform, 'PRETTY', true);
end;
/ variable v_role VARCHAR2(30);
exec :v_role := upper('&1');
select dbms_metadata.get_ddl('ROLE', r.role) AS ddl
from dba_roles r
where r.role = :v_role
union all
select dbms_metadata.get_granted_ddl('ROLE_GRANT', rp.grantee) AS ddl
from dba_role_privs rp
where rp.grantee = :v_role
and rownum = 1
union all
select dbms_metadata.get_granted_ddl('SYSTEM_GRANT', sp.grantee) AS ddl
from dba_sys_privs sp
where sp.grantee = :v_role
and rownum = 1
union all
select dbms_metadata.get_granted_ddl('OBJECT_GRANT', tp.grantee) AS ddl
from dba_tab_privs tp
where tp.grantee = :v_role
and rownum = 1
/ set long 100000
set longchunksize 200
set heading off
set feedback off
set echo off
set verify off
undefine role
select (case when ((select count(*) from dba_roles
where role = '&&role') > 0)
then dbms_metadata.get_ddl ('ROLE', '&&role')
else to_clob ('Role does not exist')
end ) Extracted_DDL from dual UNION ALL select (case when ((select count(*)
from dba_role_privs where grantee = '&&role') > 0)
then dbms_metadata.get_granted_ddl ('ROLE_GRANT', '&&role')
end ) from dual UNION ALL select (case when ((select count(*)
from dba_role_privs where grantee = '&&role') > 0)
then dbms_metadata.get_granted_ddl ('DEFAULT_ROLE', '&&role')
end ) from dual UNION ALL select (case when ((select count(*)
from dba_sys_privs where grantee = '&&role') > 0)
then dbms_metadata.get_granted_ddl ('SYSTEM_GRANT', '&&role')
end ) from dual UNION ALL select (case when ((select count(*)
from dba_tab_privs where grantee = '&&role') > 0)
then dbms_metadata.get_granted_ddl ('OBJECT_GRANT', '&&role')
end ) from dual;

create a role and assign all privileges to the role:-
========================================
create role L1_SUPPORT;
create role L2_SUPPORT;
create role L3_SUPPORT;
set pagesize 0
set echo off
set trimspool on
set linesize 120
set feedback off
spool grant.sql
select 'grant select,insert,update,delete on ' || a.owner || '."' ||
a.table_name || '"' || CHR(10) ||
' to L2_SUPPORT;' || CHR(10) ||
'grant select on ' || a.owner || '."' ||
a.table_name || '"' || CHR(10) ||
' to L1_SUPPORT;' || CHR(10) ||
'grant select on ' || a.owner || '."' ||
a.table_name || '"' || CHR(10) ||
' to L3_SUPPORT;'
from dba_tables a
where owner in ('SCHEMA_NAME')
order by owner, table_name;
spool off
set feedback on






