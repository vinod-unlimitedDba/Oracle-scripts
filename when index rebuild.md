
How to determine an index needs to be rebuilt
=============================================

Summary
 
An Oracle server index is a schema object that can speed up the retrieval of rows by using a pointer.
You can create indexes on one or more columns of a table to speed SQL statement execution on that table. If you do not have an index on the column, then a full table scan occurs.
You can reduce disk I/O by using a rapid path access method to locate data quickly. By default, Oracle creates B-tree indexes.
After a table experiences a large number of inserts, updates, and deletes, the index can become unbalanced and fragmented and can hinder query performance.

Knowing when to Rebuild Indexes
 
We must first get an idea of the current state of the index by using the ANALYZE INDEX VALIDATE STRUCTURE command. 

The VALIDATE STRUCTURE command can be safely executed without affecting the optimizer. The VALIDATE STRUCTURE command populates the SYS.INDEX_STATS table only. The SYS.INDEX_STATS table can be accessed with the public synonym INDEX_STATS. The INDEX_STATS table will only hold validation information for one index at a time. You will need to query this table before validating the structure of the next index.
 
Below is a sample output from INDEX_STATS Table.
 
 SQL> ANALYZE INDEX IDX_GAM_ACCT VALIDATE STRUCTURE;
 
Statement processed.
 
SQL> SELECT name, height,lf_rows,lf_blks,del_lf_rows FROM INDEX_STATS;
NAME                      HEIGHT    LF_ROWS    LF_BLKS    DEL_LF_ROW
---------------------- -----------   ----------      ----------   ----------------
DX_GAM_ACCT           2             1                     3               6
 
1 row selected.
 
There are two rules of thumb to help determine if the index needs to be rebuilt. 
1)     If the index has height greater than four, rebuild the index.
 
2)     The deleted leaf rows should be less than 20%.
 
If it is determined that the index needs to be rebuilt, this can easily be accomplished by the ALTER INDEX <INDEX_NAME> REBUILD | REBULID ONLINE command. It is not recommended, this command could be executed during normal operating hours. The alternative is to drop and re-create the index. Creating an index uses the base table as its data source that needs to put a lock on the table. The index is also unavailable during creation.
 
 In this example, the HEIGH column is clearly showing the value 2. This is not a good candidate for rebuilding. For most indexes, the height of the index will be quite low, i.e. one or two. I have seen an index on a 2 million-row table that had height two or three. An index with height greater than four may need to be rebuilt as this might indicate a skewed tree structure. This can lead to unnecessary database block reads of the index. Let’s take another example.
 SQL> ANALYZE INDEX IDX_GAM_FID VALIDATE STRUCTURE;
 
Statement processed.
 
SQL> SELECT name, height, lf_rows, del_lf_rows, (del_lf_rows/lf_rows) *100 as ratio FROM INDEX_STATS;
 
NAME                           HEIGHT     LF_ROWS    DEL_LF_ROW RATIO    
------------------------------ ---------- ---------- ---------- -------
IDX_GAM_FID                                  1          189         62        32.80
 
1 row selected.
 
In this example, the ratio of deleted leaf rows to total leaf rows
is clearly above 20%. This is a good candidate for rebuilding.
Let’s rebuild the index and examine the results
 
SQL> ANALYZE INDEX IDX_GAM_FID REBUILD;
 
Statement processed.
 
SQL> ANALYZE INDEX IDX_GAM_FID VALIDATE STRUCTURE;
 
Statement processed.
 
SQL> SELECT name, height, lf_rows, del_lf_rows, (del_lf_rows/lf_rows)*
100 as ratio FROM INDEX_STATS;
 
NAME                           HEIGHT     LF_ROWS    DEL_LF_ROW RATIO    
------------------------------ ---------- ---------- ---------- -------
IDX_GAM_FID                                  1          127         0        0
 
1 row selected. 
Examining the INDEX_STATS table shows that the 62 deleted leaf rows were dropped from the index. Notice that the total number of leaf rows went from 189 to 127, which is a difference of 62 leaf rows (189-127). This index should provide better performance for the application.




