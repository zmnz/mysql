# mysql

mysqldump --single-transaction --skip-lock-tables --flush-privileges --all-databases > entire_database_server.sql

In this article, we will go through the steps-by-steps to setup a MySQL replication, without needing to stop the master or o your services, means NO downtime.

Most tutorials on master-slave replication will required to lock the tables to accomplish a consistent copy during the initial setup of the replication. This will be a major problem for critical and high traffic sites where downtime is not possible.

The basic steps to setup a MySQL master-slave replication:
Make sure bin-logging is enable in the Master DB. This is very important as Slave DB will rely on binlog to perform the replication. No binlog means no replication.
Create a mysqldump backup file, with binlog position included, so that later on we know which position to for the slave to start replicate
Transfer the mysqldump file to the Slave DB, and restore the mysqldump backup file. At this point the Slave DB will have data from the master, up to the point-in-time of the mysqldump is generated.
Let the Slave DB know who is the Master DB, and the position to start replicate.
Notes: Make sure that the server-id is unique on all MySQL server.

Step 1: Enable binlog in Master DB
1. First you need to edit the /etc/my.cnf on the Master DB, and add these lines in the [mysqld] section.

server-id=1
binlog-format=mixed
log-bin=mysql-bin
innodb_flush_log_at_trx_commit=1
sync_binlog=1

 
2. A restart on the Master DB is required to reflect the above configurations

systemctl restart mysqld
Step 2: Create  a replication user
3. Slave DB will use this user to connect to the Master DB. Login into your MySQL shell and execute the following command.

CREATE USER 'replicant'@'%';
GRANT REPLICATION SLAVE ON *.* TO 'replicant'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;

 
Step 3: Create a mysqldump file, gzip it, and transfer to the Slave DB

 
4. Next, we’ll create the mysqldump backup file, with the binlog position.

mysqldump --skip-lock-tables --single-transaction --flush-logs --master-data=2 -A > ~/mysqldump.sql
5. If you mysqldump file is huge, you can gzip it before transferring to the Slave DB.

gzip -9 ~/mysqldump.sql
6. Trasfer the mysqldump file to the Slave server

scp ~/mysqldump.sql.gz root@<>:~/
Step 4: Restore the mysqldump in Slave Server
7. Extract the gzipped mysqldump file

gunzip ~/mysqldump.sql.gz
8. Import the mysqldump file into the Slave server

mysql -uroot -p < ~/mysqldump.sql
Step 5: Configure /etc/my.cnf in Slave server
9. Edit the /etc/my.cnf file in Slave server and add the following lines in [mysqld] section.

server-id = 2
binlog-format = mixed
log_bin = mysql-bin
relay-log = mysql-relay-bin
log-slave-updates = 1
read-only = 1

 
10. Restart MySQL on the slave server to reflect the configuration changes.

systemctl restart mysqld
Step 6: Setup Replication in Slave server
11. Check the MASTER_LOG_FILE and MATER_LOG_POS. This information is available in the mysqldump file.

head dump.sql -n80 | grep "MASTER_LOG_POS"
11. Login into MySQL console and execute the below command to start replication.

CHANGE MASTER TO MASTER_HOST='<>',MASTER_USER='replicant',MASTER_PASSWORD='<>', MASTER_LOG_FILE='<>', MASTER_LOG_POS=<>;
START SLAVE;
12. To check the progress of your slave.

SHOW SLAVE STATUSG
