
##### Script for Datafile Location and Size
----------------------------------------------------
     REM Data File Listing for EXAMPLE Database
     REM Script File Name: My_Directory\DATAFILE.sql
     REM Spool File Name: My_Directory\DATAFILE.lst
     REM Author: NAME
     REM Date Created: DATE
     REM Purpose: Information on operating system level datafiles used
     REM by the tablespaces in EXAMPLE Database
     REM
     REM
     COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
     TTITLE LEFT xTODAY -
     RIGHT 'Page ' FORMAT 999 SQL.PNO -
     CENTER 'Data File Listing for EXAMPLE Database' SKIP 4
     BTITLE 'Script File: My_Directory\DATAFILE.sql|
     Spool File: My_Directory\DATAFILE.lst'
     COLUMN TABLESPACE_NAME HEADING 'Tablespace|Name' FORMAT A16
     COLUMN FILE_NAME HEADING 'File|Name' FORMAT A45
     COLUMN BLOCKS HEADING 'File Size in|Oracle Blocks' FORMAT 999,999,999
     BREAK ON TABLESPACE_NAME SKIP 1
     SET LINESIZE 78
     SET PAGESIZE 41
     SET NEWPAGE 0
     SPOOL My_Directory\DATAFILE.lst
     SELECT TABLESPACE_NAME, FILE_NAME, BLOCKS,
     TO_CHAR (SysDate, 'fmMonth ddth, YYYY') TODAY
     FROM SYS.DBA_DATA_FILES
     ORDER BY TABLESPACE_NAME, FILE_NAME;
     SPOOL OFF

Script for Listing Tablespaces with Extent Information
--------------------------------------------------------------------

     REM Tablespace Extent Listing for EXAMPLE Database
     REM Script File Name: My_Directory\TABLESPC.sql
     REM Spool File Name: My_Directory \TABLESPC.lst
     REM Author: NAME
     REM Date Created: DATE
     REM Purpose: Information on the tablespace extents
     in EXAMPLE database
     REM
     REM
     COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
     TTITLE LEFT xTODAY -
     RIGHT 'Page ' FORMAT 999 SQL.PNO -
     CENTER 'Tablespace Extents for EXAMPLE Database' SKIP 4
     BTITLE 'Script File: My_Directory \TABLESPC.sql|
     Spool File: My_Directory\TABLESPC.lst'
     COLUMN TABLESPACE_NAME HEADING 'Tablespace|Name' FORMAT A16
     COLUMN INITIAL_EXTENT HEADING 'Initial|Extent' FORMAT 999,999,999
     COLUMN NEXT_EXTENT HEADING 'Next|Extent' FORMAT 999,999,999
     COLUMN MAX_EXTENTS HEADING 'Max|ExtentS' FORMAT 999,999,999,999
     BREAK ON TABLESPACE_NAME SKIP 1
     SET LINESIZE 78
     SET PAGESIZE 41
     SET NEWPAGE 0
     SPOOL My_Directory\TABLESPC.lst
     SELECT TABLESPACE_NAME, INITIAL_EXTENT, NEXT_EXTENT, MAX_EXTENTS,
     TO_CHAR (SysDate, 'fmMonth ddth, YYYY') TODAY
     FROM DBA_TABLESPACES
     ORDER BY TABLESPACE_NAME;
     SPOOL OFF
     


##### Script for Listing Segments with Extent Information


    REM Segment Extent Listing for EXAMPLE Database
    REM Script File Name: My_Directory\SEGEXTN.sql
    REM Spool File Name: My_Directory\SEGEXTN.lst
    REM Author: NAME
    REM Date Created: DATE
    REM Purpose: Information on initial, next, and max extents of
    REM segments (tables and indices) in EXAMPLE Database
    REM
    REM
    COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
    TTITLE LEFT xTODAY -
    RIGHT 'Page ' FORMAT 999 SQL.PNO -
    CENTER 'Segment Extents for EXAMPLE Database ' SKIP 4
    BTITLE 'Script File: My_Directory\SEGEXTN.sql|
    Spool File: My_Directory\SEGEXTN.lst'
    COLUMN TABLESPACE_NAME HEADING 'Tablespace|Name' FORMAT A12
    COLUMN SEGMENT_NAME HEADING 'Segment|Name' FORMAT A20
    COLUMN INITIAL_EXTENT HEADING 'Initial|Extent' FORMAT 999,999,999
    COLUMN NEXT_EXTENT HEADING 'Next|Extent' FORMAT 999,999,999
    COLUMN MAX_EXTENTS HEADING 'Max|Extents' FORMAT 999,999,999,999
    BREAK ON TABLESPACE_NAME SKIP 2
    SET LINESIZE 78
    SET PAGESIZE 41
    SET NEWPAGE 0
    SPOOL My_Directory\SEGEXTN.lst
    SELECT TABLESPACE_NAME, SEGMENT_NAME, INITIAL_EXTENT, NEXT_EXTENT,
    MAX_EXTENTS, TO_CHAR (SysDate, 'fmMonth ddth, YYYY') TODAY
    FROM DBA_SEGMENTS WHERE
    SEGMENT_TYPE = 'TABLE' OR SEGMENT_TYPE = 'INDEX'
    AND TABLESPACE_NAME != 'SYSTEM'
    ORDER BY TABLESPACE_NAME, SEGMENT_NAME;
    SPOOL OFF

##### Script for Listing Segments with Extent Allocation


     REM Tablespace, Segment, and Extent Listing in EXAMPLE Database
     REM Script File Name: My_Directory\SEG_FRAG.sql
     REM Spool File Name: My_Directory \SEG_FRAG.lst
     REM Author: NAME
     REM Date Created: DATE
     REM Purpose: Information on tablespace segments and their
     REM associated extents created on all objects in EXAMPLE database.
     REM Note: Segment = Table, Index, Rollback, Temporary
     REM
     COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
     TTITLE LEFT xTODAY -
     RIGHT 'Page ' FORMAT 999 SQL.PNO -
     CENTER 'Tablespace Segment and Extent – EXAMPLE Database’
     SKIP 4
     BTITLE 'Script File: My_Directory \SEG_FRAG.sql|
     Spool File: My_Directory\SEG_FRAG.lst'
     COLUMN TABLESPACE_NAME HEADING 'Tablespace|Name' FORMAT A15
     COLUMN SEGMENT_NAME HEADING 'Segment|Name' FORMAT A25
     COLUMN SEGMENT_TYPE HEADING 'Segment|Type' FORMAT A8
     COLUMN EXTENT_ID HEADING 'Ext|ID' FORMAT 999
     COLUMN EXTENTS HEADING 'Ext' FORMAT 999
     COLUMN BLOCKS HEADING 'Blocks' FORMAT 999,999
     BREAK ON TABLESPACE_NAME SKIP 3 ON SEGMENT_TYPE SKIP 2 -
     ON SEGMENT_NAME SKIP 0
     SET LINESIZE 78
     SET PAGESIZE 41
     SET NEWPAGE 0
     SPOOL My_Directory\SEG_FRAG.lst
     SELECT a.TABLESPACE_NAME, a.SEGMENT_TYPE, a.SEGMENT_NAME, EXTENTS,
     a.BLOCKS, EXTENT_ID, b.BLOCKS,
     TO_CHAR (SysDate, 'fmMonth ddth, YYYY') TODAY
     FROM DBA_SEGMENTS a, DBA_EXTENTS b
     WHERE
     a.TABLESPACE_NAME = b.TABLESPACE_NAME AND
     a.SEGMENT_TYPE = b.SEGMENT_TYPE AND
     a.SEGMENT_NAME = b.SEGMENT_NAME AND
     a.TABLESPACE_NAME != 'SYSTEM'
     ORDER BY a.TABLESPACE_NAME, a.SEGMENT_TYPE, a.SEGMENT_NAME,
     EXTENT_ID;
     SPOOL OFF
     


 ###### Script for Listing Table Structures

     REM Table Structure Listing for EXAMPLE Database
     REM Script File Name: My_Directory\TBL_STRC.sql
     REM Spool File Name: My_Directory\TBL_STRC.lst
     REM Author: NAME
     REM Date Created: DATE
     REM Purpose: Complete listing of tables with their
     REM column names and data types in EXAMPLE
     REM Database
     REM
     COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
     TTITLE LEFT xTODAY -
     RIGHT 'Page ' FORMAT 999 SQL.PNO -
     CENTER 'Table Structures in EXAMPLE Database' SKIP 4
     BTITLE 'Script File: My_Directory\TBL_STRC.sql|
     Spool File: My_Directory\TBL_STRC.lst’
     COLUMN S_TABLE HEADING 'Table Name' FORMAT A20
     COLUMN COLUMN_NAME HEADING 'Column Name' FORMAT A25
     COLUMN COL_DATA HEADING 'Column|Size' FORMAT A20
     COLUMN NULLABLE HEADING 'Null?' FORMAT A6
     BREAK ON TABLE_NAME SKIP 1
     SET LINESIZE 78
     SET PAGESIZE 41
     SET NEWPAGE 0
     SPOOL My_Directory\TBL_STRC.lst
     SELECT a.TABLE_NAME S_TABLE, COLUMN_NAME,
     DATA_TYPE || '(' || DATA_LENGTH || ')' COL_DATA, NULLABLE,
     TO_CHAR (SysDate, 'fmMonth ddth, YYYY') TODAY
     FROM DBA_TABLES a, DBA_TAB_COLUMNS b
     WHERE a.TABLE_NAME = b.TABLE_NAME
     ORDER BY 1;
     SPOOL OFF

##### Index Structure Listing


     REM Index Listing for EXAMPLE Database
     REM Script File Name: My_Directory\INDXRPT.sql
     REM Spool File Name: My_Directory \INDXRPT.lst
     REM Author: NAME
     REM Date Created: DATE
     REM Purpose: Information on index names and structures
     REM used in tables in EXAMPLE Database
     REM
     REM
     COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
     TTITLE LEFT xTODAY -
     RIGHT 'Page ' FORMAT 999 SQL.PNO -
     CENTER 'Index Listing for EXAMPLE Database ' SKIP 4
     BTITLE 'Script File: My_Directory\INDXRPT.sql|Spool File:
     My_Directory\INDXRPT.lst'
     COLUMN TABLE_NAME HEADING 'Table|Name' FORMAT A20
     COLUMN INDEX_NAME HEADING 'Index|Name' FORMAT A20
     COLUMN COLUMN_NAME HEADING 'Column|Name' FORMAT A20
     COLUMN COLUMN_POSITION HEADING 'Column|Position' FORMAT 999
     COLUMN COLUMN_LENGTH HEADING 'Column|Length' FORMAT 9,999
     BREAK ON TABLE_NAME SKIP 1 ON INDEX_NAME SKIP 0
     SET LINESIZE 78
     SET PAGESIZE 41
     SET NEWPAGE 0
     SPOOL My_Directory\INDXRPT.lst
     SELECT TABLE_NAME, INDEX_NAME, COLUMN_NAME, COLUMN_POSITION,
     COLUMN_LENGTH,
     TO_CHAR (SysDate, 'fmMonth ddth, YYYY') TODAY
     FROM DBA_IND_COLUMNS
     ORDER BY TABLE_NAME, INDEX_NAME;
     SPOOL OFF



##### Script for Listing Constraints

      REM Constraint Listing for EXAMPLE Database
      REM Script File Name: My_Directory\CONSTRAINT.sql
      REM Spool File Name: My_Directory\CONSTRAINT.lst
      REM Author: NAME
      REM Date Created: DATE
      REM Purpose: Information on constraints created on all tables in
      REM EXAMPLE database
      REM
      REM
      COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
      TTITLE LEFT xTODAY -
      RIGHT 'Page ' FORMAT 999 SQL.PNO -
      CENTER 'Constraint Listing for EXAMPLE Database' SKIP 4
      BTITLE 'Script File: My_Directory\CONSTRAINT.sql|
      Spool File: My_Directory\CONSTRAINT.lst'
      COLUMN TABLE_NAME HEADING 'Table Name' FORMAT A12
      COLUMN CONSTRAINT_NAME HEADING 'Constraint|Name' FORMAT A12
      COLUMN CONSTRAINT_TYPE HEADING 'Type' FORMAT A4
      COLUMN COLUMN_NAME HEADING 'Column Name' FORMAT A12
      COLUMN COLUMN_POSITION HEADING 'Column|Position' FORMAT 999
      COLUMN DELETE_RULE HEADING 'Delete|Rule' FORMAT A12
      BREAK ON TABLE_NAME SKIP 2
      SET LINESIZE 78
      SET PAGESIZE 41
      SET NEWPAGE 0
      SPOOL My_Directory \CONSTRAINT.lst
      SELECT a.TABLE_NAME, a.CONSTRAINT_NAME, CONSTRAINT_TYPE,
      COLUMN_NAME, POSITION, DELETE_RULE,
      TO_CHAR (SysDate, 'fmMonth ddth, YYYY') TODAY
      FROM DBA_CONSTRAINTS a, DBA_CONS_COLUMNS b WHERE
      a.CONSTRAINT_NAME = b.CONSTRAINT_NAME AND
      a.TABLE_NAME = b.TABLE_NAME
      ORDER BY TABLE_NAME, CONSTRAINT_NAME;
      SPOOL OFF

##### Script for Listing Users using SYSTEM Tablespace

    REM SYSTEM tablespace used by users
    REM Script File Name: My_Directory\SYSTEM_User_Objects.sql
    REM Spool File Name: My_Directory\SYSTEM_User_Objects.lst
    REM Author: NAME
    REM Date Created: DATE
    REM Purpose: Find if SYSTEM tablespace is used by any
    REM user
    REM as the default or temporary tablespace
    REM
    COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
    TTITLE LEFT xTODAY -
    RIGHT 'Page ' FORMAT 999 SQL.PNO -
    CENTER 'User Objects in SYSTEM Tablespace' SKIP 4
    BTITLE 'Script File: My_Directory\SYSTEM_User_Objects.sql|
    Spool File: My_Directory\SYSTEM_User_Objects.lst'
    COLUMN USERNAME FORMAT A15
    COLUMN DEFAULT_TABLESPACE HEADING 'Default|Tablespace' FORMAT A15
    COLUMN TEMPORARY_TABLESPACE HEADING 'Temporary|Tablespace' FORMAT 
    A15
    SET LINESIZE 78
    SET PAGESIZE 41
    SET NEWPAGE 0
    SPOOL My_Directory\SYSTEM_User_Objects.lst
    select username, TO_CHAR (created, 'DD-MON-YYYY') "Created ",
    default_tablespace, temporary_tablespace,
    TO_CHAR (SysDate, 'fmMonth ddth, YYYY') TODAY
    from dba_users where
    (dba_users.DEFAULT_TABLESPACE = 'SYSTEM' or
    dba_users.TEMPORARY_TABLESPACE = 'SYSTEM') and
    dba_users.USERNAME not in ('SYS', 'SYSTEM');
    spool off
 
##### Used Space Extent Map
 
             REM Used Space Extent Map for EXAMPLE Database
             REM Script File Name: My_Directory\extent_contiguity.sql
             REM Spool File Name: My_Directory \extent_contiguity.lst
             REM Author: NAME
             REM Date Created: DATE
             REM Purpose: For each segment, list all its extents
             REM by Extent_ID REM and Block_ID
             REM Note: Segment = Table, Index
             REM
             COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
             TTITLE LEFT xTODAY -
             RIGHT 'Page ' FORMAT 999 SQL.PNO -
             CENTER ' Used Space Extent Map - EXAMPLE Database' SKIP 4
             BTITLE 'Script File: My_Directory\ extent_contiguity.sql|
             Spool File: My_Directory\ extent_contiguity.lst'
             COLUMN SEGMENT_NAME HEADING 'Segment|Name' FORMAT A25
             COLUMN SEGMENT_TYPE HEADING 'Segment|Type' FORMAT A8
             COLUMN EXTENT_ID HEADING 'Ext|ID' FORMAT 999
             COLUMN BLOCK_ID HEADING 'Block|ID' FORMAT 99999
             COLUMN EXTENTS HEADING 'Ext' FORMAT 999
             COLUMN BLOCKS HEADING 'Blocks' FORMAT 999,999
             BREAK ON SEGMENT_NAME SKIP 0 ON SEGMENT_TYPE SKIP 0 -
             ON EXTENTS SKIP 0
             SET LINESIZE 78
             SET PAGESIZE 41
             SET NEWPAGE 0
             SPOOL My_Directory\extent_contiguity.lst
             SELECT a.SEGMENT_NAME, a.SEGMENT_TYPE, EXTENTS, EXTENT_ID,
             Block_ID, b.BLOCKS,
             TO_CHAR (SysDate, 'fmMonth ddth, YYYY') TODAY
             FROM DBA_SEGMENTS a, DBA_EXTENTS b
             WHERE
             a.SEGMENT_NAME = b.SEGMENT_NAME AND
             a.SEGMENT_TYPE = b.SEGMENT_TYPE AND
             a.TABLESPACE_NAME != 'SYSTEM' AND
             a.SEGMENT_NAME NOT LIKE '%$%'
             ORDER BY a.SEGMENT_NAME, EXTENT_ID;
             SPOOL OFF

##### Status of Free Space in Tablespaces

  SET LINESIZE 100
  SET PAGESIZE 41
  SET NEWPAGE 0
  SELECT TABLESPACE_NAME TABLESPACE, ROUND (SUM (BYTES) / 1048576)
  "Free Space in MB",
  ROUND (MAX (BYTES) / 1048576) "Largest Contiguous Chunk in MB"
  FROM DBA_FREE_SPACE
  GROUP BY TABLESPACE_NAME;


##### script for  Segments Unable to Extend

    REM Free Space Unavailable for EXAMPLE Database
    REM Script File Name: My_Directory\Free_Space_Unavailable.sql
    REM Spool File Name: My_Directory\Free_Space_Unavailable.lst
    REM Author: NAME
    REM Date Created: DATE
    REM Purpose: Information on tablespace(s) where free
    REM space is NOT AVAILABLE for the next
    REM extent of any segment in the tablespace(s)
    REM
    REM
    COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
    TTITLE LEFT xTODAY -
    RIGHT 'Page ' FORMAT 999 SQL.PNO -
    CENTER 'Free Space Unavailable in Tablespaces' SKIP 4
    BTITLE 'Script File: My_Directory\Free_Space_Unavailable.sql|
    Spool File: My_Directory\Free_Space_Unavailable.lst'
    COLUMN TABLESPACE_NAME HEADING 'Tablespace|Name' FORMAT A16
    COLUMN SEGMENT_NAME HEADING 'Segment|Name' FORMAT A20
    COLUMN TABLE_NAME HEADING 'Table|Name' FORMAT A20
    BREAK ON TABLESPACE_NAME SKIP 1 ON TABLE_NAME SKIP 0
    SET LINESIZE 78
    SET PAGESIZE 41
    SET NEWPAGE 0
    SPOOL My_Directory\Free_Space_Unavailable.lst
    select s.tablespace_name, s.segment_name, s.next_extent "Next
    Extent", f.free_bytes "Free Bytes", TO_CHAR (SysDate,
    'fmMonth ddth, YYYY')
    TODAY
    from dba_segments s,
    (select tablespace_name,
    sum(bytes) free_bytes
    from dba_free_space
    group by tablespace_name) f
    where f.tablespace_name = s.tablespace_name
    and s.next_extent > f.free_bytes;
    SPOOL OFF


##### FSFI should be 100. But as fragmentation occurs, FSFI gradually decreases.
##### In general, a tablespace with FSFI < 30 should be further examined for possible defragmentation.
##### Run the script in Figure 5.24 to calculate FSFI for each tablespace

    REM Measure the level of free space fragmentation in tablespaces
    REM Script File Name: My_directory\FSFI.sql
    REM Script File Name: My_directory\FSFI.lst
    REM Purpose: This script measures the fragmentation of
    REM free space in all of the tablespaces in a
    REM database and scores them according to an
    REM index for comparison.
    REM
    COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
    TTITLE LEFT xTODAY -
    RIGHT 'Page ' FORMAT 999 SQL.PNO -
    CENTER 'FSFI Index for EXAMPLE Database' SKIP 2
    BTITLE 'Script File: My_directory \FSFI.sql|
    Spool File: My_directory \FSFI.lst'
    COLUMN TABLESPACE_NAME HEADING 'Tablespace|Name' FORMAT A20
    COLUMN FSFI FORMAT 999.99
    BREAK ON TABLESPACE_NAME SKIP 0
    SET LINESIZE 78
    SET PAGESIZE 41
    SET NEWPAGE 0
    SPOOL My_directory \FSFI.lst
    SELECT TABLESPACE_NAME,
    SQRT(MAX(Blocks)/SUM(Blocks))*
    (100/SQRT(SQRT(COUNT(Blocks)))) FSFI,
    TO_CHAR (SysDate, 'fmMonth ddth, YYYY') TODAY
    FROM DBA_FREE_SPACE
    GROUP BY TABLESPACE_NAME
    ORDER BY 1;
    SPOOL OFF

##### Script for Listing Free Space Extents in Tablespaces

List the File IDs, Block IDs, and the total blocks for each tablespace with FSFI < 30
and determine how many of their free space extents are contiguous.

     REM Free Space Listing for Tablespaces in EXAMPLE Database
     REM Script File Name: My_Directory\FREESPACE.sql
     REM Spool File Name My_Directory\FREESPACE.lst
     REM Author: NAME
     REM Date Created: DATE
     REM Purpose: Information on free space extents in all
     REM tablespaces in EXAMPLE database
     REM
     REM
     COLUMN TODAY NEW_VALUE xTODAY NOPRINT FORMAT A1 TRUNC
     TTITLE LEFT xTODAY -
     RIGHT 'Page ' FORMAT 999 SQL.PNO -
     CENTER 'Free Space List - EXAMPLE Database ' SKIP 4
     BTITLE 'Script File: My_Directory\FREESPACE.sql|Spool File:
     My_Directory\FREESPACE.lst'
     COLUMN TABLESPACE_NAME HEADING 'Tablespace|Name' FORMAT A15
     COLUMN FILE_ID HEADING 'File ID' FORMAT 999
     COLUMN BLOCK_ID HEADING 'Block ID' FORMAT 999999999
     COLUMN BLOCKS HEADING 'Total|Blocks' FORMAT 999,999,999
     BREAK ON TABLESPACE_NAME SKIP 2 ON FILE_ID SKIP 1
     SET LINESIZE 78
     SET PAGESIZE 41
     SET NEWPAGE 0
     SPOOL My_Directory\FREESPACE.lst
     SELECT TABLESPACE_NAME, FILE_ID, BLOCK_ID, BLOCKS,
     TO_CHAR (SysDate, 'fmMonth ddth, YYYY') TODAY
     FROM DBA_FREE_SPACE
     ORDER BY TABLESPACE_NAME, FILE_ID, BLOCK_ID;
     SPOOL OFF
