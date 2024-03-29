
Finding tablespace usage in GB
==================================
```
     select b.tablespace_name, tbs_size SizeGB, a.free_space FreeGB from
     (select tablespace_name, round(sum(bytes)/1024/1024/1024,1) as free_space
     from dba_free_space group by tablespace_name UNION
     select tablespace_name, round((free_space)/1024/1024/1024,1) as free_space from dba_temp_free_space) a,
     (select tablespace_name, sum(bytes)/1024/1024/1024 as tbs_size
     from dba_data_files group by tablespace_name UNION
     select tablespace_name, sum(bytes)/1024/1024/1024 tbs_size
     from dba_temp_files group by tablespace_name ) b where a.tablespace_name(+)=b.tablespace_name 
/ 
```

2 =====> USAGE shows in graph (used column)
=================================================
```
        set pages 9999 lines 900
        col "% Used" for a6	
        col "Used" for a40
        select t.tablespace_name, t.gb "TotalGB", t.gb - nvl(f.gb,0) "UsedGB", nvl(f.gb,0) "FreeGB",lpad(ceil((1-nvl(f.gb,0)/decode(t.gb,0,1,t.gb))*100)||'%', 6) "% Used", t.ext "Ext", '|'||rpad(lpad('#',ceil((1-nvl(f.gb,0)/decode(t.gb,0,1,t.gb))*20),'#'),20,' ')||'|' "Used"
        from (
        select tablespace_name, trunc(sum(bytes)/(1024*1024*1024)) gb
        from dba_free_space
        group by tablespace_name
        union all
        select tablespace_name, trunc(sum(bytes_free)/(1024*1024*1024)) gb
        from v$temp_space_header
        group by tablespace_name) f, (select tablespace_name, trunc(sum(bytes)/(1024*1024*1024)) gb, max(autoextensible) ext
        from dba_data_files
        group by tablespace_name
        union all
        select tablespace_name, trunc(sum(bytes)/(1024*1024*1024)) gb, max(autoextensible) ext
        from dba_temp_files
        group by tablespace_name) t
        where t.tablespace_name = f.tablespace_name (+)
        order by t.tablespace_name;
```
 Monitoring all puggable tablespace in 12c
===========================================
```
SET LINES 1320 PAGES 1000
COL con_name        FORM A15 HEAD "Container|Name"
COL tablespace_name FORM A15
COL apm             FORM 999,999,999,999 HEAD "Alloc|Space GM."
COL fsm             FORM 999,999,999,999 HEAD "Free|Space GM."
--
COMPUTE SUM OF fsm apm ON REPORT
BREAK ON REPORT ON con_id ON con_name ON tablespace_name
--
WITH x AS (SELECT c1.con_id, cf1.tablespace_name, SUM(cf1.bytes)/1024/1024/1024 fsm
           FROM cdb_free_space cf1
               ,v$containers c1
           WHERE cf1.con_id = c1.con_id
           GROUP BY c1.con_id, cf1.tablespace_name),
     y AS (SELECT c2.con_id, cd.tablespace_name, SUM(cd.bytes)/1024/1024/1024 apm
           FROM cdb_data_files cd
               ,v$containers c2
           WHERE cd.con_id = c2.con_id
           GROUP BY c2.con_id
                   ,cd.tablespace_name)
SELECT x.con_id, v.name con_name, x.tablespace_name, x.fsm, y.apm
FROM x, y, v$containers v
WHERE x.con_id          = y.con_id
AND   x.tablespace_name = y.tablespace_name
AND   v.con_id          = y.con_id
UNION
SELECT vc2.con_id, vc2.name, tf.tablespace_name, null, SUM(tf.bytes)/1024/1024/1024
FROM v$containers vc2, cdb_temp_files tf
WHERE vc2.con_id = tf.con_id
GROUP BY vc2.con_id, vc2.name, tf.tablespace_name
ORDER BY 1, 2;
 ```
Details for one particular TBS
========================================================
```
SELECT d.status "Status", d.tablespace_name "Name", d.contents "Type", d.extent_management "Extent Management",
NVL(a.bytes / 1024 / 1024, 0) "Size (M)",
(NVL(a.bytes -NVL(f.bytes, 0), 0)/1024/1024) "Used (M)",
TO_CHAR(NVL((a.bytes -NVL(f.bytes, 0)) / a.bytes * 100, 0), '990.00') "Used %"
FROM sys.dba_tablespaces d, (select
tablespace_name, sum(bytes) bytes from dba_data_files group by tablespace_name) a, (select
tablespace_name, sum(bytes) bytes from dba_free_space group by tablespace_name) f WHERE
d.tablespace_name like '&TABLESPACE%' and d.tablespace_name = a.tablespace_name(+) AND d.tablespace_name = f.tablespace_name(+) AND NOT
(d.extent_management like 'LOCAL' AND d.contents like 'TEMPORARY')
UNION ALL
SELECT d.status "Status", d.tablespace_name "Name", d.contents "Type", d.extent_management "Extent Management",
NVL(a.bytes / 1024 / 1024, 0) "Size (M)", NVL(t.bytes, 0)/1024/1024 "Used (M)",
TO_CHAR(NVL(t.bytes / a.bytes * 100, 0), '990.00') "Used %"
FROM sys.dba_tablespaces d, (select tablespace_name, sum(bytes) bytes from dba_temp_files
group by tablespace_name) a, (select tablespace_name, sum(bytes_cached) bytes from
v$temp_extent_pool group by tablespace_name) t WHERE d.tablespace_name = a.tablespace_name(+) AND
d.tablespace_name = t.tablespace_name(+) AND d.extent_management like 'LOCAL' AND d.contents like
'TEMPORARY'
order by 2
/
```
```
select round((bytes/1024)/1024,0) "Used Space(MB)",
round(total,0) "Allocated size(MB)",
round(max,0) "Maximum allowable(MB)",
round(max-(BYTES/1024)/1024,0) "Effective free(MB)",
round(((max-(BYTES/1024)/1024)/max)*100,2) "FREE(%)"
from SYS.SM$TS_USED,
(select sum((BYTES/1024)/1024) total, sum((decode(MAXBYTES,0,bytes,maxbytes)/1024)/1024) max
from dba_data_files where tablespace_name='&1') where tablespace_name='&1'; 
```
TBS AND Datafile details 
===================
```
set pages 9999 lines 300 
set VERIFY OFF FEEDBACK OFF
COLUMN file_name         FORMAT A51        HEADING 'File Name'
COLUMN tablespace_name   FORMAT A15        HEADING 'Tablespace'
COLUMN meg               FORMAT 99,999.90  HEADING 'Megabytes'
COLUMN status            FORMAT A10        HEADING 'Status'
COLUMN autoextensible    FORMAT A3         HEADING 'Auto Extend'
COLUMN maxmeg            FORMAT 99,999     HEADING 'Max|Megabytes'
COLUMN Increment_by      FORMAT 9,999      HEADING 'Inc|By'

SELECT tablespace_name,file_name,bytes/1048576 meg,status,autoextensible,maxbytes/1048576 maxmeg,increment_by
FROM dba_data_files
UNION
SELECT tablespace_name,file_name,bytes/1048576 meg,status,autoextensible,maxbytes/1048576 maxmeg,increment_by
FROM dba_temp_files
ORDER BY tablespace_name
/
```
Very good script for tablespace usage =====Good ONE
==================================

```
	set pages 9999 lines 900
	col TABLESPACE_NAME for a30
	select a.tablespace_name,
	       ceil(sum(a.tots)/1024/1024/1024) "AllocatedSize(MB)", round((sum(a.tots)-sum(a.sumb))/1024/1024/1024,1) "Used(MB)",
	       round(((sum(a.tots)-sum(a.sumb))/sum(a.tots))*100,1) "Used%", round(sum(a.sumb)/1024/1024/1024,1) "Free(MB)",
	       round(sum(a.sumb)*100/sum(a.tots),1) "Free%", round(sum(a.largest)/1024/1024/1024,1) "LargestFreeSize(MB)",
	       sum(a.chunks) "ChunksFree"
	from
	  (select tablespace_name, 0 tots,sum(bytes) sumb, max(bytes) largest, count(*) chunks
	  from dba_free_space a
	  group by tablespace_name
	  union 
	select tablespace_name, sum(bytes) tots, 0, 0, 0
	  from dba_data_files
	  group by tablespace_name
	  union
	  select tablespace_name, sum(bytes), 0,0,0  from dba_temp_files
	  group by tablespace_name) a,  dba_tablespaces b
	where a.tablespace_name = b.tablespace_name
	group by a.tablespace_name
order by 4; 
```
Dynamic script for adding datafile or resize particular tablespace
==========================================================
DEFINE tbs=&APPLICATION_TS

```
	select cmd from 
	(select file_name,'alter database datafile ''' || file_name 
	|| ''' resize ' ||  bytes/1024/1024/1024 || 'G;' cmd 
	from dba_data_files where tablespace_name = upper('&tbs') 
	union
	select file_name, 'alter tablespace ' || tablespace_name || ' add datafile ''' 
	|| file_name || ''' size ' ||  to_char(bytes/1024/1024/1024 ) || 'G;'  cmd 
	from dba_data_files 
	where tablespace_name = upper('&tbs') )
	union 
	select cmd from 
	(select file_name,'alter database tempfile ''' || file_name 
	|| ''' resize ' ||  bytes/1024/1024/1024 || 'G;' cmd 
	from dba_temp_files where tablespace_name = upper('&tbs') 
	union
	select file_name, 'alter tablespace ' || tablespace_name || ' add tempfile ''' 
	|| file_name || ''' size ' ||  to_char(bytes/1024/1024/1024 ) || 'G;'  cmd 
	from dba_temp_files 
	where tablespace_name = upper('&tbs') )
	order by cmd
/
```
TABLESPACE LOCATION
=======================
```
set lines 150
column file_name format a80
column MB format 999,999,999
column MAXMB format 999,999,999
select file_name, bytes/1024/1024/1024 GB, autoextensible, maxbytes/1024/1024/1024 MAXGB
from dba_data_files
where tablespace_name = upper('&ts_name');
```

Finding which datafile having autoextend on
===============================================
```
select substr(file_name,1,50), AUTOEXTENSIBLE from dba_data_files
select tablespace_name,AUTOEXTENSIBLE from dba_data_files;
 ```
******* particular datafile deatils************************
 ```
 set pages 9999 lines 300
col tablespace_name for a30
col file_name for a80
select tablespace_name, file_name, bytes/1024/1024/1024 SIZE_GB, autoextensible,
maxbytes/1024/1024/1024 MAXSIZE_GB 
from dba_data_files
where tablespace_name='&tablespace_name' order by 1,2;
```


finding Distinct Tablespace for user
==================
```
SELECT DISTINCT sgm.TABLESPACE_NAME , dtf.FILE_NAME
FROM DBA_SEGMENTS sgm
JOIN DBA_DATA_FILES dtf ON (sgm.TABLESPACE_NAME = dtf.TABLESPACE_NAME)
WHERE sgm.OWNER = '&USER'
/
```
How To Get Tablespace Quota Details Of An User In Oracle:
========================================================

TABLESPACE QUOTA DETAILS OF ALL THE USERS:
==========================================

```
set pagesize 200
set lines 200
col ownr format a20         justify c heading 'Owner'
col name format a20         justify c heading 'Tablespace' trunc
col qota format a12         justify c heading 'Quota (KB)'
col used format 999,999,990 justify c heading 'Used (KB)'
set colsep '|'
SELECT username
       ownr,
       tablespace_name
       name,
       Decode(Greatest(max_bytes, -1), -1, 'UNLIMITED',
                                       To_char(max_bytes / 1024, '999,999,990'))
       qota,
       bytes / 1024
       used
FROM   dba_ts_quotas
WHERE  max_bytes != 0
        OR bytes != 0
ORDER  BY 1,2
/

```

TABLESPAE QUOTA DETAILS FOR A PARTICULAR USER:
==============================================
```
set pagesize 200
set lines 200
col ownr format a20         justify c heading 'Owner'
col name format a20         justify c heading 'Tablespace' trunc
col qota format a12         justify c heading 'Quota (KB)'
col used format 999,999,990 justify c heading 'Used (KB)'
set colsep '|'
SELECT username
       ownr,
       tablespace_name
       name,
       Decode(Greatest(max_bytes, -1), -1, 'UNLIMITED',
                                       To_char(max_bytes / 1024, '999,999,990'))
       qota,
       bytes / 1024
       used
FROM   dba_ts_quotas
WHERE  ( max_bytes != 0
          OR bytes != 0 )
       AND username = '&USERNAME'
ORDER  BY 1,2
/ 

```

Which schema taking more space under TS
----------------------------------------------
```
Select obj.owner "Owner", obj_cnt "Objects", decode(seg_size, NULL, 0, seg_size) "size MB"
from (select owner, count(*) obj_cnt from dba_objects group by owner) obj,
 (select owner, ceil(sum(bytes)/1024/1024) seg_size  from dba_segments group by owner) seg
  where obj.owner  = seg.owner(+)
  order    by 3 desc ,2 desc, 1;
```
```
SELECT t.tablespace_name, b.ts#, t.block_size, b.status, COUNT(*) num_bufs, ROUND(COUNT(*) * t.block_size / 1048576) pool_mb 
FROM v$bh b, dba_tablespaces t, v$tablespace vt
WHERE b.ts# = vt.ts# AND vt.name = t.tablespace_name GROUP BY t.tablespace_name, b.ts#, t.block_size, b.status ORDER BY COUNT(*) DESC;
```

To check Growth rate of  Tablespace
--------------------------------------------
```
SELECT TO_CHAR (sp.begin_interval_time,'DD-MM-YYYY') days,
 ts.tsname , max(round((tsu.tablespace_size* dt.block_size )/(1024*1024),2) ) cur_size_MB,
 max(round((tsu.tablespace_usedsize* dt.block_size )/(1024*1024),2)) usedsize_MB
 FROM DBA_HIST_TBSPC_SPACE_USAGE tsu, DBA_HIST_TABLESPACE_STAT ts, DBA_HIST_SNAPSHOT sp,
 DBA_TABLESPACES dt
 WHERE tsu.tablespace_id= ts.ts# AND tsu.snap_id = sp.snap_id
 AND ts.tsname = dt.tablespace_name AND ts.tsname NOT IN ('SYSAUX','SYSTEM')
 GROUP BY TO_CHAR (sp.begin_interval_time,'DD-MM-YYYY'), ts.tsname
 ORDER BY ts.tsname, days;
```
Count of object in tablespace
====================================
```
SELECT tablespace_name, owner, segment_type "Object Type",
       COUNT(owner) "Number of Objects",
       ROUND(SUM(bytes) / 1024 / 1024/1024, 2) "Total Size in GB"
FROM   sys.dba_segments
WHERE  tablespace_name IN ('&TBS')
GROUP BY tablespace_name, owner, segment_type
ORDER BY tablespace_name, owner, segment_type;
```

detail abt segment of the tablespace 
------------
```
 select tablespace_name, count(*) NUM_OBJECTS,
 sum(bytes), sum(blocks), sum(extents) from dba_segments 
 group by rollup (tablespace_name) order by 2;
```



Dynamic To remove datafiles from os location 
==========================
```
select 'rm ' || name from (select name  from v$datafile
 union all
 select name  from  v$tempfile
 union  all
 select  member  from  v$logfile
  union   all
  select  name   from    v$controlfile
 )
 /
```

------------------------------------------------
-- Script written for a case where data was loaded rapidly and without prior notice
-- DBAs got tired of adding more space every few hours
-- But there was a global company policy against auto-extend
-- So we decided that tablespace size should double on every resize
-- To minimize the number of resizes DBAs had to do.
------------------------------------------------------------

```
declare
  new_size number :=0;
  file_to_grow dba_data_files.file_name%TYPE := null;
  last_file dba_data_files.file_name%TYPE := null;
  l_instance_name v$instance.instance_name%TYPE;
begin
  FOR tbs_rec in
    (SELECT a.TABLESPACE_NAME tbs_name,
    a.BYTES bytes_used,
    b.BYTES bytes_free,
    b.largest,
    round(((a.BYTES-b.BYTES)/a.BYTES)*100,2) percent_used
    from
    (
    select TABLESPACE_NAME,
    sum(BYTES) BYTES
    from dba_data_files
    group by TABLESPACE_NAME
    )
    a join 
    (
    select TABLESPACE_NAME,
    sum(BYTES) BYTES ,
    max(BYTES) largest
    from dba_free_space
    group by TABLESPACE_NAME
    )
    b  on a.TABLESPACE_NAME=b.TABLESPACE_NAME
    where 1=1
    and round(((a.BYTES-b.BYTES)/a.BYTES)*100,2)>90
    and a.tablespace_name not like 'UNDO%'
    order by ((a.BYTES-b.BYTES)/a.BYTES) desc)
  LOOP
    dbms_output.put_line('Need more space in: ' || tbs_rec.tbs_name);
    for file_rec in
      (select file_name,bytes from dba_data_files where tablespace_name=tbs_rec.tbs_name order by file_id)
    LOOP
      dbms_output.put_line('found file ' || file_rec.file_name || ' with ' || file_rec.bytes || ' bytes size'); 
      if file_rec.bytes < 5000000000 /*5G*/ then
        new_size := file_rec.bytes*2; /*double file size*/
        file_to_grow := file_rec.file_name;
        exit; /* if found file we can grow, no need to check more files */
      elsif file_rec.bytes < 20000000000 /*20G*/ then
        new_size := file_rec.bytes+10000000000; /*add 10G*/
        file_to_grow := file_rec.file_name;
        exit; /* if found file we can grow, no need to check more files */
      else
        /* we need to keep track of existing files in case we need to add more */
        last_file := file_rec.file_name;
      end if;
    END LOOP;
    if file_to_grow is not null then
    /* grow file*/
    dbms_output.put_line('ALTER DATABASE DATAFILE ''' || file_to_grow || ''' RESIZE ' || new_size);
    else
    null;
    /* add new file */
    select instance_name into l_instance_name from v$instance;
    dbms_output.put_line(to_number(REGEXP_REPLACE(last_file,'.*_(\d+).dbf','\1'))+1);
    dbms_output.put_line('alter tablespace ' ||  tbs_rec.tbs_name || ' add datafile /u05/oradata/' ||  l_instance_name || '/' || tbs_rec.tbs_name ||'_' || to_char(to_number(REGEXP_REPLACE(last_file,'.*_(\d+).dbf','\1'))+1) || '.dbf size 4M autoextend off;');
    end if;
  END LOOP;
end;
/
```




