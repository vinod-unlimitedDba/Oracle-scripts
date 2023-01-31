

```
Script gives information about the user sessions locking a particular object
===================================================================
set linesize 1000
column program format a15
column object format a15
select substr(username||'(‘|| se0.sid||’)’,1,5) “User Session”,
substr(owner,1,5) “Object Owner”,
substr(object,1,15) “Object”,
se0.sid,
substr(serial#,1,6) “Serial#”,
substr(program,1,15) “Program”,
logon_time “Logon Time”,
process “Unix Process”
from v$access ac, v$session se0
where ac.sid = se0.sid
and Object = ‘&PACKAGE’
order by logon_time,”Object Owner”,”Object”
/
```

