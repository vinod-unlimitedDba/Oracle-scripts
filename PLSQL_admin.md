
To kill several sessions of a user, following PLSQL block can be used 
-----------------------------------------------------------------

		BEGIN
		  FOR r IN (select sid,serial# from v$session where username = '&USER')
		  LOOP
		    EXECUTE IMMEDIATE 'alter system kill session ''' || r.sid 
		      || ',' || r.serial# || '''';
		  END LOOP;
		END;
		/

 To kill sessions of many users, following PLSQL block can be used.
-----------------------------------------------------------------

	begin
	   for i in (SELECT 'alter system kill session '''||SID||','||serial#||'''' as command               FROM   
		     gv$session where username in ('USERNAME1', 'USERNAME2')
		    )
	   loop
	      execute immediate i.command;
	   end loop;
	end;
	/


##### Killing session for particular user

	DECLARE
	 schema_name VARCHAR2(255):='username1'; -- Insert your username instead of 'username1'
	 row_count NUMBER;
	BEGIN
	 FOR r IN (SELECT sid,serial# FROM v$session WHERE username = schema_name)
	 LOOP
	   EXECUTE IMMEDIATE 'ALTER SYSTEM DISCONNECT SESSION ''' || r.sid || ',' || r.serial# || ''''||' IMMEDIATE';
	   EXECUTE IMMEDIATE 'ALTER SYSTEM KILL SESSION ''' || r.sid || ',' || r.serial# || '''';
	 END LOOP;

###### Table lock

              	DECLARE
		 CURSOR v_curqu IS
		 SELECT A.SID, A.SERIAL#, A.USERNAME
		 FROM V$SESSION A, V$LOCKED_OBJECT B, DBA_OBJECTS C
		 WHERE B.OBJECT_ID = C.OBJECT_ID
		 AND A.SID = B.SESSION_ID
		 AND OBJECT_NAME='LGCL_DB_ID'
		 AND A.TYPE != 'BACKGROUND';
		BEGIN
		FOR rec in v_curqu
		LOOP
		 DBMS_OUTPUT.PUT_LINE('User '|| rec.USERNAME ||'Killed');
		 EXECUTE IMMEDIATE 'ALTER SYSTEM KILL SESSION '''|| rec.Sid || ',' || rec.Serial# || '''';
		END LOOP;
		END;
		/

##### Creating synonym
		begin
		  FOR r IN (SELECT owner,object_name from dba_objects where owner='RDTEAM1' and object_type='TABLE')
		  loop 
		    EXECUTE IMMEDIATE 'CREATE SYNONYM TEST."'|| r.object_name ||'" FOR "' || r.owner || '"."' || r.object_name || '"';
		  end loop;
		END;
		/   
		
##### Drop synonym
	
	begin
		  FOR r IN (SELECT owner,object_name from dba_objects where owner='TEST' and object_type='SYNONYM')
		  loop 
		    EXECUTE IMMEDIATE 'drop synonym TEST."'|| r.object_name ||'"';
		  end loop;
		END;
		/
    
 ##### INvalid object_
 	
	DECLARE
	CURSOR c IS
	SELECT owner, object_name, object_type
	FROM dba_objects
	WHERE status='INVALID';
	v_sql VARCHAR2(500);
	BEGIN
	  FOR v IN c LOOP
	    IF v.object_type='PACKAGE BODY' THEN
		v_sql:='ALTER PACKAGE ' || v.owner || '.' || v.object_name || ' COMPILE BODY'; 
	    ELSIF v.object_type='SYNONYM' THEN
		v_sql:=dbms_metadata.get_ddl(object_type=>'SYNONYM',
					     name=>v.object_name,
					     schema=>v.owner);
	    ELSE
		v_sql:='ALTER ' || v.object_type || ' ' || v.owner || '.' || v.object_name || ' COMPILE'; 
	    END IF;
	    --DBMS_OUTPUT.PUT_LINE(v_sql);
	    BEGIN
		EXECUTE IMMEDIATE v_sql;
	    EXCEPTION
		WHEN OTHERS THEN
		    DBMS_OUTPUT.PUT_LINE(v_sql);
		    DBMS_OUTPUT.PUT_LINE(SQLERRM);
	    END;
	  END LOOP;
	END;
	/


###### To check object DDL's for specific typye 


 	begin
		  FOR r IN (SELECT owner,object_name,object_type from dba_objects where owner='ECDEV1' and object_type='TYPE')
		  loop 
		    EXECUTE IMMEDIATE 'SELECT DBMS_METADATA.GET_DDL('''|| r.object_type ||''','''|| r.object_name ||''','''|| r.owner ||''') FROM 		DUAL';		    
		  end loop; 
		 DBMS_OUTPUT.PUT_LINE('');  
		END;
		/  














