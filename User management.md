
##### User particular info
		set linesize 2000 verify off
		Set page 1000
		col username  format a15
		col account_status  format a16
		col default_tablespace  format a15
		col temporary_tablespace  format a15
		col profile  format a15
		col initial_rsrc_consumer_group for a23
		SELECT  username,account_status,lock_date,expiry_date,default_tablespace,temporary_tablespace,created,
		profile,initial_rsrc_consumer_group,editions_enabled,
		authentication_type from   dba_users where  username like upper('%&username%') ORDER BY username;

##### 
		SELECT USERNAME, ACCOUNT_STATUS ,CREATED, profile FROM DBA_USERS WHERE USERNAME ='&A';



##### Script for login failed if audit is enabled
	
	set pagesize 200
	set linesize 150
	column os_username format a15
	column username format a15
	column userhost format a40
	column timestamp format a20
	column returncode format 9999999999

	select os_username, username, userhost, to_char(timestamp,'mm/dd/yyyy hh24:mi:ss') timestamp, action_name, returncode from dba_audit_session
	where action_name = 'LOGON' and returncode  0
	order by timestamp ; 

##### script Show all tablespaces used by a user

	select tablespace_name, ceil(sum(bytes) / 1024 / 1024/1024) "GB"
	from dba_extents where owner like '&user_id' group by tablespace_name
	order by tablespace_name
	/


##### GET ALL USERS  NOT DEFAULTS

	Set lines 500 pages 1000
	Col  username for a35
	SELECT username   FROM dba_users  WHERE TRUNC(created) > (SELECT MIN(TRUNC(created)) FROM dba_users);

##### Dynamic script for non default users

	SET HEADING OFF
	spool /export/home/oracle/dropuser.sql

     SELECT 'DROP USER '||USERNAME||' CASCADE ;' FROM DBA_USERS
     WHERE TRUNC(created) > (SELECT MIN(TRUNC(created)) FROM dba_users);
    /
######
 
 ```
SELECT 'DROP USER '||USERNAME||' CASCADE ;' FROM DBA_USERS where username not in 		  ('ANONYMOUS','CTXSYS','DBSNMP','DIP','DMSYS','MDSYS','MGMT_VIEW','OLAPSYS','ORACLE_OCM','ORDPLUGINS','ORDSYS','OUTLN','SI_INFORMTN_SCHEMA','EXFSYS','SQLTXADMIN','SQLTXPLAIN','SYS','SYSTEM','TSMSYS','WMSYS','XDB'); 
```

##### User Info along with what roles granted
		Set linesize 5000 pagesize 50000
		col username format a23 heading 'Username'             justify c
		col role     format a30 heading 'Role (admin,grant)'   justify c
		col dts      format a12 heading 'Default|Tablespace'   justify c
		col tts      format a12 heading 'Temporary|Tablespace' justify c
		col prof     format a30 heading 'Profile'              justify c
		break on username on role on dts on tts
		select
		  username,
		  default_tablespace    dts,
		  temporary_tablespace  tts,
		  profile        prof,
		  granted_role || '-' ||  decode(admin_option,'YES','Y','N') ||  decode(granted_role,'YES','Y','N') role,ACCOUNT_STATUS
		from  dba_users,  dba_role_privs
		where  dba_users.username = dba_role_privs.grantee 
		and username != 'PUBLIC'
		order by   1,2,3,4
		/

###### what roles, system privilege, table pri, on which cloumn assigned for the user ========== Good one

		set line 200 pages 200
		column grantee for a7
		column priv for a25
		column tabnm for a15
		column colnm for a15
		column owner for a7
		set wrap on
		select 'ROLE' typ, grantee grantee, granted_role priv, admin_option ad, '--' tabnm, '--' colnm, '--' owner
		from dba_role_privs
		where grantee='&XNTRG'
		union
		select 'SYSTEM' typ, grantee grantee, privilege priv, admin_option ad, '--' tabnm, '--' colnm,'--' owner
		from dba_sys_privs
		where grantee='&XNTRG'
		union
		select 'TABLE' typ, grantee grantee, privilege priv, grantable ad, table_name tabnm, '--' colnm, owner owner
		from dba_tab_privs
		where grantee='&XNTRG'
		union
		select 'COLUMN' typ, grantee grantee, privilege priv, grantable ad, table_name tabnm, column_name colnm, owner owner
		from dba_col_privs
		where grantee='&XNTRG'
		order by 1;


###### what roles assigned to users

	select granted_role from DBA_ROLE_PRIVS where grantee in (SELECT usr.username FROM sys.dba_users usr WHERE usr.created > (SELECT created FROM 	     sys.v_$database));


##### Dynamic scripts to get what privs given for all roles 

        set long 20000 longchunksize 20000 pagesize 0 linesize 1000 feedback off verify off trimspool on

###### 

	spool file_name.sql
	select 'grant '||privilege||' on '||owner||'.'||table_name||' to '||grantee||';' from DBA_TAB_PRIVS where grantee in (select granted_role from   	DBA_ROLE_PRIVS where grantee in (SELECT usr.username FROM sys.dba_users usr WHERE usr.created > (SELECT created FROM sys.v_$database)));
	spool off


###### GIVES DETAILS OF ALL USER AND ROLES

        col username  format a15
	col "default ts"   format a15
	col "temporary ts" format a15
	col roles format a25
	col profile format a10

	break on username on created on "default TS" on "Temporary TS" on profile

	select username,created,
	       default_tablespace "Default TS",
	       temporary_tablespace "Temporary TS",
	       profile,
	       Granted_role "roles"
	from sys.dba_users,sys.dba_role_privs
	where username = grantee(+)
	order by username;



##### User role and priv for particular user

change at define usercheck to ur  username/Role 

		set lines 110 pages 1000 verify off
		col GRANTEE for a30
		col role for a16
		col pv for a75 head 'PRIVILEGE OR ROLE'
		break on role on type skip 1

		define usercheck = '&DAOTRG'

		select grantee, 'ROL' type, granted_role pv
		from dba_role_privs where grantee = '&usercheck' union
		select grantee, 'PRV' type, privilege pv
		from dba_sys_privs where grantee = '&usercheck' union
		select grantee, 'OBJ' type,
		max(decode(privilege,'WRITE','WRITE,'))||max(decode(privilege,'READ','READ'))||
		max(decode(privilege,'EXECUTE','EXECUTE'))||max(decode(privilege,'SELECT','SELECT'))||
		max(decode(privilege,'DELETE',',DELETE'))||max(decode(privilege,'UPDATE',',UPDATE'))||
		max(decode(privilege,'INSERT',',INSERT'))||' ON '||object_type||' "'||a.owner||'.'||table_name||'"' pv
		from dba_tab_privs a, dba_objects b
		where a.owner=b.owner and a.table_name = b.object_name and a.grantee='&usercheck'
		group by a.owner,table_name,object_type,grantee union
		select username grantee, '---' type, 'empty user ---' pv from dba_users
		where not username in (select distinct grantee from dba_role_privs) and
		not username in (select distinct grantee from dba_sys_privs) and
		not username in (select distinct grantee from dba_tab_privs) and username like '%&usercheck%'
		group by username
		order by grantee, type, pv;


###### List  locked user in the database  Get a list of Locked users 

 
	 col USERNAME for a20
	set lines 750 pages 1000 
	SELECT username, account_status, lock_date, expiry_date
	  FROM DBA_USERS
	 WHERE account_status <> 'OPEN'
	ORDER BY username;

#####
	Set lines750 pages 1000
	Col username for a30
	SELECT 'Administrative User    : ', username, account_status, lock_date, expiry_date
	  FROM dba_users
	 WHERE username in 
	      ('ANONYMOUS', 'CTXSYS',   'DBSNMP', 'EXFSYS', 'LBACSYS',
	       'MDSYS',     'MGMT_VIEW','OLAPSYS','OWBSYS', 'ORDPLUGINS', 
	       'ORDSYS',    'OUTLN',    'SI_INFORMTN_SCHEMA','SYS', 
	       'SYSMAN',    'SYSTEM',   'TSMSYS', 'WK_TEST', 'WKSYS',
	       'WKPROXY',   'WMSYS',    'XDB')
	UNION
	SELECT 'Sample Schema User     : ', username, account_status, lock_date, expiry_date
	  FROM dba_users
	 WHERE username in ('BI','HR','OE','PM','IX','SH')
	union
	SELECT 'Non-Administrative User: ', username, account_status, lock_date, expiry_date
	  FROM dba_users
	 WHERE username in 
	      ('APEX_PUBLIC_USER','DIP',   'FLOWS_30000','FLOWS_FILES','MDDATA',
	       'ORACLE_OCM',      'PUBLIC','SPATIAL_CSW_ADMIN_USER',
	       'SPATIAL_WFS_ADMIN_USR',    'XS$NULL');


##### user and roles

	COL "USER,HIS ROLES AND PRIVILEGES" FORMAT a100
	set linesize 300 pages 1000
	SELECT
	LPAD(' ', 5*level) || granted_role "USER,HIS ROLES AND PRIVILEGES"
	FROM
	(
	  SELECT NULL grantee, username granted_role
	  FROM dba_users
	  WHERE username LIKE UPPER('&USR')
	  UNION
	  SELECT grantee,granted_role
	  FROM dba_role_privs
	  UNION
	  SELECT grantee,privilege
	  FROM dba_sys_privs
	)
	START WITH grantee IS NULL
	CONNECT BY grantee = PRIOR granted_role;   
 
###### PROMPT Table Privileges granted to a user through roles

	Set linesize 1000 pagesize 1000
	Col granted_role for a20
	Col owner for a20
	Col table_name for a35
	SELECT granted_role, owner, table_name, privilege 
	FROM ( SELECT granted_role 
	     FROM dba_role_privs WHERE grantee=UPPER('&username')
	       UNION
	       SELECT granted_role 
	     FROM role_role_privs
	     WHERE role in (SELECT granted_role 
			FROM dba_role_privs WHERE grantee=UPPER('&username')
		       )
	    ) roles, dba_tab_privs
	WHERE granted_role=grantee;

##### PROMPT System Privileges assigned to a user through roles

	SELECT granted_role, privilege
	FROM ( SELECT granted_role 
	     FROM dba_role_privs WHERE grantee=UPPER('&username')
	       UNION
	       SELECT granted_role 
	     FROM role_role_privs
	     WHERE role in (SELECT granted_role 
			FROM dba_role_privs WHERE grantee=UPPER('&username')
		       )
	    ) roles, dba_sys_privs
	WHERE granted_role=grantee;


##### List of privilege

	select
	  lpad(' ', 2*level) || granted_role "User, his roles and privileges"
	from
	  (
	  /* THE USERS */
	    select 
	      null     grantee, 
	      username granted_role
	    from 
	      dba_users
	    where
	      username like upper('%&enter_username%')
	  /* THE ROLES TO ROLES RELATIONS */ 
	  union
	    select 
	      grantee,
	      granted_role
	    from
	      dba_role_privs
	  /* THE ROLES TO PRIVILEGE RELATIONS */ 
	  union
	    select
	      grantee,
	      privilege
	    from
	      dba_sys_privs
	  )
	start with grantee is null
	connect by grantee = prior granted_role;












