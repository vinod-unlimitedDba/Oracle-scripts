
DECLARE
    -- Variables to hold schema lists
    v_source_schemas VARCHAR2(1000) := 'NGCMREGSOLTVT,NGCMTAXSOLTVT,NGCMTINSOLTVTR';
    v_target_schemas VARCHAR2(1000) := 'CMREGSOLSC,CMTAXSOLSC,CMTINSOLSC';
    
    -- Variables to hold current schema pair
    v_source_schema VARCHAR2(100);
    v_target_schema VARCHAR2(100);
    
    -- Variables for parsing
    v_source_pos NUMBER := 1;
    v_target_pos NUMBER := 1;
    v_source_comma NUMBER;
    v_target_comma NUMBER;
    
    -- SQL statement template
    v_stmt VARCHAR2(32767);
    
    -- Table names to process
    TYPE table_array IS TABLE OF VARCHAR2(100);
    v_tables table_array := table_array(
        'PT_ENCRYPTION_ATTRIBUTES',
        'PT_ALGORITHMS',
        'PT_ALGORITHM_CONFIG_LOADERS',
        'PT_PROTECT_ROUTES',
        'PT_PROTECT_STEPS',
        'PT_PROTECT_STEP_CONFIG',
        'PT_PROTECTION_GROUPS',
        'PT_PROTECTION_POINT_GROUPS',
        'PT_PROTECTION_POINTS',
        'PT_CONFIG_GROUPS',
        'PT_ALGORITHM_CONFIGS',
        'PT_CONFIG_GROUP_PARAMS',
        'PT_PROTECTION_PROCESS',
        'PT_POINT_GROUP_PROTECTION',
        'PT_ATTRIB_GROUPS',
        'PT_PROTECTION_GROUP_ATTIBS',
        'PT_PROTECTION_POINT_ATTIBS',
        'PT_ATTRIBS'
    );
    
    -- Flag to track if any errors occurred
    v_has_error BOOLEAN;
    
    -- Variables to store error information
    v_error_table VARCHAR2(100);
    v_error_message VARCHAR2(4000);
BEGIN
    -- Loop through each schema pair
    LOOP
        -- Extract next source schema
        v_source_comma := INSTR(v_source_schemas, ',', v_source_pos);
        IF v_source_comma = 0 THEN
            v_source_schema := SUBSTR(v_source_schemas, v_source_pos);
            EXIT WHEN v_source_schema IS NULL;
        ELSE
            v_source_schema := SUBSTR(v_source_schemas, v_source_pos, v_source_comma - v_source_pos);
            v_source_pos := v_source_comma + 1;
        END IF;
        
        -- Extract next target schema
        v_target_comma := INSTR(v_target_schemas, ',', v_target_pos);
        IF v_target_comma = 0 THEN
            v_target_schema := SUBSTR(v_target_schemas, v_target_pos);
        ELSE
            v_target_schema := SUBSTR(v_target_schemas, v_target_pos, v_target_comma - v_target_pos);
            v_target_pos := v_target_comma + 1;
        END IF;
        
        DBMS_OUTPUT.PUT_LINE('Processing: ' || v_source_schema || ' -> ' || v_target_schema);
        
        -- Reset error flag for this schema pair
        v_has_error := FALSE;
        v_error_table := NULL;
        v_error_message := NULL;
        
        -- Process each table
        FOR i IN 1..v_tables.COUNT LOOP
            BEGIN
                v_stmt := 'INSERT INTO ' || v_target_schema || '.' || v_tables(i) || 
                          ' SELECT * FROM ' || v_source_schema || '.' || v_tables(i) || '@DBLINKEC';
                
                EXECUTE IMMEDIATE v_stmt;
                
                DBMS_OUTPUT.PUT_LINE('  Successfully migrated: ' || v_tables(i));
            EXCEPTION
                WHEN OTHERS THEN
                    -- Record the error
                    v_has_error := TRUE;
                    v_error_table := v_tables(i);
                    v_error_message := SQLERRM;
                    
                    -- Exit the loop immediately when an error occurs
                    EXIT;
            END;
        END LOOP;
        
        -- Check if any errors occurred
        IF v_has_error THEN
            -- Rollback all changes for this schema pair
            ROLLBACK;
            
            DBMS_OUTPUT.PUT_LINE('ERROR: Migration failed for schema pair ' || 
                                v_source_schema || ' -> ' || v_target_schema);
            DBMS_OUTPUT.PUT_LINE('Failed table: ' || v_error_table);
            DBMS_OUTPUT.PUT_LINE('Error message: ' || v_error_message);
            DBMS_OUTPUT.PUT_LINE('All changes for this schema pair have been rolled back.');
        ELSE
            -- Commit all changes for this schema pair
            COMMIT;
            DBMS_OUTPUT.PUT_LINE('Successfully migrated all tables for: ' || 
                                v_source_schema || ' -> ' || v_target_schema);
        END IF;
        
        -- Exit loop if we've processed the last source schema
        EXIT WHEN v_source_comma = 0;
    END LOOP;
    
    DBMS_OUTPUT.PUT_LINE('Migration process completed');
EXCEPTION
    WHEN OTHERS THEN
        -- Handle any unexpected errors
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Unexpected error occurred: ' || SQLERRM);
        DBMS_OUTPUT.PUT_LINE('All changes have been rolled back.');
END;
/


-- Prompt for connection parameters
ACCEPT p_link_name CHAR PROMPT 'Enter DB Link Name: '
ACCEPT p_hostname CHAR PROMPT 'Enter Remote Host: '
ACCEPT p_service CHAR PROMPT 'Enter Service Name: '
ACCEPT p_source_schemas CHAR PROMPT 'Enter Source Schemas (comma-separated): '
ACCEPT p_target_schemas CHAR PROMPT 'Enter Target Schemas (comma-separated): '
ACCEPT p_source_ldbid NUMBER PROMPT 'Enter Source LDBID (e.g., 2800): '
ACCEPT p_target_ldbid NUMBER PROMPT 'Enter Target LDBID (e.g., 5830): '

DECLARE
  v_password VARCHAR2(100) := LOWER('system' || '&p_service');
  v_dblink_created BOOLEAN := FALSE;
  v_sql VARCHAR2(4000);
  v_column_list CLOB;
  v_source_ldbid NUMBER := &p_source_ldbid;
  v_target_ldbid NUMBER := &p_target_ldbid;
  
  TYPE schema_array IS TABLE OF VARCHAR2(100);
  v_source_schemas schema_array := schema_array();
  v_target_schemas schema_array := schema_array();
  
  -- Function to split comma-separated string into array
  FUNCTION split_string(p_string IN VARCHAR2, p_delimiter IN VARCHAR2 DEFAULT ',') 
  RETURN schema_array IS
    v_array schema_array := schema_array();
    v_pos NUMBER;
    v_str VARCHAR2(4000) := p_string;
  BEGIN
    LOOP
      v_pos := INSTR(v_str, p_delimiter);
      IF v_pos > 0 THEN
        v_array.EXTEND;
        v_array(v_array.COUNT) := TRIM(SUBSTR(v_str, 1, v_pos - 1));
        v_str := SUBSTR(v_str, v_pos + 1);
      ELSE
        v_array.EXTEND;
        v_array(v_array.COUNT) := TRIM(v_str);
        EXIT;
      END IF;
    END LOOP;
    RETURN v_array;
  END split_string;
  
  -- Cursor to get tables with their ID column type
  CURSOR table_cur(p_schema VARCHAR2) IS
    SELECT table_name,
           CASE
             WHEN EXISTS (SELECT 1 FROM all_tab_columns 
                         WHERE owner = UPPER(p_schema)
                         AND table_name = ut.table_name
                         AND column_name = 'LDBID') THEN 'LDBID'
             WHEN EXISTS (SELECT 1 FROM all_tab_columns 
                         WHERE owner = UPPER(p_schema)
                         AND table_name = ut.table_name
                         AND column_name = 'LDI_LDBID') THEN 'LDI_LDBID'
             ELSE 'NONE'
           END AS id_type
    FROM all_tables ut
    WHERE owner = UPPER(p_schema);
BEGIN
  -- Split the comma-separated schema lists into arrays
  v_source_schemas := split_string('&p_source_schemas');
  v_target_schemas := split_string('&p_target_schemas');
  
  -- Validate that we have the same number of source and target schemas
  IF v_source_schemas.COUNT <> v_target_schemas.COUNT THEN
    RAISE_APPLICATION_ERROR(-20001, 'Number of source schemas must match number of target schemas');
  END IF;
  
  -- Create database link
  BEGIN
    EXECUTE IMMEDIATE '
      CREATE PUBLIC DATABASE LINK &p_link_name
      CONNECT TO SYSTEM IDENTIFIED BY "' || v_password || '"
      USING ''(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)
             (HOST=&p_hostname)
             (PORT=1521))
             (CONNECT_DATA=(SERVICE_NAME=&p_service)))''';
    v_dblink_created := TRUE;
    DBMS_OUTPUT.PUT_LINE('Database link &p_link_name created successfully.');
  EXCEPTION
    WHEN OTHERS THEN
      DBMS_OUTPUT.PUT_LINE('Error creating database link: ' || SQLERRM);
      RAISE;
  END;

  -- Process each schema pair
  FOR i IN 1..v_source_schemas.COUNT LOOP
    DBMS_OUTPUT.PUT_LINE('');
    DBMS_OUTPUT.PUT_LINE('-- Processing schema pair: ' || v_source_schemas(i) || ' -> ' || v_target_schemas(i));
    DBMS_OUTPUT.PUT_LINE('-- Source LDBID: ' || v_source_ldbid || ', Target LDBID: ' || v_target_ldbid);
    
    -- Generate insert statements for each table in the current source schema
    FOR table_rec IN table_cur(v_source_schemas(i)) LOOP
      BEGIN
        -- Get column list with LDBID transformation if needed
        SELECT 
          LISTAGG(
            CASE 
              WHEN (table_rec.id_type = 'LDBID' AND column_name = 'LDBID') THEN 
                v_target_ldbid || ' AS LDBID'
              WHEN (table_rec.id_type = 'LDI_LDBID' AND column_name = 'LDI_LDBID') THEN 
                v_target_ldbid || ' AS LDI_LDBID'
              ELSE 
                column_name
            END, 
            ', '
          ) WITHIN GROUP (ORDER BY column_id)
        INTO v_column_list
        FROM all_tab_columns
        WHERE owner = UPPER(v_source_schemas(i))
        AND table_name = table_rec.table_name;

        -- Build the SQL statement based on ID type
        CASE table_rec.id_type
          WHEN 'LDBID' THEN
            v_sql := 'INSERT INTO ' || v_target_schemas(i) || '.' || table_rec.table_name ||
                     ' SELECT ' || v_column_list ||
                     ' FROM ' || v_source_schemas(i) || '.' || table_rec.table_name ||
                     '@&p_link_name WHERE LDBID IN (' || v_source_ldbid || ',0)';
          WHEN 'LDI_LDBID' THEN
            v_sql := 'INSERT INTO ' || v_target_schemas(i) || '.' || table_rec.table_name ||
                     ' SELECT ' || v_column_list ||
                     ' FROM ' || v_source_schemas(i) || '.' || table_rec.table_name ||
                     '@&p_link_name WHERE LDI_LDBID IN (' || v_source_ldbid || ',0)';
          ELSE
            v_sql := 'INSERT INTO ' || v_target_schemas(i) || '.' || table_rec.table_name ||
                     ' SELECT * FROM ' || v_source_schemas(i) || '.' || table_rec.table_name ||
                     '@&p_link_name';
        END CASE;
        
        DBMS_OUTPUT.PUT_LINE(v_sql || '; COMMIT;');
      EXCEPTION
        WHEN OTHERS THEN
          DBMS_OUTPUT.PUT_LINE('Error processing table ' || table_rec.table_name || ': ' || SQLERRM);
      END;
    END LOOP;
  END LOOP;

EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
    IF v_dblink_created THEN
      BEGIN
        EXECUTE IMMEDIATE 'DROP PUBLIC DATABASE LINK &p_link_name';
        DBMS_OUTPUT.PUT_LINE('Database link &p_link_name dropped due to error.');
      EXCEPTION
        WHEN OTHERS THEN
          DBMS_OUTPUT.PUT_LINE('Error dropping database link: ' || SQLERRM);
      END;
    END IF;
    RAISE;
END;
/


