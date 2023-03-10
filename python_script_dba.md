
Monitoring Tablespace and notify to dba when it reaches 90%
------------------------

>**Change the values according to the enviroment**

```
import cx_Oracle
import smtplib

# Define Oracle connection details
user = 'username'
password = 'password'
dsn = 'hostname:port/service_name'

# Connect to the Oracle database
connection = cx_Oracle.connect(user=user, password=password, dsn=dsn)

# Query data dictionary to get tablespace information
cursor = connection.cursor()
cursor.execute("""
    SELECT tablespace_name,
           round((total_space - free_space) / total_space * 100) as pct_used
    FROM (
        SELECT tablespace_name,
               sum(bytes) as total_space,
               sum(decode(autoextensible, 'YES', maxbytes, bytes)) as max_space,
               sum(decode(autoextensible, 'YES', maxbytes, bytes)) - sum(bytes) as free_space
        FROM dba_data_files
        GROUP BY tablespace_name
    )
""")

# Loop through each tablespace and check if it is 90% full
for tablespace, pct_used in cursor:
    if pct_used >= 90:
        # Send email notification
        sender_email = 'sender@example.com'
        receiver_email = 'receiver@example.com'
        password = 'email_password'
        message = f'Tablespace {tablespace} is {pct_used}% full.'
        smtp_server = 'smtp.example.com'
        port = 587
        context = ssl.create_default_context()
        with smtplib.SMTP(smtp_server, port) as server:
            server.starttls(context=context)
            server.login(sender_email, password)
            server.sendmail(sender_email, receiver_email, message)
            
# Close database connection and cursor
cursor.close()
connection.close()
            
            
```
