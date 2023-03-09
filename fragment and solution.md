##### What is Oracle Table Fragmentation?

If a table is only subject to inserts, there will not be any fragmentation. Fragmentation comes with when we update/delete data in table. 
The space which gets freed up during non-insert DML operations is not immediately re-used (or sometimes, may not get reuse ever at all).
This leaves behind holes in table which results in table fragmentation.


When rows are not stored contiguously, or if rows are split onto more than one block, performance decreases because these rows require additional block accesses.
Note that table fragmentation is different from file fragmentation. 


When a lot of DML operations are applied on a table, the table will become fragmented because DML does not release free space from the table below the HWM. 
HWM is an indicator of USED BLOCKS in the database. Blocks below the high water mark (used blocks) have at least once contained data. This data might have been deleted.


Since Oracle knows that blocks beyond the high water mark don’t have data, it only reads blocks up to the high water mark when doing a full table scan. 
DDL statement always resets the HWM. What are the reasons to reorganization of table? 
a) Slower response time (from that table)
b) High number of chained (actually migrated) rows. 
c) Table has grown many folds and the old space is not getting reused.

Note: Index based queries may not get that much benefited by reorg as compared to queries which does Full table scan.


How to find Table Fragmentation? In Oracle schema there are tables which has huge difference in actual size 
(size from User_segments) and expected size from user_tables (Num_rows*avg_row_length (in bytes)).
This all is due to fragmentation in the table or stats for table are not updated into dba_tables.

TO remove fragmentation 
----------------
1. Gather table stats: ———————
2.check table size
3. check fragmentation table


query to check fragmentation for the table
-------------

        set pages 50000 
        lines 32767 
        select owner,table_name,round((blocks*8),2)||'kb' "Fragmented size", round((num_rows*avg_row_len/1024),2)||'kb' "Actual size", round((blocks*8),2)-round((num_rows*avg_row_len/1024),2)||'kb', 
        ((round((blocks*8),2)-round((num_rows*avg_row_len/1024),2))/round((blocks*8),2))*100 -10 "reclaimable space % " from dba_tables where table_name ='&table_Name' AND OWNER LIKE '&schema_name' /



Query to find top 20 big tables with their fragmented status-
-------------

          set pages 200
          set lines 200
          col OWNER format a20
          col TABLE_NAME format a30
          select owner,table_name,round((blocks*8),2)/1024/1024 "size (Gb)" , 
          round((num_rows*avg_row_len/1024),2)/1024/1024 "actual_data (Gb)",
          (round((blocks*8),2) - round((num_rows*avg_row_len/1024),2))/1024/1024 "wasted_space (Gb)"
          from dba_tables
          where 
          (round((blocks*8),2) > round((num_rows*avg_row_len/1024),2))
          and 
          table_name in 
          (select segment_name from (select owner, segment_name, bytes/1024/1024 meg from dba_segments
          where 
          segment_type = 'TABLE' 
          and
          owner != 'SYS' and owner != 'SYSTEM' and owner != 'OLAPSYS' and owner != 'SYSMAN' and owner != 'ODM' and owner != 'RMAN' and owner != 'ORACLE_OCM' and owner != 'EXFSYS' and owner != 'OUTLN' and owner != 'DBSNMP' and owner != 'OPS' and owner != 'DIP' and owner != 'ORDSYS' and owner != 'WMSYS' and owner != 'XDB' and owner != 'CTXSYS' and owner != 'DMSYS' and owner != 'SCOTT' and owner != 'TSMSYS' and owner != 'MDSYS' and owner != 'WKSYS' and owner != 'ORDDATA' and owner != 'OWBSYS' and owner != 'ORDPLUGINS' and owner != 'SI_INFORMTN_SCHEMA' and owner != 'PUBLIC' and owner != 'OWBSYS_AUDIT' and owner != 'APPQOSSYS' and owner != 'APEX_030200' and owner != 'FLOWS_030000' and owner != 'WK_TEST' and owner != 'SWBAPPS' and owner != 'WEBDB' and owner != 'OAS_PUBLIC' and owner != 'FLOWS_FILES' and owner != 'QMS'
          order by bytes/1024/1024 desc) 
          where rownum <= 20)
          order by 5 desc;
          
          
          
