This Document show how to create a Mysql Replica and examine the switchover process.
# Environment
- Os: Ubuntu 24 LTS
  - db-inst1    192.168.100.200
  - db-inst2    192.168.100.201
- DB: Mysql 8 Latest

# Os Readiness for Mysql Installation
## Update the Disk Parameters.
update the parmaters on disk which will hold MySql DataFiles.
```
$$ cat /etc/fstab | grep var
/dev/disk/..... /var xfs defaults,noatime,nodiratime 0 1
```
## Configuring swap
Most database needs to disable swap features. Best practice for mysql is `set Minimum amount of Swapping without Disabling`.
> You can choose a number between 0(disabling) to 9 (highest amount of using swap).
```
$$ echo "# Swappiness" >> /etc/sysctl.conf
$$ echo "vm.swappiness = 1" >> /etc/sysctl.conf
$$ sysctl -a
```
## Disable `Transparent Huge Page`
To make `Transparent Huge Page` permanently disable, make a service to run at startup.
```
$$ sudo su -
$$ cat <<EOF > /usr/lib/systemd/system/disable-thp.service
[Unit]
Description=Disable Transparent Huge Pages (THP)
 
[Service]
Type=simple
ExecStart=/bin/sh -c "echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled && echo 'never' > /sys/kernel/mm/transparent_hugepage/defrag"
 
[Install]
WantedBy=multi-user.target
EOF
 
$$ sudo systemctl daemon-reload
$$ sudo systemctl start disable-thp
$$ sudo systemctl enable disable-thp
```
## Update firewall 
to make all connections be secure set only the needed rules on you hosts.
```
$$ sudo ufw allow from 10.11.12.0/24 to any port 22
$$ sudo ufw allow from 192.168.100.0/24 to any port 3306
$$ sudo ufw enable
$$ sudo ufw status
Status: active
To                         Action      From
--                         ------      ----
22                         ALLOW       10.11.12.0/24
3306                       ALLOW       192.168.100.0/24
```
# Install and configure mysql on db-inst1
Let's install mysql on first server and configure the config file.
## Install mysql-server
```
$$ sudo apt-get install mysql-server
```
> Don't forget to run `mysql_secure_installation`
## update mysqld.conf
### set bind address and other paramters in mysqld.conf
```
$$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
server-id             = 1
log_bin               = /var/log/mysql/mysql-bin.log
binlog_do_db          = DB_prod             # this is the name of a db you want to replicate
#binlog_do_db          = DB_TEST            # In the case you want to replicate multiple databases

relay-log               = /var/log/mysql/mysql-relay-bin.log
bind-address            = 192.168.100.200
innodb_buffer_pool_size = 20G 
innodb_log_file_size    = 1G
```
> Note: set half of your system RAM on `innodb_buffer_pool_size`
> the innodb_buffer_pool_size and innodb_log_file_size will seedup mysql restore dump process. 
> Note: dont forget to comment default bidn-address parametter.
### Restart mysql server service.
```
$$ sudo systemctl restart mysql
```
### create replication user on db-inst1
you need to create replication user with sufficient grant permissions.
```
$$ sudo mysql -uroot
mysql > _
```
mysql> CREATE USER 'replica_user'@'db-inst2 IP address' IDENTIFIED WITH mysql_native_password BY 'password';
> db-inst2 IP address in my case is 192.168.100.201
```
mysql> CREATE USER 'replica_user'@'192.168.100.201' IDENTIFIED WITH mysql_native_password BY '$tr0nGpa$$';

mysql> GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'192.168.100.201';

mysql> FLUSH PRIVILEGES;
```
# Install and configure mysql on db-inst2
Let's install mysql on first server and configure the config file.
## Install mysql-server
```
$$ sudo apt-get install mysql-server
```
> Don't forget to run `mysql_secure_installation`
## update mysqld.conf and start mysql-server
### set bind address and other paramters in mysqld.conf
```
$$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
server-id             = 2
log_bin               = /var/log/mysql/mysql-bin.log
binlog_do_db          = DB_prod             # this is the name of a db you want to replicate
#binlog_do_db          = DB_TEST            # In the case you want to replicate multiple databases

relay-log               = /var/log/mysql/mysql-relay-bin.log
bind-address            = 192.168.100.201
innodb_buffer_pool_size = 20G 
innodb_log_file_size    = 1G
```
> Note: set half of your system RAM on `innodb_buffer_pool_size`
> the innodb_buffer_pool_size and innodb_log_file_size will seedup mysql restore dump process. 
> Note: don't forget to comment default bind-address parameter.
### Restart mysql server service.
```
$$ sudo systemctl restart mysql
```
### create replication user on db-inst2
you need to create replication user with sufficient grant permissions.
```
$$ sudo mysql -uroot
mysql > _
```
mysql> CREATE USER 'replica_user'@'db-inst1 IP address' IDENTIFIED WITH mysql_native_password BY 'password';
> db-inst1 IP address in my case is 192.168.100.200
```
mysql> CREATE USER 'replica_user'@'192.168.100.200' IDENTIFIED WITH mysql_native_password BY '$tr0nGpa$$';

mysql> GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'192.168.100.200';

mysql> FLUSH PRIVILEGES;
```
# Start Replication Process
In this step replication process will be start. First you need to choose which instance will play `Source Role`. In my case I choose `DB-INST1` as a source instance.
## db-inst1 source configuration
### Retrieving Binary Log Coordinates from source.
```
mysql> FLUSH TABLES WITH READ LOCK;
mysql> show master status;
+------------------+----------+----------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB   | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+----------------+------------------+-------------------+
| mysql-bin.000001 |      871 | DB_PROD        |                  |                   |
+------------------+----------+----------------+------------------+-------------------+
1 row in set (0.00 sec)
```
### make source database ready to replicate
- If your source database does not have any existing data:
```
mysql> UNLOCK TABLES;
mysql> CREATE DATABASE DB_PROD;
mysql> exit;
```
> Note: remember `DB_PROD` are present on mysqld.conf in `binlog_do_db` Parameter.
- If your source has already present and contain data you need to dump and restore the database on replica server (db-inst2).
  - dump data on source database `db-inst1`
```
$$ sudo mysqldump -u root DB_PROD > db.sql
$$ sudo scp db.sql my-user@db-inst2:/home/my-user/
```
  - restore the data on `db-inst2`
  ```
  $$ sudo mysql -uroot
  mysql> create database DB_PROD;
  mysql> exit;
  $$ sudo mysql -uroot DB_PROD < /home/my-user/db.sql
  ```
## db-inst2 Destination Configuration.
db-inst2 is replica target.
### set source bin log and its position to start replication.
- you need to set source IP address (db-inst1 IP= 192.168.100.200)
- you need to set user Name and password which is `replica_user`
- you need to set current bin log which will be present on command `show master status` from source instance. in my case it is `mysql-bin.000001`
- you need to set the position of the file which will be retrieved from cammand `show master status` form source instance. In my case it is `871`
> so the complete command is as follow:

```
mysql> CHANGE REPLICATION SOURCE TO
SOURCE_HOST='192.168.100.200',
SOURCE_USER='replica_user',
SOURCE_PASSWORD='$tr0nGpa$$',
SOURCE_LOG_FILE='mysql-bin.000001',
SOURCE_LOG_POS=871;
```
### start the replication and verify.
to start the replication run:
```
mysql > start replica;
```

in order to verify the replication status:
```
mysql > SHOW REPLICA STATUS\G;

mysql> show replica status\G
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: 192.168.100.200
                  Source_User: replica_user
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-bin.000001
          Read_Source_Log_Pos: 1086
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 541

. . .
```

## Test replication in action
- create a table and enter some records to source instance (db-inst1)
```
mysql> USE DB_PROD;
mysql> CREATE TABLE example_table (example_column varchar(30));

mysql> INSERT INTO example_table VALUES
('This is the first row'),
('This is the second row'),
('This is the third row');
```
- check data existance on db-inst2 (target replica server)
```
mysql> USE DB_PROD;
mysql> SHOW TABLES;

	Output
+---------------+
| Tables_in_db  |
+---------------+
| example_table |
+---------------+
1 row in set (0.00 sec)

mysql> SELECT * FROM example_table;
    Output
+------------------------+
| example_column         |
+------------------------+
| This is the first row  |
| This is the second row |
| This is the third row  |
+------------------------+
3 rows in set (0.00 sec)
```
# Switch over 
Note: normal condition means both instances are alive and you want to switch their role because of maintenance matters for example. 
so in my current state db-inst1 plays master node and db-inst2 is replica. I would change the roles and db-inst2 become master.
## check current status
before begin switch over process lets verify the cluster conditions and states.
### data sync check
- on `db-inst1`:
login to master and flush logs and then see the status
```
mysql> Flush logs;
mysql> show master status \G
*************************** 1. row ***************************
             File: mysql-bin.000002
         Position: 157
     Binlog_Do_DB: DB_PROD
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)
```
- on `db-inst2`:
Make sure slave running without log-slave-updates.
```
mysql> show replica status \G
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: 192.168.100.200
                  Source_User: replica_user
                  Source_Port: 3306
                Connect_Retry: 60
.
.
.
    Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
.
.
1 row in set (0.00 sec)
```
> 192.168.100.200 is my db-inst1 ip address which is in `Master` state.
> `Replica_SQL_Running_state:Replica has read all relay log; waiting for more updates` indicate the replica is sync with the source.
### process list check
- on `db-inst1`: make sure replica_user has sent all binlog to replica in state description.
```
mysql> SHOW PROCESSLIST\G
*************************** 1. row ***************************
     Id: 5
   .
   .
*************************** 2. row ***************************
     Id: 13
   User: root
   Host: localhost
   Info: SHOW PROCESSLIST
*************************** 3. row ***************************
     Id: 14
   User: replica_user
   Host: replica:48936
     db: NULL
Command: Binlog Dump
   Time: 525
  State: Source has sent all binlog to replica; waiting for more updates
   Info: NULL
3 rows in set, 1 warning (0.00 sec)
```
- on `db-inst2`: make sure system user has read all relay log
```
mysql> SHOW PROCESSLIST\G
*************************** 1. row ***************************
     Id: 5
   User: system user
   Host: connecting host
     db: NULL
Command: Connect
   Time: 655
  State: Waiting for source to send event
   Info: NULL
*************************** 2. row ***************************
     Id: 6
   User: system user
   Host:
     db: NULL
Command: Query
   Time: 360
  State: Replica has read all relay log; waiting for more updates
   Info: NULL
```
## switchover
After all conditions passes we can switch over replica server (db-inst2) become master.
### switchover db-inst2 to master
on `db-inst2`:
```
mysql> stop replica;
Query OK, 0 rows affected, 1 warning (0.06 sec)

mysql> reset master;
Query OK, 0 rows affected (0.06 sec)

mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 157
     Binlog_Do_DB: DB_PROD
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)
```
### switch over db-inst1 to replica
on `db-inst1`:
```
mysql> reset slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql > CHANGE REPLICATION SOURCE TO
SOURCE_HOST='192.168.100.201',
SOURCE_USER='replica_user',
SOURCE_PASSWORD='$tr0nGpa$$',
SOURCE_LOG_FILE='mysql-bin.000001',
SOURCE_LOG_POS=157;

mysql> start replica;
Query OK, 0 rows affected, 1 warning (0.13 sec)
```
### verify the switch over
to verify the switchover process, take placed on `db-inst1` which is our newly Replica server and check replica status:
```
mysql> show replica status \G
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: 192.168.100.201
                  Source_User: replica_user
                  Source_Port: 3306
                Connect_Retry: 60
.
.
.
    Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
.
.
1 row in set (0.00 sec)
```
