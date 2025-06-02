
-- Advanced Oracle Database Monitoring Script
-- This script provides a comprehensive view of database activity

SET PAGESIZE 0
SET LINESIZE 200
SET FEEDBACK OFF
SET VERIFY OFF
SET HEADING OFF
SET SERVEROUTPUT ON SIZE UNLIMITED
SET TRIMSPOOL ON

SPOOL advanced_monitoring_report.txt

PROMPT ============================================================
PROMPT 1. PROCESS AND SESSION COUNT SUMMARY
PROMPT ============================================================
SELECT '1. PROCESS AND SESSION COUNT SUMMARY' FROM dual;
PROMPT

SELECT 'Process Count: ' || a.ProcessCount || ', Session Count: ' || b.SessionCount 
FROM (SELECT COUNT(1) ProcessCount FROM v$process) a, 
     (SELECT COUNT(1) SessionCount FROM v$session) b;

PROMPT
SELECT 'Username: ' || DECODE(username, NULL, 'INTERNAL', USERNAME) || 
       ', Total: ' || COUNT(*) || 
       ', Active: ' || COUNT(DECODE(status, 'ACTIVE', STATUS)) || 
       ', Inactive: ' || COUNT(DECODE(status, 'INACTIVE', STATUS))
FROM v$session 
WHERE status IN ('ACTIVE', 'INACTIVE')
GROUP BY username
ORDER BY username;

PROMPT
PROMPT ============================================================
PROMPT 2. ACTIVE SESSIONS WITH SQL
PROMPT ============================================================
SELECT '2. ACTIVE SESSIONS WITH SQL' FROM dual;
PROMPT

DECLARE
    v_output VARCHAR2(4000);
BEGIN
    FOR x IN (
        SELECT s.sid, 
               s.serial#, 
               s.username, 
               s.machine, 
               s.program, 
               s.logon_time,
               s.sql_id,
               q.sql_text
        FROM v$session s
        LEFT JOIN v$sql q ON s.sql_id = q.sql_id
        WHERE s.status = 'ACTIVE'
        AND s.type != 'BACKGROUND'
        AND s.username IS NOT NULL
        ORDER BY s.logon_time
    ) LOOP
        v_output := 'SID: ' || x.sid || 
                   ', User: ' || NVL(x.username, 'N/A') || 
                   ', Machine: ' || SUBSTR(NVL(x.machine, 'N/A'), 1, 20) ||
                   ', Program: ' || SUBSTR(NVL(x.program, 'N/A'), 1, 30) ||
                   ', Logon: ' || TO_CHAR(x.logon_time, 'DD-MON-YYYY HH24:MI:SS');
        DBMS_OUTPUT.PUT_LINE(v_output);

        IF x.sql_id IS NOT NULL THEN
            DBMS_OUTPUT.PUT_LINE('SQL_ID: ' || x.sql_id);
            DBMS_OUTPUT.PUT_LINE('SQL: ' || SUBSTR(NVL(x.sql_text, 'N/A'), 1, 200));
        ELSE
            DBMS_OUTPUT.PUT_LINE('SQL: None');
        END IF;
        DBMS_OUTPUT.PUT_LINE('----------------------------------------');
    END LOOP;
END;
/

PROMPT
PROMPT ============================================================
PROMPT 3. BLOCKING CHAIN
PROMPT ============================================================
SELECT '3. BLOCKING CHAIN' FROM dual;
PROMPT

SELECT LPAD(' ', 2*(LEVEL-1)) || sid || ' (' || NVL(username, 'N/A') || ')' AS blocking_chain 
FROM (
    SELECT sid, username, blocking_session 
    FROM v$session
    WHERE blocking_session IS NOT NULL 
    OR sid IN (
        SELECT DISTINCT blocking_session 
        FROM v$session 
        WHERE blocking_session IS NOT NULL
    )
) 
START WITH blocking_session IS NULL 
CONNECT BY PRIOR sid = blocking_session;

PROMPT
PROMPT ============================================================
PROMPT 4. LONG RUNNING QUERIES (> 60 MINUTES)
PROMPT ============================================================
SELECT '4. LONG RUNNING QUERIES (> 60 MINUTES)' FROM dual;
PROMPT

SELECT 'User: ' || s.username || 
       ', SID: ' || s.sid || 
       ', Serial#: ' || s.serial#  || 
       ', Machine: ' || SUBSTR(s.machine, 1, 15) || 
       ', Program: ' || SUBSTR(s.program, 1, 30) || 
       ', Logon: ' || TO_CHAR(s.logon_time, 'DD-MON-YY HH24:MI:SS AM') || 
       ', Minutes Logged On: ' || ROUND((SYSDATE-s.LOGON_TIME)*24*60, 1) || 
       ', Minutes for Current SQL: ' || ROUND(s.LAST_CALL_ET/60, 1)
FROM v$session s
WHERE s.STATUS = 'ACTIVE' 
AND s.USERNAME IS NOT NULL 
AND ROUND((SYSDATE-s.LOGON_TIME)*24*60, 1) > 60
ORDER BY (SYSDATE-s.LOGON_TIME) DESC;

PROMPT
PROMPT ============================================================
PROMPT 5. CURRENTLY RUNNING SQL WITH WAIT EVENTS
PROMPT ============================================================
SELECT '5. CURRENTLY RUNNING SQL WITH WAIT EVENTS' FROM dual;
PROMPT

SELECT 'Count: ' || COUNT(*) || 
       ', SQL_ID: ' || sql_id || 
       ', State: ' || CASE state WHEN 'WAITING' THEN 'WAITING' ELSE 'ON CPU' END || 
       ', Event: ' || CASE state WHEN 'WAITING' THEN event ELSE 'On CPU / runqueue' END
FROM v$session 
WHERE status = 'ACTIVE' 
AND type != 'BACKGROUND' 
AND wait_class != 'Idle' 
AND sid != (SELECT sid FROM v$mystat WHERE rownum = 1)
GROUP BY sql_id, 
         CASE state WHEN 'WAITING' THEN 'WAITING' ELSE 'ON CPU' END, 
         CASE state WHEN 'WAITING' THEN event ELSE 'On CPU / runqueue' END
ORDER BY COUNT(*) DESC;

PROMPT
PROMPT ============================================================
PROMPT 6. PLAN CHANGES FOR CURRENTLY RUNNING SQL
PROMPT ============================================================
SELECT '6. PLAN CHANGES FOR CURRENTLY RUNNING SQL' FROM dual;
PROMPT

DECLARE
    v_output VARCHAR2(4000);
BEGIN
    FOR x IN (
        SELECT DISTINCT s.sql_id
        FROM v$session s
        WHERE s.status = 'ACTIVE'
        AND s.type != 'BACKGROUND'
        AND s.sql_id IS NOT NULL
        AND s.username IS NOT NULL
    ) LOOP
        -- Check if this SQL has multiple plans
        FOR y IN (
            SELECT sql_id, plan_hash_value, timestamp, executions, 
                   elapsed_time/DECODE(executions, 0, 1, executions)/1000000 avg_elapsed_sec
            FROM v$sql_plan_statistics_all
            WHERE sql_id = x.sql_id
            GROUP BY sql_id, plan_hash_value, timestamp, executions, elapsed_time
            ORDER BY timestamp DESC
        ) LOOP
            v_output := 'SQL_ID: ' || y.sql_id || 
                       ', Plan Hash: ' || y.plan_hash_value || 
                       ', Executions: ' || y.executions || 
                       ', Avg Elapsed (sec): ' || TO_CHAR(y.avg_elapsed_sec, '999990.999');
            DBMS_OUTPUT.PUT_LINE(v_output);
        END LOOP;

        -- Get SQL text for reference
        FOR z IN (
            SELECT SUBSTR(sql_text, 1, 100) sql_text
            FROM v$sql
            WHERE sql_id = x.sql_id
            AND ROWNUM = 1
        ) LOOP
            DBMS_OUTPUT.PUT_LINE('SQL: ' || z.sql_text);
        END LOOP;

        DBMS_OUTPUT.PUT_LINE('----------------------------------------');
    END LOOP;
END;
/

SPOOL OFF

PROMPT Report generated as advanced_monitoring_report.txt
