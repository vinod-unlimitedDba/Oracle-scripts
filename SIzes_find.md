

DB SIZE
 ======
          select 
          ( select sum(bytes)/1024/1024/1024 data_size from dba_data_files ) +
          ( select nvl(sum(bytes),0)/1024/1024/1024 temp_size from dba_temp_files ) +
          ( select sum(bytes)/1024/1024/1024 redo_size from sys.v_$log ) +
          ( select sum(BLOCK_SIZE*FILE_SIZE_BLKS)/1024/1024/1024 controlfile_size from v$controlfile) "Size in GB"
          From dual
          /
#####

          col "Database Size" format a20
          col "Free space" format a20
          col "Used space" format a20
          select round(sum(used.bytes) / 1024 / 1024 / 1024 ) || ' GB' "Database Size"
          , round(sum(used.bytes) / 1024 / 1024 / 1024 ) - 
          round(free.p / 1024 / 1024 / 1024) || ' GB' "Used space"
          , round(free.p / 1024 / 1024 / 1024) || ' GB' "Free space"
          from (select bytes
          from v$datafile
          union all
          select bytes
          from v$tempfile
          union all
          select bytes
          from v$log) used
          , (select sum(bytes) as p
          from dba_free_space) free
          group by free.p
          /

 FOR ALL PUGGABLE Database
 ----------------------------------        
         
         Set lines 1000 pages 1000
          Col "Pluggable Database Name" for a20
          select con_id "Container ID", name "Pluggable Database Name", open_mode "PD_State", total_size/1024/1024/1024 "Size of DB in GB" from    v$pdbs;

To find the database Growth in month wise
-----------------------------------------


          select to_char(creation_time, 'MM-RRRR') "Month", sum(bytes)/1024/1024/1024 "Growth in GB"
          from sys.v_$datafile
          where to_char(creation_time,'RRRR')='2014'
          group by to_char(creation_time, 'MM-RRRR')
          order by  to_char(creation_time, 'MM-RRRR');
##### 2
          col growth_gb format 999G999D99
          col month     format a20
          select to_char(creation_time, 'YYYY MM') month, sum(bytes)/1024/1024/1024 growth_gb,
          ROUND(SUM(tablespace_size) * 8192 / 1024 / 1024 / 1024, 1) size_gb,
          ROUND(SUM(tablespace_usedsize) * 8192 / 1024 / 1024 / 1024, 1) usedsize_gb
          from  v$datafile
          where creation_time > SYSDATE - 720
          group by to_char(creation_time, 'YYYY MM')
          order by 1 desc
          /
          
  oracle script to check the database growth
  ---------------------------------------
    SET LINESIZE 200
    SET PAGESIZE 200
    COL "Database Size" FORMAT a13
    COL "Used Space" FORMAT a11
    COL "Used in %" FORMAT a11
    COL "Free in %" FORMAT a11
    COL "Database Name" FORMAT a13
    COL "Free Space" FORMAT a12
    COL "Growth DAY" FORMAT a11
    COL "Growth WEEK" FORMAT a12
    COL "Growth DAY in %" FORMAT a16
    COL "Growth WEEK in %" FORMAT a16

    SELECT
    (select min(creation_time) from v$datafile) "Create Time",
    (select name from v$database) "Database Name",
    ROUND((SUM(USED.BYTES) / 1024 / 1024 ),2) || ' MB' "Database Size",
    ROUND((SUM(USED.BYTES) / 1024 / 1024 ) - ROUND(FREE.P / 1024 / 1024 ),2) || ' MB' "Used Space",
    ROUND(((SUM(USED.BYTES) / 1024 / 1024 ) - (FREE.P / 1024 / 1024 )) / ROUND(SUM(USED.BYTES) / 1024 / 1024 ,2)*100,2) || '% MB' "Used in %",
    ROUND((FREE.P / 1024 / 1024 ),2) || ' MB' "Free Space",
    ROUND(((SUM(USED.BYTES) / 1024 / 1024 ) - ((SUM(USED.BYTES) / 1024 / 1024 ) - ROUND(FREE.P / 1024 / 1024 )))/ROUND(SUM(USED.BYTES) / 1024 / 1024,2 )*100,2) || '% MB' "Free in %",
    ROUND(((SUM(USED.BYTES) / 1024 / 1024 ) - (FREE.P / 1024 / 1024 ))/(select sysdate-min(creation_time) from v$datafile),2) || ' MB' "Growth DAY",
    ROUND(((SUM(USED.BYTES) / 1024 / 1024 ) - (FREE.P / 1024 / 1024 ))/(select sysdate-min(creation_time) from v$datafile)/ROUND((SUM(USED.BYTES) / 1024 / 1024 ),2)*100,3) || '% MB' "Growth DAY in %",
    ROUND(((SUM(USED.BYTES) / 1024 / 1024 ) - (FREE.P / 1024 / 1024 ))/(select sysdate-min(creation_time) from v$datafile)*7,2) || ' MB' "Growth WEEK",
    ROUND((((SUM(USED.BYTES) / 1024 / 1024 ) - (FREE.P / 1024 / 1024 ))/(select sysdate-min(creation_time) from v$datafile)/ROUND((SUM(USED.BYTES) / 1024 / 1024 ),2)*100)*7,3) || '% MB' "Growth WEEK in %"
    FROM    (SELECT BYTES FROM V$DATAFILE
    UNION ALL
    SELECT BYTES FROM V$TEMPFILE
    UNION ALL
    SELECT BYTES FROM V$LOG) USED,
    (SELECT SUM(BYTES) AS P FROM DBA_FREE_SPACE) FREE
    GROUP BY FREE.P;

Schema related query
===============
      set linesize 150
      set pagesize 5000
      col owner for a35
      col segment_name for a30
      col segment_type for a20
      col TABLESPACE_NAME for a30
      Compute sum of SIZE_IN_GB on report
      break on report
      select OWNER,sum(bytes)/1024/1024/1000 "SIZE_IN_GB" from dba_segments group by owner order by owner;

particular schema size in MB
====================
      col du_MB head MB FOR 99999999.9	
      col du_GB head GB FOR 99999999.9
      col du_owner HEAD OWNER FOR A30
      select
      owner du_owner
      , sum(bytes)/1048576 du_MB
      , sum(bytes)/1048576/1024 du_GB
      from
      dba_segments where
      lower(owner) like lower('&1')
      group by owner
      order by du_MB desc
      /

For single schema
===================
      select sum(bytes)/1024/1024/1024 from dba_segments where owner=upper('&owner');


Find SCHEMA size:
-----------------------
    SELECT sum(bytes)/1024/1024 MB FROM dba_segments where owner=’&owner_name’;

Schema size of segment
----------------------------
```  
   BREAK ON REPORT
   COMPUTE SUM LABEL TOTAL OF "Size of Each Segment in GB" ON REPORT
   select segment_type, sum(bytes/1024/1024/1024) "Size of Each Segment in GB" from dba_segments where owner='&SYS' group by segment_type order by 1;
```

Table size
=====

    select sum(bytes/1024/1024/1024) “size in GB” from dba_segments 
    where segment_name='&TABLE_NAME' and segment_type='TABLE';

##### 2

        SELECT SUM(bytes), SUM(bytes)/1024/1024/1024 GB
        FROM dba_extents     WHERE owner = '&owner'
        AND segment_name = '&table_name'
        /

##### 3
    
    select NUM_ROWS, BLOCKS, EMPTY_BLOCKS, AVG_SPACE, AVG_ROW_LEN, NUM_ROWS, NUM_FREELIST_BLOCKS 
    from all_tables where owner = 'DBUSR' and table_name = 'T'; 

How to find table size in database.
-------------------------------------
     SELECT DS.TABLESPACE_NAME, SEGMENT_NAME, ROUND(SUM(DS.BYTES) / (1024 * 1024 * 1024)) AS GB FROM DBA_SEGMENTS DS WHERE SEGMENT_NAME = '&tableName'
     GROUP BY DS.TABLESPACE_NAME,  SEGMENT_NAME;

To Get The Oracle Table size and index size
--------------------------------------
 ```  
   select segment_name,TABLESPACE_NAME ,segment_type, bytes/1024/1024/1024 size_gb from dba_segments where segment_name = ‘&segment_name’ or segment_name in (select       index_name from dba_indexes where table_name=’&tablename’ and table_owner=’&owner’);
```
Index size for table
==================
```
        SELECT owner, segment_name AS table_name, NULL as index_name, SUM(bytes)/1024/1024/1024 size_mb
        FROM dba_segments
        WHERE segment_name = '&TABLE'
        AND owner = '&OWNER'
        GROUP BY owner, segment_name
        UNION ALL
        SELECT i.owner, i.table_name, i.index_name, SUM(bytes)/1024/1024 size_mb
        FROM dba_indexes i, dba_segments s
        WHERE s.owner = i.owner
        AND s.segment_name = i.index_name
        AND i.table_name = '&TABLE'
        AND i.owner = '&OWNER'
        GROUP BY i.owner, i.table_name, i.index_name;
```

Show the size of an object
---------------------------
    col segment_name format a20
    select segment_name
    ,      bytes "SIZE_BYTES"
    ,      ceil(bytes / 1024 / 1024) "SIZE_MB"
    from   dba_segments
    where  segment_name like '&obj_name'
    /

Redolog size
===============
      SELECT
              a.GROUP#,
              a.THREAD#,
              a.SEQUENCE#,
              a.ARCHIVED,
              a.STATUS,
              b.MEMBER    AS REDOLOG_FILE_NAME,
              (a.BYTES/1024/1024) AS SIZE_MB
          FROM v$log a
          JOIN v$logfile b ON a.Group#=b.Group#
          ORDER BY a.GROUP# ASC;


##### FRA size

    select SPACE_USED/1024/1024/1024 "SPACE_USED(GB)" ,SPACE_LIMIT/1024/1024/1024 "SPACE_LIMIT(GB)" from  v$recovery_file_dest;

All in one in script all file size
=============== 

       SET LINESIZE 147
       SET PAGESIZE 9999
       SET VERIFY OFF

       COLUMN tablespace FORMAT a29 HEADING 'Tablespace Name / File Class'
       COLUMN filename FORMAT a64 HEADING 'Filename'
       COLUMN filesize FORMAT 99,999,999,999 HEADING 'File Size'
       COLUMN autoextensible FORMAT a4 HEADING 'Auto'
       COLUMN increment_by FORMAT 99,999,999,999 HEADING 'Next'
       COLUMN maxbytes FORMAT 99,999,999,999 HEADING 'Max'

       BREAK ON report
       COMPUTE SUM OF filesize ON report

       SELECT /*+ ordered */
       d.tablespace_name tablespace
       , d.file_name filename
       , d.bytes filesize
       , d.autoextensible autoextensible
       , d.increment_by * e.value increment_by
       , d.maxbytes maxbytes
       FROM
       sys.dba_data_files d , v$datafile v , (SELECT value FROM v$parameter WHERE name = 'db_block_size') e
       WHERE
       (d.file_name = v.name)
       UNION
       SELECT
       d.tablespace_name tablespace 
       , d.file_name filename
       , d.bytes filesize
       , d.autoextensible autoextensible
       , d.increment_by * e.value increment_by
       , d.maxbytes maxbytes
       FROM
       sys.dba_temp_files d , (SELECT value FROM v$parameter WHERE name = 'db_block_size') e
       UNION
       SELECT '[ ONLINE REDO LOG ]' , a.member , b.bytes , null , TO_NUMBER(null), TO_NUMBER(null)
       FROM v$logfile a , v$log b
       WHERE a.group# = b.group#
       UNION
       SELECT '[ CONTROL FILE ]' , a.name , TO_NUMBER(null) , null , TO_NUMBER(null), TO_NUMBER(null)
       FROM v$controlfile a ORDER BY 1,2
       /

ASM Diskgrp size
===================

    set wrap off
    set lines 120
    set pages 999
    col “Group”   form a25
    col “State”  form a15
    col “Type”   form a7
    col “Free GB”   form 9,999
     select group_number  "Group",  name "Group Name",state  "State" ,type  "Type",total_mb/1024 "Total GB" , free_mb/1024  "Free GB" from     v$asm_diskgroup
     /

##### LOB segment 
-------

```
          Declare
          TOTAL_BLOCKS number;
          TOTAL_BYTES number;
          UNUSED_BLOCKS number;
          UNUSED_BYTES number;
          LAST_USED_EXTENT_FILE_ID number;
          LAST_USED_EXTENT_BLOCK_ID number;
          LAST_USED_BLOCK number;

          begin
          dbms_space.unused_space('<owner>','<lob_name>','LOB',
          TOTAL_BLOCKS, TOTAL_BYTES, UNUSED_BLOCKS, UNUSED_BYTES,
          LAST_USED_EXTENT_FILE_ID, LAST_USED_EXTENT_BLOCK_ID,
          LAST_USED_BLOCK);

          dbms_output.put_line('SEGMENT_NAME = <lob segment="" name="">');
          dbms_output.put_line('-----------------------------------');
          dbms_output.put_line('TOTAL_BLOCKS = '||TOTAL_BLOCKS);
          dbms_output.put_line('TOTAL_BYTES = '||TOTAL_BYTES);
          dbms_output.put_line('UNUSED_BLOCKS = '||UNUSED_BLOCKS);
          dbms_output.put_line('UNUSED BYTES = '||UNUSED_BYTES);
          dbms_output.put_line('LAST_USED_EXTENT_FILE_ID = '||LAST_USED_EXTENT_FILE_ID);
          dbms_output.put_line('LAST_USED_EXTENT_BLOCK_ID = '||LAST_USED_EXTENT_BLOCK_ID);
          dbms_output.put_line('LAST_USED_BLOCK = '||LAST_USED_BLOCK);

          end;
          / 

```
 ##### Size and fragmentation
 ---------------------------------

     select a.owner, a.TABLE_NAME, b.SIZE_GB, ((a.BLOCKS*8192/1024/1024/1024)-(a.NUM_ROWS*AVG_ROW_LEN/1024/1024/1024)) as ACTUAL_GB, 
     (b.SIZE_GB-((a.BLOCKS*8192/1024/1024/1024)-(a.NUM_ROWS*AVG_ROW_LEN/1024/1024/1024))) Savings,
     a.TABLESPACE_NAME, b.SEGMENT_TYPE TABLE_TYPE, a.last_analyzed
     from 
     dba_tables a, 
     (select * from 
     (select owner, SEGMENT_NAME, SEGMENT_TYPE, TABLESPACE_NAME, sum(bytes/1024/1024/1024) Size_GB 
     from 
     dba_segments 
     where owner='&schema_name' and segment_type like '%TABLE%' 
     group by segment_name, owner, SEGMENT_TYPE, TABLESPACE_NAME 
     order by 5 desc) 
     where rownum<11) b 
     where a.TABLE_NAME=b.SEGMENT_NAME and a.owner=b.owner
     order by SIZE_GB desc;============== table 

  ######                     
                      
     select e.*, d.owner index_owner,d.index_name, d.index_type, sum(g.bytes/1024/1024/1024) index_size_gb, d.TABLESPACE_NAME index_tbs from         dba_segments g,         dba_indexes d
     right outer join 
      (select * from (
     select a.owner tabown, a.TABLE_NAME, b.SIZE_GB, ((a.BLOCKS*8192/1024/1024/1024)-(a.NUM_ROWS*AVG_ROW_LEN/1024/1024/1024)) as ACTUAL_GB, 
     (b.SIZE_GB-((a.BLOCKS*8192/1024/1024/1024)-(a.NUM_ROWS*AVG_ROW_LEN/1024/1024/1024))) Savings_GB,
     a.TABLESPACE_NAME tab_tbs, b.SEGMENT_TYPE TABLE_TYPE, a.last_analyzed
     from 
     dba_tables a, 
     (select * from 
      (select owner tabown, SEGMENT_NAME, SEGMENT_TYPE, TABLESPACE_NAME, sum(bytes/1024/1024/1024) Size_GB 
      from 
      dba_segments 
      where owner='&schema_name' and segment_type like '%TABLE%' 
      group by segment_name, owner, SEGMENT_TYPE, TABLESPACE_NAME 
      order by 5 desc) 
     where rownum<11) b 
     where a.TABLE_NAME=b.SEGMENT_NAME and a.owner=b.tabown
     order by SIZE_GB desc))
     e on e.tabown=d.table_owner and e.table_name=d.table_name
     where g.segment_name=d.index_name and d.owner=g.owner
     group by 
     e.tabown, e.table_name, size_gb, actual_gb, e.savings_GB, e.tab_tbs, e.table_type, e.last_analyzed, d.owner, d.index_name, d.index_type, d.tablespace_name
     order by size_gb desc ================= index
     ;


