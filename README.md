# mysql_cluster
## Goal: 
Create 3 docker containers: mysql-m, mysql-s1, mysql-s2  
Setup master slave replication (Master: mysql-m, Slave: mysql-s1, mysql-s2)  
Write script that will frequently write data to database  
Ensure, that replication is working  
Try to turn off mysql-s1  
Try to remove a column in  database on slave node  

## Preparation
Give permissions:
```
mkdir master
mkdir master/binlog
mkdir master/config
mkdir master/data
chmod 777 master/binlog/
chmod 777 master/data/

mkdir slave1
slave1/binlog
slave1/config
slave1/data
chmod 777 slave1/binlog/
chmod 777 slave1/data/

mkdir slave2
slave2/binlog
slave2/config
slave2/data
chmod 777 slave2/binlog/
chmod 777 slave2/data/
```

## Master node
### Connect to mysql servee instance terminal
```
docker-compose exec mysql-m bash
mysql -uroot -p
passwordm
```

### Create user, db, table and share permission for user
Create user, db, table and share permission for user
```
CREATE USER 'my_user'@'%' IDENTIFIED WITH mysql_native_password BY 'my_password';
CREATE DATABASE mydb;
USE mydb;
CREATE TABLE Logs (
    Logid int NOT NULL AUTO_INCREMENT, 
    Log varchar(255) NOT NULL,
    User varchar(255) NOT NULL,
    Col1 varchar(255) NOT NULL,
    Col2 varchar(255) NOT NULL,
    Col3 varchar(255) NOT NULL,
    Col4 varchar(255) NOT NULL,
    Col5 varchar(255) NOT NULL,
    PRIMARY KEY (Logid)
    );
GRANT ALL PRIVILEGES ON * TO 'my_user'@'%';
FLUSH PRIVILEGES;
```

### Create users in MySQL with permission or replication
```
CREATE USER 'slave1'@'%' IDENTIFIED BY 'password1';
GRANT REPLICATION SLAVE ON *.* TO 'slave1'@'%';
FLUSH PRIVILEGES;
CREATE USER 'slave2'@'%' IDENTIFIED BY 'password2';
GRANT REPLICATION SLAVE ON *.* TO 'slave2'@'%';
FLUSH PRIVILEGES;
```

### Set Lock
```
USE mydb; 
FLUSH TABLES WITH READ LOCK;
```

See status: 
```
SHOW MASTER STATUS;
```
Output:
```
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000005 |      157 | mydb         |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

### Create dump
```
mysqldump -u root -p mydb > mydb.sql
passwordm
```

### Unlock
```
UNLOCK TABLES;
```

## Slave 1
### Connect to mysql servee instance terminal
```
docker-compose exec mysql-s1 bash
mysql -uroot -p
```

### Create db
```
CREATE DATABASE mydb;
```

### Load data from dump
```
mysql -u root -p mydb < share/mydb.sql
passwords1
```

### Configure replica
```
CHANGE MASTER TO MASTER_HOST='192.168.60.2', MASTER_USER='slave1', MASTER_PASSWORD='password1', MASTER_LOG_FILE = 'mysql-bin.000005',MASTER_LOG_POS = 157;
CHANGE MASTER TO GET_MASTER_PUBLIC_KEY=1;
```

### Launch slave
```
START SLAVE;
```
Output:
```
mysql-s1_1  | 2022-11-10T20:59:03.068319Z 244 [System] [MY-010562] [Repl] Slave I/O thread for channel '': connected to master 'slave1@192.168.60.2:3306',replication started in log 'mysql-bin.000005' at position 157
```

### Chack slave status
```
SHOW SLAVE STATUS;
```
Output
```
SHOW SLAVE STATUS;
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
| Slave_IO_State                   | Master_Host  | Master_User | Master_Port | Connect_Retry | Master_Log_File  | Read_Master_Log_Pos | Relay_Log_File         | Relay_Log_Pos | Relay_Master_Log_File | Slave_IO_Running | Slave_SQL_Running | Replicate_Do_DB | Replicate_Ignore_DB | Replicate_Do_Table | Replicate_Ignore_Table | Replicate_Wild_Do_Table | Replicate_Wild_Ignore_Table | Last_Errno | Last_Error | Skip_Counter | Exec_Master_Log_Pos | Relay_Log_Space | Until_Condition | Until_Log_File | Until_Log_Pos | Master_SSL_Allowed | Master_SSL_CA_File | Master_SSL_CA_Path | Master_SSL_Cert | Master_SSL_Cipher | Master_SSL_Key | Seconds_Behind_Master | Master_SSL_Verify_Server_Cert | Last_IO_Errno | Last_IO_Error | Last_SQL_Errno | Last_SQL_Error | Replicate_Ignore_Server_Ids | Master_Server_Id | Master_UUID                          | Master_Info_File        | SQL_Delay | SQL_Remaining_Delay | Slave_SQL_Running_State                                  | Master_Retry_Count | Master_Bind | Last_IO_Error_Timestamp | Last_SQL_Error_Timestamp | Master_SSL_Crl | Master_SSL_Crlpath | Retrieved_Gtid_Set | Executed_Gtid_Set | Auto_Position | Replicate_Rewrite_DB | Channel_Name | Master_TLS_Version | Master_public_key_path | Get_master_public_key | Network_Namespace |
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
| Waiting for source to send event | 192.168.60.2 | slave1      |        3306 |            60 | mysql-bin.000008 |                 504 | mysql-relay-bin.000005 |           720 | mysql-bin.000008      | Yes              | Yes               |                 |                     |                    |                        |                         |                             |          0 |            |            0 |                 504 |            1146 | None            |                |             0 | No                 |                    |                    |                 |                   |                |                     0 | No                            |             0 |               |              0 |                |                             |                1 | 4e89b210-60e7-11ed-8aca-0242c0a8c002 | mysql.slave_master_info |         0 |                NULL | Replica has read all relay log; waiting for more updates |              86400 |             |                         |                          |                |                    |                    |                   |             0 |                      |              |                    |                        |                     1 |                   |
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
1 row in set, 1 warning (0.00 sec)
```

## Slave 2
### Connect to mysql servee instance terminal
```
docker-compose exec mysql-s2 bash
mysql -uroot -p
```

### Create db
```
CREATE DATABASE mydb;
```

### Load data from dump
```
mysql -u root -p mydb < share/mydb.sql
passwords2
```

### Configure replica
```
CHANGE MASTER TO MASTER_HOST='192.168.60.2', MASTER_USER='slave2', MASTER_PASSWORD='password2', MASTER_LOG_FILE = 'mysql-bin.000005',MASTER_LOG_POS = 157;
CHANGE MASTER TO GET_MASTER_PUBLIC_KEY=1;
```

### Launch slave
```
START SLAVE;
```
Output:
```
mysql-s2_1  | 2022-11-10T20:59:29.661891Z 248 [System] [MY-010562] [Repl] Slave I/O thread for channel '': connected to master 'slave2@192.168.60.2:3306',replication started in log 'mysql-bin.000005' at position 157
```

### Chack slave status
```
SHOW SLAVE STATUS;
```
Output:
```
SHOW SLAVE STATUS;
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
| Slave_IO_State                   | Master_Host  | Master_User | Master_Port | Connect_Retry | Master_Log_File  | Read_Master_Log_Pos | Relay_Log_File         | Relay_Log_Pos | Relay_Master_Log_File | Slave_IO_Running | Slave_SQL_Running | Replicate_Do_DB | Replicate_Ignore_DB | Replicate_Do_Table | Replicate_Ignore_Table | Replicate_Wild_Do_Table | Replicate_Wild_Ignore_Table | Last_Errno | Last_Error | Skip_Counter | Exec_Master_Log_Pos | Relay_Log_Space | Until_Condition | Until_Log_File | Until_Log_Pos | Master_SSL_Allowed | Master_SSL_CA_File | Master_SSL_CA_Path | Master_SSL_Cert | Master_SSL_Cipher | Master_SSL_Key | Seconds_Behind_Master | Master_SSL_Verify_Server_Cert | Last_IO_Errno | Last_IO_Error | Last_SQL_Errno | Last_SQL_Error | Replicate_Ignore_Server_Ids | Master_Server_Id | Master_UUID                          | Master_Info_File        | SQL_Delay | SQL_Remaining_Delay | Slave_SQL_Running_State                                  | Master_Retry_Count | Master_Bind | Last_IO_Error_Timestamp | Last_SQL_Error_Timestamp | Master_SSL_Crl | Master_SSL_Crlpath | Retrieved_Gtid_Set | Executed_Gtid_Set | Auto_Position | Replicate_Rewrite_DB | Channel_Name | Master_TLS_Version | Master_public_key_path | Get_master_public_key | Network_Namespace |
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
| Waiting for source to send event | 192.168.60.2 | slave2      |        3306 |            60 | mysql-bin.000008 |                 504 | mysql-relay-bin.000005 |           720 | mysql-bin.000008      | Yes              | Yes               |                 |                     |                    |                        |                         |                             |          0 |            |            0 |                 504 |            1146 | None            |                |             0 | No                 |                    |                    |                 |                   |                |                     0 | No                            |             0 |               |              0 |                |                             |                1 | 4e89b210-60e7-11ed-8aca-0242c0a8c002 | mysql.slave_master_info |         0 |                NULL | Replica has read all relay log; waiting for more updates |              86400 |             |                         |                          |                |                    |                    |                   |             0 |                      |              |                    |                        |                     1 |                   |
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
1 row in set, 1 warning (0.01 sec)
```

## Check replication
### Insert test row
```
insert into Logs  (Log, User, Col1, Col2, Col3, Col4, Col5) Values('First', 'Me', 'First','First','First','First','First');
select * from Logs;
```
Output
```
+-------+-------+------+-------+-------+-------+-------+-------+
| Logid | Log   | User | Col1  | Col2  | Col3  | Col4  | Col5  |
+-------+-------+------+-------+-------+-------+-------+-------+
|     1 | First | Me   | First | First | First | First | First |
+-------+-------+------+-------+-------+-------+-------+-------+
1 row in set (0.00 sec)
```

### Check data in slave
```
select * from Logs;
```
Output
```
select * from Logs;
+-------+-------+------+-------+-------+-------+-------+-------+
| Logid | Log   | User | Col1  | Col2  | Col3  | Col4  | Col5  |
+-------+-------+------+-------+-------+-------+-------+-------+
|     1 | First | Me   | First | First | First | First | First |
+-------+-------+------+-------+-------+-------+-------+-------+
1 row in set (0.00 sec)
```

### Stop slave1
/usr/sbin/mysqld stop
Output
```
bash-4.4# /usr/sbin/mysqld stop
2022-11-10T21:07:00.488886Z 0 [Warning] [MY-010139] [Server] Changed limits: max_open_files: 4096 (requested 8161)
2022-11-10T21:07:00.488891Z 0 [Warning] [MY-010142] [Server] Changed limits: table_open_cache: 1967 (requested 4000)
2022-11-10T21:07:00.676011Z 0 [Warning] [MY-011068] [Server] The syntax '--skip-host-cache' is deprecated and will be removed in a future release. Please use SET GLOBAL host_cache_size=0 instead.
2022-11-10T21:07:00.677101Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.30) starting as process 2016
2022-11-10T21:07:00.684751Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2022-11-10T21:07:00.699262Z 1 [ERROR] [MY-012574] [InnoDB] Unable to lock ./ibdata1 error: 11
2022-11-10T21:07:01.699577Z 1 [ERROR] [MY-012574] [InnoDB] Unable to lock ./ibdata1 error: 11
```

### Insert few test row
```
insert into Logs  (Log, User, Col1, Col2, Col3, Col4, Col5) Values('Two', 'Me', 'Two','Two','Two','Two','Two');
insert into Logs  (Log, User, Col1, Col2, Col3, Col4, Col5) Values('Three', 'Me', 'Three','Three','Three','Three','Three');
insert into Logs  (Log, User, Col1, Col2, Col3, Col4, Col5) Values('Four', 'Me', 'Four','Four','Four','Four','Four');
insert into Logs  (Log, User, Col1, Col2, Col3, Col4, Col5) Values('Fifth', 'Me', 'Fifth','Fifth','Fifth','Fifth','Fifth');
select * from Logs;
```
Output
```
+-------+-------+------+-------+-------+-------+-------+-------+
| Logid | Log   | User | Col1  | Col2  | Col3  | Col4  | Col5  |
+-------+-------+------+-------+-------+-------+-------+-------+
|     1 | First | Me   | First | First | First | First | First |
|     2 | Two   | Me   | Two   | Two   | Two   | Two   | Two   |
|     3 | Three | Me   | Three | Three | Three | Three | Three |
|     4 | Four  | Me   | Four  | Four  | Four  | Four  | Four  |
|     5 | Fifth | Me   | Fifth | Fifth | Fifth | Fifth | Fifth |
+-------+-------+------+-------+-------+-------+-------+-------+
5 rows in set (0.00 sec)
```

### Check data in slave
```
select * from Logs;
```
Output
```
+-------+-------+------+-------+-------+-------+-------+-------+
| Logid | Log   | User | Col1  | Col2  | Col3  | Col4  | Col5  |
+-------+-------+------+-------+-------+-------+-------+-------+
|     1 | First | Me   | First | First | First | First | First |
|     2 | Two   | Me   | Two   | Two   | Two   | Two   | Two   |
|     3 | Three | Me   | Three | Three | Three | Three | Three |
|     4 | Four  | Me   | Four  | Four  | Four  | Four  | Four  |
|     5 | Fifth | Me   | Fifth | Fifth | Fifth | Fifth | Fifth |
+-------+-------+------+-------+-------+-------+-------+-------+
5 rows in set (0.00 sec)
```

### Start slave1 and check data
Start 
```
docker-compose exec mysql-s1 /usr/sbin/mysqld start
```
Check 
```
mysqldump -u root -p
passwords1
```
Output:
```
mysql> select * from mydb.Logs;
+-------+-------+------+-------+-------+-------+-------+-------+
| Logid | Log   | User | Col1  | Col2  | Col3  | Col4  | Col5  |
+-------+-------+------+-------+-------+-------+-------+-------+
|     1 | First | Me   | First | First | First | First | First |
|     2 | Two   | Me   | Two   | Two   | Two   | Two   | Two   |
|     3 | Three | Me   | Three | Three | Three | Three | Three |
|     4 | Four  | Me   | Four  | Four  | Four  | Four  | Four  |
|     5 | Fifth | Me   | Fifth | Fifth | Fifth | Fifth | Fifth |
+-------+-------+------+-------+-------+-------+-------+-------+
5 rows in set (0.00 sec)
```

### Delete middle column in slave1
```
ALTER TABLE mydb.Logs DROP COLUMN Col1;
SELECT * FROM mydb.Logs;
```
Output:
```
+-------+-------+------+-------+-------+-------+-------+
| Logid | Log   | User | Col2  | Col3  | Col4  | Col5  |
+-------+-------+------+-------+-------+-------+-------+
|     1 | First | Me   | First | First | First | First |
|     2 | Two   | Me   | Two   | Two   | Two   | Two   |
|     3 | Three | Me   | Three | Three | Three | Three |
|     4 | Four  | Me   | Four  | Four  | Four  | Four  |
|     5 | Fifth | Me   | Fifth | Fifth | Fifth | Fifth |
+-------+-------+------+-------+-------+-------+-------+
5 rows in set (0.00 sec)
```

### Delete last column in slave2
```
ALTER TABLE mydb.Logs DROP COLUMN Col5;
```
Output:
```
+-------+-------+------+-------+-------+-------+-------+
| Logid | Log   | User | Col1  | Col2  | Col3  | Col4  |
+-------+-------+------+-------+-------+-------+-------+
|     1 | First | Me   | First | First | First | First |
|     2 | Two   | Me   | Two   | Two   | Two   | Two   |
|     3 | Three | Me   | Three | Three | Three | Three |
|     4 | Four  | Me   | Four  | Four  | Four  | Four  |
|     5 | Fifth | Me   | Fifth | Fifth | Fifth | Fifth |
+-------+-------+------+-------+-------+-------+-------+
5 rows in set (0.00 sec)
```

### Insert few test row
```
insert into Logs  (Log, User, Col1, Col2, Col3, Col4, Col5) Values('Six', 'Me', 'Six','Six','Six','Six','Six');
insert into Logs  (Log, User, Col1, Col2, Col3, Col4, Col5) Values('Seven', 'Me', 'Seven','Seven','Seven','Seven','Seven');
select * from Logs;
```
Output:
```
+-------+-------+------+-------+-------+-------+-------+-------+
| Logid | Log   | User | Col1  | Col2  | Col3  | Col4  | Col5  |
+-------+-------+------+-------+-------+-------+-------+-------+
|     1 | First | Me   | First | First | First | First | First |
|     2 | Two   | Me   | Two   | Two   | Two   | Two   | Two   |
|     3 | Three | Me   | Three | Three | Three | Three | Three |
|     4 | Four  | Me   | Four  | Four  | Four  | Four  | Four  |
|     5 | Fifth | Me   | Fifth | Fifth | Fifth | Fifth | Fifth |
|     6 | Six   | Me   | Six   | Six   | Six   | Six   | Six   |
|     7 | Seven | Me   | Seven | Seven | Seven | Seven | Seven |
+-------+-------+------+-------+-------+-------+-------+-------+
7 rows in set (0.00 sec)
```

### Check data in slave1
```
SELECT * FROM mydb.Logs;
```
Output:
```
+-------+-------+------+-------+-------+-------+-------+
| Logid | Log   | User | Col2  | Col3  | Col4  | Col5  |
+-------+-------+------+-------+-------+-------+-------+
|     1 | First | Me   | First | First | First | First |
|     2 | Two   | Me   | Two   | Two   | Two   | Two   |
|     3 | Three | Me   | Three | Three | Three | Three |
|     4 | Four  | Me   | Four  | Four  | Four  | Four  |
|     5 | Fifth | Me   | Fifth | Fifth | Fifth | Fifth |
|     6 | Six   | Me   | Six   | Six   | Six   | Six   |
|     7 | Seven | Me   | Seven | Seven | Seven | Seven |
+-------+-------+------+-------+-------+-------+-------+
7 rows in set (0.00 sec)
```

### Check data in slave2
```
SELECT * FROM mydb.Logs;
```
Output:
```
+-------+-------+------+-------+-------+-------+-------+
| Logid | Log   | User | Col1  | Col2  | Col3  | Col4  |
+-------+-------+------+-------+-------+-------+-------+
|     1 | First | Me   | First | First | First | First |
|     2 | Two   | Me   | Two   | Two   | Two   | Two   |
|     3 | Three | Me   | Three | Three | Three | Three |
|     4 | Four  | Me   | Four  | Four  | Four  | Four  |
|     5 | Fifth | Me   | Fifth | Fifth | Fifth | Fifth |
|     6 | Six   | Me   | Six   | Six   | Six   | Six   |
|     7 | Seven | Me   | Seven | Seven | Seven | Seven |
+-------+-------+------+-------+-------+-------+-------+
7 rows in set (0.00 sec)
```

## After deletion middle column data move to nex column
main:
![image](https://user-images.githubusercontent.com/52753625/201881695-c3283aaf-4cea-41f4-8c54-2f89900bbc63.png)
slaves1:
![image](https://user-images.githubusercontent.com/52753625/201881744-daf117a2-0a9c-4b94-9550-e1cfc317794a.png)


### Check slave1 status one more time
```
SHOW SLAVE STATUS;
```
Output:
```
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
| Slave_IO_State                   | Master_Host  | Master_User | Master_Port | Connect_Retry | Master_Log_File  | Read_Master_Log_Pos | Relay_Log_File         | Relay_Log_Pos | Relay_Master_Log_File | Slave_IO_Running | Slave_SQL_Running | Replicate_Do_DB | Replicate_Ignore_DB | Replicate_Do_Table | Replicate_Ignore_Table | Replicate_Wild_Do_Table | Replicate_Wild_Ignore_Table | Last_Errno | Last_Error | Skip_Counter | Exec_Master_Log_Pos | Relay_Log_Space | Until_Condition | Until_Log_File | Until_Log_Pos | Master_SSL_Allowed | Master_SSL_CA_File | Master_SSL_CA_Path | Master_SSL_Cert | Master_SSL_Cipher | Master_SSL_Key | Seconds_Behind_Master | Master_SSL_Verify_Server_Cert | Last_IO_Errno | Last_IO_Error | Last_SQL_Errno | Last_SQL_Error | Replicate_Ignore_Server_Ids | Master_Server_Id | Master_UUID                          | Master_Info_File        | SQL_Delay | SQL_Remaining_Delay | Slave_SQL_Running_State                                  | Master_Retry_Count | Master_Bind | Last_IO_Error_Timestamp | Last_SQL_Error_Timestamp | Master_SSL_Crl | Master_SSL_Crlpath | Retrieved_Gtid_Set | Executed_Gtid_Set | Auto_Position | Replicate_Rewrite_DB | Channel_Name | Master_TLS_Version | Master_public_key_path | Get_master_public_key | Network_Namespace |
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
| Waiting for source to send event | 192.168.60.2 | slave1      |        3306 |            60 | mysql-bin.000008 |                2556 | mysql-relay-bin.000005 |          2772 | mysql-bin.000008      | Yes              | Yes               |                 |                     |                    |                        |                         |                             |          0 |            |            0 |                2556 |            3198 | None            |                |             0 | No                 |                    |                    |                 |                   |                |                     0 | No                            |             0 |               |              0 |                |                             |                1 | 4e89b210-60e7-11ed-8aca-0242c0a8c002 | mysql.slave_master_info |         0 |                NULL | Replica has read all relay log; waiting for more updates |              86400 |             |                         |                          |                |                    |                    |                   |             0 |                      |              |                    |                        |                     1 |                   |
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
1 row in set, 1 warning (0.00 sec)
```

### Check slave2 status one more time
```
SHOW SLAVE STATUS;
```
Output:
```
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
| Slave_IO_State                   | Master_Host  | Master_User | Master_Port | Connect_Retry | Master_Log_File  | Read_Master_Log_Pos | Relay_Log_File         | Relay_Log_Pos | Relay_Master_Log_File | Slave_IO_Running | Slave_SQL_Running | Replicate_Do_DB | Replicate_Ignore_DB | Replicate_Do_Table | Replicate_Ignore_Table | Replicate_Wild_Do_Table | Replicate_Wild_Ignore_Table | Last_Errno | Last_Error | Skip_Counter | Exec_Master_Log_Pos | Relay_Log_Space | Until_Condition | Until_Log_File | Until_Log_Pos | Master_SSL_Allowed | Master_SSL_CA_File | Master_SSL_CA_Path | Master_SSL_Cert | Master_SSL_Cipher | Master_SSL_Key | Seconds_Behind_Master | Master_SSL_Verify_Server_Cert | Last_IO_Errno | Last_IO_Error | Last_SQL_Errno | Last_SQL_Error | Replicate_Ignore_Server_Ids | Master_Server_Id | Master_UUID                          | Master_Info_File        | SQL_Delay | SQL_Remaining_Delay | Slave_SQL_Running_State                                  | Master_Retry_Count | Master_Bind | Last_IO_Error_Timestamp | Last_SQL_Error_Timestamp | Master_SSL_Crl | Master_SSL_Crlpath | Retrieved_Gtid_Set | Executed_Gtid_Set | Auto_Position | Replicate_Rewrite_DB | Channel_Name | Master_TLS_Version | Master_public_key_path | Get_master_public_key | Network_Namespace |
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
| Waiting for source to send event | 192.168.60.2 | slave2      |        3306 |            60 | mysql-bin.000008 |                2556 | mysql-relay-bin.000005 |          2772 | mysql-bin.000008      | Yes              | Yes               |                 |                     |                    |                        |                         |                             |          0 |            |            0 |                2556 |            3198 | None            |                |             0 | No                 |                    |                    |                 |                   |                |                     0 | No                            |             0 |               |              0 |                |                             |                1 | 4e89b210-60e7-11ed-8aca-0242c0a8c002 | mysql.slave_master_info |         0 |                NULL | Replica has read all relay log; waiting for more updates |              86400 |             |                         |                          |                |                    |                    |                   |             0 |                      |              |                    |                        |                     1 |                   |
+----------------------------------+--------------+-------------+-------------+---------------+------------------+---------------------+------------------------+---------------+-----------------------+------------------+-------------------+-----------------+---------------------+--------------------+------------------------+-------------------------+-----------------------------+------------+------------+--------------+---------------------+-----------------+-----------------+----------------+---------------+--------------------+--------------------+--------------------+-----------------+-------------------+----------------+-----------------------+-------------------------------+---------------+---------------+----------------+----------------+-----------------------------+------------------+--------------------------------------+-------------------------+-----------+---------------------+----------------------------------------------------------+--------------------+-------------+-------------------------+--------------------------+----------------+--------------------+--------------------+-------------------+---------------+----------------------+--------------+--------------------+------------------------+-----------------------+-------------------+
1 row in set, 1 warning (0.00 sec)
```
